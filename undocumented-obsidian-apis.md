---
title: Undocumented Obsidian APIs
description: Useful undocumented properties and methods discovered through runtime inspection.
author: 🤖 Generated with Claude Code
updated: 2026-09-25
---
# Undocumented Obsidian APIs

Useful undocumented properties and methods discovered through runtime inspection. These are NOT in the official API and may change without notice.

For comprehensive type definitions of Obsidian's internal APIs, search `dist/types.d.ts` in the community-maintained [`obsidian-typings`](https://github.com/Fevol/obsidian-typings) package. Contains 37K lines of typed interfaces for `App`, `DragManager`, `FileManager`, `MetadataCache`, and hundreds more.

## App

- **`app.debugMode`**: Boolean. Toggles Obsidian's internal debug mode, which surfaces extra logging.

## DragManager (`app.dragManager`)

Manages drag-and-drop operations. Key methods:

- **`dragFile(event, file)`**: Creates a draggable for a single `TFile`. Ghost shows **file icon** (`lucide-file`).
- **`dragLink(event, linkText, sourcePath, title?, source?)`**: Creates a draggable for a link. Ghost shows **link icon** (`lucide-link`). `sourcePath` is used for link resolution — pass `''` for absolute vault paths.
- **`dragFolder(event, folder)`**: Creates a draggable for a `TFolder`.
- **`onDragStart(event, draggable)`**: Registers the draggable with the drag manager. Must be called after `dragFile`/`dragLink`/`dragFolder`.
- **`onDragEnd()`**: Cleans up after a drag — detaches the ghost element, clears drag state (`draggable`, `dragStart`, hover/source tracking), and removes `is-grabbing` from `document.body`.

Vanilla Bases uses `dragLink` for card drags (producing a link ghost), not `dragFile`.

```ts
// Card drag — matches vanilla Bases ghost icon
const dragData = app.dragManager.dragLink(e, card.path, '');
app.dragManager.onDragStart(e, dragData);
```

### `onDragEnd()` cleanup wiring, and the `stopPropagation()` gotcha

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2)

DragManager registers its own `dragstart` listener on `window` in the bubble phase, not capture. On every `dragstart` that reaches it, the listener adds a once-only `dragend` listener on the drag source (`event.targetNode`) that calls `onDragEnd()` — this is the only place a normal drag's cleanup gets wired. On mobile, `onDragStart()` additionally adds a `window`-level `touchend` listener that also calls `onDragEnd()` once all touches lift, covering touch-driven drags that never fire a `dragend`.

A plugin `dragstart` handler that calls `event.stopPropagation()` before its own `onDragStart()` call — to keep an inner element's drag from also being handled by an ancestor's listener, for instance — hides that drag from DragManager's bubble-phase `window` listener too, since `stopPropagation()` stops the event before it reaches `window`. DragManager never gets the chance to register its `dragend` cleanup, so the handler must register the same once-only `dragend` → `onDragEnd()` listener on the drag source itself. Skipping this leaves the ghost element and `is-grabbing` on `document.body` after the drop.

## Menu

- **`Menu.addItem(fn)`**: Pushes the `MenuItem` to `menu.items[]` BEFORE calling `fn(item)`. During the callback, the item is already in the array — enabling synchronous reordering within the callback itself.
- **`Menu.addSections(names[])`**: Defines section ordering. At render time (`showAtPosition`), items are grouped and reordered by their section assignment. Without `addSections`, named sections float to the top and the default (empty-string) section goes below.
- **`menu.items`**: Mutable array of `MenuItem` objects. Can be spliced synchronously before `showAtPosition` to reorder items.
- **`MenuItem.menu`**: Back-reference to the parent `Menu` instance. Available during `addItem` callbacks.
- **`MenuItem.titleEl`**: The title element. Separators lack `titleEl`, `iconEl`, and `section` — only have `menu` and `dom`. Useful for detecting separators: `!item.titleEl`.

### File explorer folder context menu sections

Obsidian's built-in file explorer uses these section names for folder context menus. Use `item.setSection(name)` to place items in the correct group. Items without a section (empty string) land at the bottom.

| Section | Items |
|---|---|
| `action-primary` | New note, New folder, New canvas, New base |
| `action` | Duplicate, Move folder to..., Search in folder, Bookmark... |
| `info.copy` | Copy path (from vault folder), Copy path (from system root) |
| `system` | Reveal in Finder |
| `danger` | Rename..., Delete |

Observed in Obsidian 1.12.5.

## Plugins (`app.plugins`)

- **No event system**: `app.plugins` has no events for plugin enable/disable. `workspace.on('layout-change')` does NOT reliably fire on `enablePlugin`/`enablePluginAndSave`. Polling or retry loops are the only option for detecting another plugin becoming available after startup.

## PopoverSuggest and the history stack

Obsidian keeps its own history stack, one per window, behind the patched `window.history.back()`/`forward()` — the stack a mouse back button, Android's back gesture and Electron's `swipe`/`app-command` handling all step through. It is module-private, and there is no documented way to push an entry onto it directly.

