---
title: Undocumented Obsidian APIs
description: Useful undocumented properties and methods discovered through runtime inspection.
author: 🤖 Generated with Claude Code
updated: 2026-09-29
---
# Undocumented Obsidian APIs

Useful undocumented properties and methods discovered through runtime inspection. These are NOT in the official API and may change without notice.

For comprehensive type definitions of Obsidian's internal APIs, search `dist/types.d.ts` in the community-maintained [`obsidian-typings`](https://github.com/Fevol/obsidian-typings) package. Contains 37K lines of typed interfaces for `App`, `DragManager`, `FileManager`, `MetadataCache`, and hundreds more.

## App

- **`app.debugMode`**: Boolean. Toggles Obsidian's internal debug mode, which surfaces extra logging.

### `app.getObsidianUrl(file)` encodes the path once

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2), read from `app.js`

`app.getObsidianUrl(file)` builds `obsidian://open?vault=…&file=…`, passing the file path through `encodeURIComponent` exactly once.

- **Only Markdown files lose their extension**: it strips `.md` from a Markdown file's path; any other file keeps its extension.
- **Never decode a second time**: `URLSearchParams.get('file')` already decodes the path once. A second `decodeURIComponent` throws `URIError` on a literal `%` in a file name, and turns a name's own `%20` into a space.
- **Resolving the file back**: try the path plus `.md` first, then the path as given.

## Bases query controller

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2)

Every rendered base — standalone, embedded with `![[File.base]]`, or written as a `base` code block — is driven by a query controller. For its settings menu, see "Bases view settings menu" below. For why a plugin reload leaves a stale view in place, see `obsidian-api-quirks.md`.

### One config object per view

`controller.query.views` holds one distinct config object per view in the file, and `controller.view.config` is identity-equal to one of them. Probed on a two-view base: two entries, two distinct objects, the live view's config among them.

- **A view switch always changes `config` identity**: the view showing after a switch holds a different config object from the one before it — in standalone bases and embeds alike.
- **Identity alone tells views apart**: code that needs to know which of a base's views a config belongs to can compare config objects, with no view id.

### Finding every controller

Walking `_children` — each `Component`'s list of child components — down from every `leaf.view` reaches every controller:

- **Every form**: standalone bases, embedded bases and code block bases.
- **Hidden ones too**: in Reading view, and in the editor Obsidian keeps rendered but hidden behind it — plus a detached controller that was still loaded.
- **Except on a canvas**: a canvas's embeds hang off `leaf.view.canvas.nodes` → `node.child`, not off `_children`.

Measured: the walk across 19 leaves took 0.4ms.

### Replacing a view in place

`update()` builds a new view only when the live view's `type` differs from its config's. To force a rebuild — of a view left over from an earlier load of the plugin, say — run the steps Obsidian's own `selectView` runs, in the same order:

```js
controller.removeChild(controller.view);
controller.view = null; // nulling the view is what makes update() build a new one
controller.viewContainerEl.empty();
controller.update();
```

Measured on macOS desktop after a real plugin reload:

- **The swap takes**: the new view was an instance of the current module's class, the old one unloaded, and 8 swaps raised no errors.
- **The rebuilt view starts at the top**: two views scrolled to 1380px and 1400px both came back at 0.
- **Ephemeral state carries the position**: bracketing the swap with `getEphemeralState()` before and `setEphemeralState(state)` after brought a popout scrolled to 1195px back with its anchor item at exactly its saved 28px offset. During an in-place reload, though, the old view's scroll drifts before the plugin's own code can snapshot it — see `obsidian-api-quirks.md`.
- **A hidden view gets no data**: Bases feeds a view in the hidden editor behind Reading view no data at all. Rebuilt views there had `data` unset and rendered nothing; switching the note to Live Preview then rendered 20 and 2 items in the same views.

## Bases view settings menu (`viewMenu`)

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2)

Driving a Bases view's settings menu ("Configure view") from a probe, rather than through the GUI:

- **The open settings page**: the Bases query controller's `viewMenu.pageStack.last()` is the currently open settings page. Its `.view` is the view config object the page edits, held for the whole lifetime of the page — see `obsidian-api-quirks.md` for why that reference can go stale after a reload.
- **Opening it synthetically**: a synthetic `contextmenu` `MouseEvent` dispatched on `viewMenu.toolbarItem.button.buttonEl` opens the settings page for the current view (verified on an embedded Bases view).
- **Switching layout persists only through the dropdown's own handler**: replaying it — `page.view.type = '<type>'; page.display(page.view); page.controller.query.save()` — reaches disk, because it goes through the query's own save function. Assigning `config.type` directly and calling `setQueryAndView` does not.
- **CDP clicks on the Layout dropdown are unreliable**: clicking the Layout combobox's suggestion items via CDP times out as "not interactive". The popover also renders into the app's currently active window, which may not be the window the base itself is open in.
- **Capturing an embed's controller**: wrap the controller prototype's `update` method — taken from any standalone Bases leaf's controller, since the prototype is shared — and filter invocations on `this.query.file.path`. One embed rendered in Reading view can construct several controllers; only one of them is the one actually visible.

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

