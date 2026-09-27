---
title: Obsidian API quirks
description: Undocumented Obsidian API behaviors. Covers file write timing, race conditions, Bases config quirks, and workarounds.
author: 🤖 Generated with Claude Code
updated: 2026-09-27
---

# Obsidian API quirks

## Debounced disk writes (~2 seconds)

Obsidian debounces all file writes via `TextFileView.requestSave` with a **2-second** delay (documented in the TypeScript API). This applies globally — Markdown files, `.base` files, and any file managed by Obsidian's editor system.

### Implication for `vault.process()`

`vault.process(file, fn)` reads the file **from disk**, not from Obsidian's in-memory state. If Obsidian has pending in-memory changes that haven't been flushed (within the ~2s debounce window), `vault.process()` reads stale content. Writing back the transformed result overwrites the file **without** the pending changes, causing data loss.

### Known race condition

When a user creates a new Bases view and quickly switches its type (e.g., table → dynamic-views-grid), the new view exists in Obsidian's memory but not on disk. If `vault.process()` runs within the debounce window, it reads the file without the new view, rewrites it, and the view is lost. Obsidian then shows "View X not found."

## Notice container stale cache

> Observed in Obsidian **1.12.1**, installer 1.11.4.

Obsidian caches the `.notice-container` DOM element in a `Map<Window, HTMLDivElement>` keyed by `activeWindow`. When the last notice in a container fades out, `Notice.hide()` detaches both the notice element and the empty container from the DOM — but does **not** remove the Map entry.

On the next `new Notice()`, the constructor finds the cached (but detached) container via `Map.get(activeWindow)`, appends the new notice to it, and never re-attaches the container to `document.body`. The notice is invisible.

### When it triggers

This only affects notices created **after all previous notices have fully faded** (animation complete + detach). If notices overlap (a new one while another is still visible), the container stays in the DOM and the bug doesn't manifest.

### Workaround

After `new Notice()`, check if the container is connected and re-attach if stale:

```typescript
const notice = new Notice("...");
const nc = (notice as { containerEl?: HTMLElement }).containerEl?.parentElement;
if (nc && !nc.isConnected) {
  activeWindow.document.body.appendChild(nc);
}
```

## `BasesEntry.getValue()` undocumented `.data` property

> Observed in Obsidian **1.12.1**, installer 1.11.4.

`BasesEntry.getValue(propertyId)` returns `Value | null`. The official `Value` class hierarchy only exposes `toString()`, `isTruthy()`, `equals()`, `looseEquals()`, and `renderTo()`. No `.data` accessor is typed.

At runtime, `Value` subclasses store their raw data in an undocumented `.data` property:

- `PrimitiveValue<T>` (StringValue, NumberValue, BooleanValue, etc.): `.data` is the primitive value (`string`, `number`, `boolean`).
- `ListValue` (multitext properties like `tags`, `aliases`): `.data` is an array of `Value` objects or primitives.

### Accessing raw data

The plugin accesses `.data` via type assertion since it's not in the type definitions:

```typescript
const value = entry.getValue("note.author") as { data?: unknown } | null;
const data = value?.data;
if (Array.isArray(data)) {
  // multitext: data is an array
} else if (typeof data === "string") {
  // text: data is a string
}
```

### Fragility

This relies on Obsidian's internal `Value` implementation. If the internal property is renamed or restructured, access breaks silently (returns `undefined`). There is no public API alternative for extracting raw values beyond `toString()`.

## How an open `.base` saves and reloads

> Observed in Obsidian **1.14.2**.

A `.base` file's save and reload behavior depends on how it is open, and the differences make an outside write to any open form unsafe.

- **Config writes never touch disk directly.** `BasesViewConfig.set(key, value)` mutates the config's in-memory data object — `null` deletes a key, any other value (including `undefined`) is assigned — then calls the owning query's `save()`, routed to whichever save function the hosting instance registered. `get(key)` returns the raw stored value with no schema-default fallback (see "Bases `config.get()` returns the raw stored value" below). `getAll()` returns that same live data object, or a fresh empty one before anything is set.
- **A standalone tab** debounces its save 2000ms from the *first* pending change — a later change in the same window does not restart the timer — and the flush serializes whatever the view's query holds at flush time, not the value at the moment of the edit. The tab ignores its own resulting disk write (a saving flag, then an equality check against what it last wrote), but an outside write that differs is loaded as a new query object, **replacing** the tab's own unflushed changes rather than merging with them.
- **An embedded view** debounces its save 1000ms and writes the query object its last change belonged to — even when a write from elsewhere has since reloaded the embed into a newer one, so its flush can land a stale copy over that write. Unlike a standalone tab, it reloads on every change to the file, its own writes included.
- **A view written as a code block in a note** saves through a short-debounced rewrite of the block's own text in the host note — it never touches a `.base` file at all.
- **A same-content reload keeps the same config object.** `QueryController.setQuery()` re-uses the existing query object when the incoming query serializes identically to what it already holds, so a round-trip that changes nothing does not invalidate an object a caller is still holding. Any other reload assigns a new query object, and the controller re-points the active view's config to it on every update regardless.
- **The Configure view UI keeps its own reference.** Once opened, it holds the config object that was live at the time. If the file reloads underneath it into a new query object, the UI's next edit still writes to the old object, and saving from that old query hands the controller the old query back — undoing the reload.
- **Each `set()` has a real cost.** It serializes the whole query about three times — the save function's last-written copy and the controller's identical-content check — which measured about 1ms per call against a small file (20 calls took 22ms on an M4 Pro with a 715-byte base), scaling with file size.
- **Patching a live view's save methods observes nothing.** Replacing `save` or `onModify` on an already-constructed standalone view logged no calls, because the debounced save and the vault's modify listener hold references taken when the view was built. Read the file from disk with a direct filesystem read to see when a save lands.
- **Multiple visible instances of the same file race independently.** Each instance debounces and flushes on its own schedule, so when two instances of the same file are visible and initializing at once, an edit made in one can be silently discarded if the other instance's shorter flush window reloads the file first. This needs two concurrently-initializing visible instances of the same file plus an edit landing inside the other's flush window — not a path an ordinary single-instance edit reaches.