`PopoverSuggest`'s `open()`/`close()` do it internally: `open()` pushes an entry onto `activeWindow`'s stack (and pushes the popover's own keymap `Scope` at the same time), `close()` pops both. A subclass that never draws anything — empty `renderSuggestion`/`selectSuggestion`, and `attachDom`/`detachDom` overridden to no-ops so `open()` appends nothing visible to the DOM — is a working handle onto the stack with no UI cost. Implement the public `HistoryHandler` interface alongside it: `onHistoryBack()` is the hook a back press calls on the top-of-stack entry. Leave out the optional `onHistoryForward()`, and a forward press with the entry on top does nothing.

The popover's own scope shadows whatever key bindings were active before it, so when the entry's scope must not intercept anything, pop it immediately after `open()` returns (`app.keymap.popScope(entry.scope)`) — the entry stays on the history stack regardless, since the push and the scope are two separate operations underneath.

**A leaf torn into its own window does not consult its own stack for its back feeders.** Such a popout forwards its `history.back()` and its mouse back/forward presses to the MAIN window, so they resolve through the main window's stack (measured) — an entry pushed only on a leaf popout's own stack is unreachable from the very back feeders a user would expect to trigger it there. Only modal popouts, such as the settings window, keep a stack of their own.

## Scope

- **`Scope.prototype.handleKey(evt, ctx)`**: The dispatch `Keymap.onKeyEvent` calls on the top scope of `activeWindow`'s stack — undocumented, but present at runtime. It tries the scope's entries in order, then its `parent`. A catch-all entry (`register(null, null, fn)`) that returns `undefined` falls through to the next entry; a specific-key entry returns its listener's result even when that is `undefined`, so it never falls through. The built-in hotkey catch-all returns `false` whenever a matching hotkey's command is found and run — even when that command's check callback then does nothing — and `undefined` when no hotkey matches.

  Calling it directly from inside a custom scope's own catch-all handler lets that scope defer to the user's configured hotkeys before running its own key handling: `app.scope` is the keymap root and holds the hotkey catch-all, so `app.scope.handleKey(evt, ctx)` answers "does a hotkey already claim this key?" A custom, single-purpose overlay scope can run this check first and only fall through to its own handling when the result is `undefined` — so a user's own binding for a key a plugin would otherwise also bind (an arrow key, a modifier combination) wins.

### The global `activeWindow` pin

Some API methods — `PopoverSuggest.open()` above among them — read the global `activeWindow` internally to decide which window's own state (a history stack, a scope) they operate on, rather than accepting a window as an argument. A plugin that needs the operation to target a SPECIFIC window regardless of which window is actually focused when the call happens can bracket it: write `window.activeWindow = <target window>` immediately before the call, and restore the previous value in a `finally` block immediately after, so the override never outlives the single call it exists for. Use the property form, `window.activeWindow = …`: a bare `activeWindow = …` trips ESLint's `no-global-assign`, and in strict-mode code it throws a `ReferenceError` wherever the global is not defined, a jsdom test run among them.

## Workspace

### `data-ignore-swipe` attribute

Setting `data-ignore-swipe` on a DOM element prevents Obsidian's mobile touch gesture system from activating when the touch starts on or inside that element. Blocks ALL gesture types — sidebar swipe AND pull-to-action (`mobilePullAction`). Binary flag with no directional control. Undocumented — discovered via source inspection.

**Behavior**:
- Works on the touch target OR any **ancestor** — Obsidian's handler walks from `event.target` up via `parentNode`, returning early when it finds `dataset.ignoreSwipe` on any element in the chain.
- Persists until removed — if set permanently, the element (and its descendants) never trigger any Obsidian touch gestures.
- Blocks both horizontal gestures (sidebar swipe) and vertical gestures (pull-to-action at scroll top). Not directionally selective — there is no way to block sidebar while allowing pull-to-action.

**Obsidian sidebar gesture internals** (relevant when investigating alternatives to `data-ignore-swipe`):
- Obsidian uses **capture-phase** `touchmove` listeners on `document` that apply inline `translateX` on `.workspace-split`, `.mobile-navbar`, `.workspace-drawer.mod-left`, and `.workspace-drawer.mod-right` simultaneously (finger-follow during sidebar transition preview).
- A `workspace.trigger("swipe", { direction, points })` event fires after the gesture completes, where `direction` is `"x"` (horizontal) or `"y"` (vertical) and `points` is the finger count.
- Capture-phase listeners fire before any plugin's bubble-phase handlers. Plugins cannot register capture listeners on `document` that fire before Obsidian's (same-element same-phase listeners fire in registration order — Obsidian registers at app startup before plugins load).

**Use case**: Preventing ALL Obsidian touch gesture interference with custom touch interactions (drag handles, resize dividers, swipeable cards, image viewer pan/pinch).

Observed in Obsidian 1.8.9. Ancestor walk verified via source inspection in 1.12.5. Pull-to-action blocking and capture-phase internals confirmed empirically via CDP diagnostics in 1.8+.