## History stack

**Observed**: 2026-09-24 to 2026-09-26, Obsidian 1.14.2 (installer 1.14.2) — read from `app.js` and the app's Electron main-process script; the popout behavior measured

Obsidian keeps its own history stack, one per window, behind the patched `window.history.back()`/`forward()`/`go()` — the stack a mouse back button, Android's back gesture and Electron's `swipe`/`app-command` handling all step through. It is module-private, and there is no documented way to push an entry onto it directly. Its entries implement the public `HistoryHandler` interface, as `Menu`, `Modal` and `PopoverSuggest` do.

### How it steps

- **Back calls the top entry and pops nothing**: `history.back()` calls `onHistoryBack()` on the window's top entry; the entry is expected to pop itself as it closes. `forward()` calls the top entry's `onHistoryForward()` when it has one, and `go(n)` repeats either one `|n|` times.
- **A pop removes the entry wherever it sits**: popping takes the entry out of the stack it was pushed on, even when it is not on top.
- **The desktop main window has a base entry**: its stack starts with a permanent entry that steps the workspace's active leaf back or forward. With nothing else pushed, a back press navigates the active leaf — so an overlay that pushed no entry of its own stays on screen while the note behind it changes, or is torn down with a view that unloads.

### What feeds it

| Feeder | Where | Resolves through |
|---|---|---|
| Android back button | Android | The main window's stack. When that is empty: collapse the left sidebar, then the right one (a pinned sidebar is skipped), then step the active leaf back, then show "back again to exit" — a second press within 5s minimizes the app |
| Mouse back/forward buttons (`button` 3/4) | Desktop, except Linux | A capture-phase `mousedown` listener on the main window: `preventDefault()`, `stopPropagation()`, then the patched `history.back()`/`forward()` |
| Electron `swipe` (left/right) and `app-command` (`browser-backward`/`browser-forward`) | Desktop | The main process runs `history.back()`/`forward()` in that window's page — registered for popout windows too, not only the main one |
| A leaf popout's `history.back()`/`forward()`/`go()` | Desktop | Replaced with forwarders to the main window's |
| A leaf popout's `mousedown` | Desktop | Relayed to the main window as a copy whose `preventDefault()`/`stopPropagation()` reach the original, so the main window's back-button listener handles it |
| A modal popout (`body.is-popout-modal`) | Desktop | Its own patched stack and its own capture-phase back-button listener |

**A leaf torn into its own window does not consult its own stack for its back feeders.** Such a popout forwards its `history.back()` and its mouse back/forward presses to the MAIN window, so they resolve through the main window's stack (measured) — an entry pushed only on a leaf popout's own stack is unreachable from the very back feeders a user would expect to trigger it there. Only modal popouts, such as the settings window, keep a stack of their own.

### What pushes onto it

| Pusher | Pushes onto |
|---|---|
| `Menu` — non-native menus only | The window of the document it shows in (`activeDocument` by default) |
| `Modal.open()` | `activeWindow` |
| `PopoverSuggest.open()` | `activeWindow` |
| The image lightbox | `activeWindow` |
| The mobile tab switcher | The main window |

- **An entry lands on the active window's stack, which a leaf popout never reads**: a real click in a popout makes it `activeWindow` (through the popout's `focus` listener). The native image lightbox opened by such a click pushes onto the popout's own stack, which none of the feeders above read, so a back press in that popout skips the lightbox and reaches the main window's stack instead (measured).
- **The lightbox stays on top while it animates out**: it pops its entry only when its close animation ends, and ignores a close while already closing — so a second back press during the animation does nothing.
- **Probe precondition**: CDP `Input.dispatchMouseEvent` does not fire the popout's `focus` event, so under CDP input `activeWindow` stays the main window and a push lands on the main stack — dispatch `focus` on the popout window first. Read `activeWindow` from the main window's realm: each popout realm has its own `activeWindow` global, which always equals that popout.

### `PopoverSuggest` as a handle onto the stack

