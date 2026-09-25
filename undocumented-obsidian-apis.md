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