### Implication

A background write to an open `.base` file through `vault.process()` or `vault.modify()` races every mechanism above — it can be silently replaced by a pending in-memory save, or itself replace another instance's unflushed edit. Writing through `config.set()` on the config object the running instance already holds rides Obsidian's own save path instead, avoiding the race entirely outside the multi-instance case above.

## A Bases layout switch keeps the view's data

**Observed**: 2026-09-27, Obsidian 1.14.2 (read from `app.js`)

A view's settings live in one data object that every layout reads, so switching a view's layout hands the same keys to the next layout.

- **Own properties vs data**: `type`, `name`, `filters`, `groupBy`, `order`, `sort`, `limit` and `summaries` are properties of the view config itself. Every other key lives in its data object — the one `getAll()` returns and `set()` writes.
- **A layout switch changes only `type`**: the Configure view page reassigns `type` on the same config object, so the data object and every key in it carry into the new layout, including keys the new layout never reads.
- **Unknown keys round-trip**: Obsidian saves keys it does not recognize, so one view's data can hold the keys of every layout it has been, core and plugin-provided alike.
- **Core layouts read shared names**: Cards reads `cardSize` (slider 50–800, default 200), `image`, `imageFit` and `imageAspectRatio`. Kanban reads `image`, `imageFit`, `imageAspectRatio`, `columnWidth` (200–500, default 280), `hideEmptyGroups` and `groupOrder`. Table reads `rowHeight` and `columnSize`, and List reads `markers`, `indentProperties` and `separator`. Cards and Kanban treat `imageFit: 'contain'` as contain and any other value as cover; their Cover option stores `''`.

### Implication

- **Never delete a key you do not own**: a view type that cleans its data against an allowlist erases every other layout's settings the moment a view is switched to it. Delete only keys your own plugin wrote.
- **A key named like a core one is shared**: when a custom layout also reads `cardSize` or `imageFit`, the core layout reads the same value back after a switch. Dropping it as default-equal against your own default changes what the core layout shows, since its default differs.

## Bases `config.get()` returns the raw stored value

> Read from `app.js` in Obsidian **1.14.2**. An earlier note, observed on 1.12.1 (installer 1.11.4), said `get()` fell back to the schema default; 1.14.2 does not.

- **No fallback**: `BasesViewConfig.get(key)` returns `data?.[key]`, so a key the view does not store reads `undefined`. `getAll()` returns the live `data` object, or a fresh `{}` while nothing is stored.
- **Schema defaults live in the settings UI only**: each control in the view settings menu shows `config.get(key) ?? option.default`. A reader that wants the same value for an absent key must apply its own default.

### Implication for dynamic defaults

- **Panel and render can disagree**: a schema default computed at runtime (e.g., merged with a settings template) reaches only the settings UI. The control shows it for an absent key while `config.get()` returns `undefined`, so the reader must resolve the same fallback itself.
- **A cleared picker shows the default again**: a property picker clears with `set(key, undefined)`. `get()` then returns `undefined`, the YAML dumper drops the key on save, and the next time the menu is shown the control displays the schema default.

## Bases YAML normalization strips default values

> Observed in Obsidian **1.8.9**, installer 1.7.7.

When Obsidian writes `.base` files, it strips YAML properties whose values match the schema default. For example, setting `rightPropertyPosition: right` (the default) causes the line to be removed entirely on the next write cycle.

### Implication

After testing with a non-default value (e.g., `rightPropertyPosition: column`), reverting to the default in the UI removes the YAML line rather than writing `rightPropertyPosition: right`. Code that checks for the presence of a YAML key to determine whether it was explicitly set cannot distinguish "explicitly set to default" from "never set."

### Workaround

Accept that default-valued properties are absent from YAML, and resolve an absent key against the reader's own defaults at read time. `config.get()` returns `undefined` for it (see "Bases `config.get()` returns the raw stored value" above).

## Bases view options have no change callback

**Observed**: 2026-09-25, API 1.13.1 typings; re-run behavior read from `app.js` 1.14.2.

No `Bases*Option` type — nor the shared `BasesOption` base interface, nor `BasesOptionGroup` — declares an `onChange`; the only per-control hook is `shouldHide?: () => boolean`. A setting that depends on another can therefore only be hidden and ignored when read — never cleared or reset when the other one changes.

- **Re-run trigger**: after every control's own `set(key, value)` call, Obsidian calls `updateHiddenOptions()`, which re-invokes `shouldHide()` for every control across all groups in the panel — not just the control that changed.
- **Group auto-hide**: for a `"group"`-type control that isn't itself hidden by its own `shouldHide()`, `updateHiddenOptions()` also hides the group when every one of its items is hidden — the group stays visible only while at least one item inside it is.
- **Groups cannot nest**: `BasesAllOptions` is `BasesOptions | BasesOptionGroup<BasesOptions>` — a group's `items` are typed as `BasesOptions[]`, the leaf-option union, never `BasesAllOptions[]`. A group cannot contain another group.