`PopoverSuggest`'s `open()`/`close()` do it internally: `open()` pushes an entry onto `activeWindow`'s stack (and pushes the popover's own keymap `Scope` at the same time), `close()` pops both. A subclass that never draws anything — empty `renderSuggestion`/`selectSuggestion`, and `attachDom`/`detachDom` overridden to no-ops so `open()` appends nothing visible to the DOM — is a working handle onto the stack with no UI cost. Implement the public `HistoryHandler` interface alongside it: `onHistoryBack()` is the hook a back press calls on the top-of-stack entry. Leave out the optional `onHistoryForward()`, and a forward press with the entry on top does nothing.

The popover's own scope shadows whatever key bindings were active before it, so when the entry's scope must not intercept anything, pop it immediately after `open()` returns (`app.keymap.popScope(entry.scope)`) — the entry stays on the history stack regardless, since the push and the scope are two separate operations underneath.

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

## MetadataCache (`app.metadataCache`)

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2), read from `app.js`

- **`getBacklinksForFile(file)` walks the whole vault**: every call walks every reference in the vault, so calling it once per file costs one whole-vault walk per file.
- **`iterateAllRefs(callback)` visits every reference once**: it calls back with `(sourcePath, ref)` for each Markdown file's frontmatter links, links and embeds, then for every canvas's references.
- **One walk can index a whole batch**: resolving each `ref.link` as `getFirstLinkpathDest(getLinkpath(ref.link), sourcePath)` reproduces `getBacklinksForFile`'s own resolution, and each file's sources come out in the same first-seen order `getBacklinksForFile` reports.
- **`resolvedLinks` is not a substitute**: its sources are ordered by when each file was resolved, not by the walk, and it lags edits behind an async queue.

```ts
// One walk for a batch of files: target path → source paths, in first-seen order
const sourcesByTarget = new Map<string, string[]>();
app.metadataCache.iterateAllRefs((sourcePath, ref) => {
  const target = app.metadataCache.getFirstLinkpathDest(getLinkpath(ref.link), sourcePath);
  if (!target || !batchPaths.has(target.path)) return;
  const sources = sourcesByTarget.get(target.path) ?? [];
  if (!sources.includes(sourcePath)) sources.push(sourcePath);
  sourcesByTarget.set(target.path, sources);
});
```

## Plugins (`app.plugins`)

**Observed**: 2026-09-25 to 2026-09-29, Obsidian 1.14.2 (installer 1.14.2) — the event order measured during an in-place reload, the call sites read from `app.js`

- **`changed` fires on enable and disable**: `app.plugins` is an `Events` emitter. It triggers `changed` through a 0ms debounce after `enablePlugin()` succeeds (app.js 1.14.2 L171423) and after `disablePlugin()` unloads a plugin (L171442), among other call sites. The event carries no arguments, so a listener re-checks the plugin it is waiting for — no polling needed. During an in-place reload one fires after the disable and one after the enable; see `obsidian-api-quirks.md` for where they land relative to the plugin's stylesheet.
- **`workspace.on('layout-change')` does NOT reliably fire** on `enablePlugin`/`enablePluginAndSave`.

## `requestUrl` on desktop

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2)

On desktop, `requestUrl` runs in Electron's main process. The renderer sends each request as `require('electron').ipcRenderer.send('request-url', id, request)` and takes the answer on `ipcRenderer.once(id, …)` (app.js 1.14.2 L49776–49793). On iOS it is a native Capacitor call instead — see `ios-webkit-quirks.md`.

- **Invisible to the page's network tools**: the page's CDP Network panel and network throttling never see these requests.
- **Logging every request**: `send` lives on the prototype, so an own-property wrapper on the `ipcRenderer` instance sees every call, and `delete ipc.send` restores the original. It is the only per-request view available.
- **Settle times**: the wrapper can register its own `ipc.once(id, …)` per request and record when it settled and with what status. A start and an end per request give the in-flight count at every moment — enough to check a cap on concurrent downloads.
- **Popout requests included**: plugin code runs in the main window's realm, so the wrapper also sees requests made on behalf of a popout.
- **Precondition**: desktop only. Mobile has no IPC transport to wrap.

```js
const ipc = require('electron').ipcRenderer;
const send = ipc.send; // the prototype's method
ipc.send = function (channel, id, request, ...rest) {
  if (channel === 'request-url') {
    const startedAt = performance.now();
    ipc.once(id, (_event, ...reply) => {
      console.log(request, startedAt, performance.now(), reply);
    });
  }
  return send.call(this, channel, id, request, ...rest);
};
// Restore the prototype's method with `delete ipc.send`
```

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
