---
title: Electron popout window quirks
description: Platform-specific quirks when running plugin code in Electron popout (BrowserWindow) windows — observers, animation frames, hit-testing, element creation, image downloads and cleared image sources.
author: 🤖 Generated with Claude Code
updated: 2026-09-29
---

# Electron popout window quirks

Obsidian's "Open in new window" creates an Electron `BrowserWindow` (popout) with its own document and its own `window` global — a separate JavaScript realm, with its own constructors — in the same renderer process and V8 isolate as the main window. Plugin JS runs in the main window's context but operates on DOM elements in the popout's document. This causes several non-obvious issues.

## Module-scope `document`/`window` resolve to main window

All bare references to `document`, `window`, `requestAnimationFrame`, `ResizeObserver`, etc. at module scope resolve to the main window. For popout-aware code, derive from the element:

```ts
const doc = el.ownerDocument;
const win = doc.defaultView ?? window;
win.requestAnimationFrame(() => { ... });
new win.ResizeObserver((entries) => { ... });
```

**Affected APIs**: `document.body`, `document.activeElement`, `document.hasFocus()`, `document.createElement()`, `window.innerWidth/innerHeight`, `window.focus()`, `ResizeObserver`, `IntersectionObserver`, `requestAnimationFrame`, `addEventListener` on `document`/`window`.

## `createEl` on leaf DOM builds in the main window's document

**Observed**: 2026-09-26, Obsidian 1.14.2 (installer 1.14.2), Electron 43

Obsidian's `enhance.js` runs in every window, and each run gives that window's `Node.prototype` a `createEl` that calls the same window's global `createEl`. So `el.createEl()` and `el.createDiv()` build the new element in the document of the window whose prototype `el` inherits — `el.constructorWin` — which is not always `el.doc`.

- **Leaf DOM inherits from the main window**: every workspace item's `containerEl` comes from the main window's global `createDiv()`, and a view's `containerEl` from `leaf.containerEl.createDiv(...)`. Those elements keep main-window prototypes after the leaf moves to a popout — they pass the main window's `instanceof HTMLElement` while their `ownerDocument` is the popout's — so anything built from them with `createEl`/`createDiv` starts in the main window's document and moves into the popout's on append.
- **Options apply before the append**: the global `createEl` creates the element, applies `cls`, `text`, `attr` and the other options, runs the callback, and only then appends it to `parent`.
- **A `src` in `attr` loads in the wrong document**: `el.createEl('img', { attr: { src } })` in a popout starts the request in the main window's document, before the element moves. Downloads are shared only within one document (next section), so that request is not shared with a load of the same URL in the popout.

**Fix**: append first, then set `src`.

```ts
// Wrong: the request starts in the main window's document
const img = el.createEl('img', { attr: { src: url } });

// Right: the element is in its final document before it loads
const img = el.createEl('img');
img.src = url;
```

**Diagnostic**: `el.constructorWin !== el.win` is `true` for main-built DOM inside a popout.

## Image downloads are shared only within one document

**Observed**: 2026-09-26, Electron 43 (Chromium 150)

All of Obsidian's windows run in one renderer process, yet Chromium shares an in-flight image download — and a finished `no-store` response — only between elements of the same document. The same URL requested from another window's document is a second download.

| Same URL, loaded by | Requests |
|---|---|
| The main window's `new Image()`, then a popout's `img` 1s later | 2 |
| `new popoutWindow.Image()` (constructed from main-window code), then the popout's `img` | 1 |

- **Build a prefetch on the element's own window**: a prefetch meant to share its download with an element must use that element's window's constructor — `new el.win.Image()` — not the module-scope `Image`, which belongs to the main window.
- **A finished `no-store` response stays servable only while referenced**: it is served again only within its own document, and only while some element still references it. Once the last reference is dropped and a garbage collection runs, the URL is requested again.
- **A cacheable response crosses windows, but not synchronously**: an `http` response that allows caching comes back from the HTTP cache in any window, yet the element reads `complete` false right after its `src` is set.

Measured against a local server that logs how each response ended. Use a fresh URL per trial: within one document, a repeated request for the same URL is served from the earlier download. See `image-loading-quirks.md` for how downloads behave within a document.

## A cleared `img` in a popout reads back the opener's URL

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2), Electron 43.7.1

A popout is an `about:blank` document that takes its opener's base URL. After `img.src = ''`, the `src` getter returns `app://obsidian.md/index.html`, while the popout's own `location.href` is `about:blank` (measured).

- **Test the attribute**: detect a cleared image with `!img.getAttribute('src')`. NEVER compare the `src` getter with a page URL.
- **Comparing with the owner window's URL fails**: a guard that compares `img.src` with the popout's own `location.href` never matches, so every cleared-image check built that way fails in popouts.
- **Comparing with the main window's URL passes by accident**: the main window's `location.href` is the base URL the popout inherited, so that comparison works without being correct.

Precondition for reproducing it: an image in a popout. In the main window both URLs are the same, so either comparison passes there.

## Cross-context observers silently fail

`ResizeObserver` and `IntersectionObserver` created in the main window's JS context silently fail to observe elements in a popout's DOM. The constructor must come from the popout's window:

```ts
// Wrong: uses main window's RO constructor
new ResizeObserver(callback).observe(popoutElement);

// Right: uses popout's RO constructor
const win = popoutElement.ownerDocument.defaultView ?? window;
new win.ResizeObserver(callback).observe(popoutElement);
```

Diagnostic: `ownerDocument.defaultView.ResizeObserver !== window.ResizeObserver` → `true` in popouts.

## `requestAnimationFrame` IDs are per-window

`cancelAnimationFrame(id)` must be called on the same window where `requestAnimationFrame(cb)` was called — RAF IDs are scoped to the window that issued them. Canceling an ID on a different window is a silent no-op.

When tearing down state for a popout move, cancel all pending RAFs BEFORE nullifying the stored window reference. Otherwise, the `?? window` fallback targets the main window and the cancel does nothing — the orphaned callback fires in the old window's context.

```ts
// Wrong: observerWindow already null, cancel targets main window (no-op)
this.observerWindow = null;
(this.observerWindow ?? window).cancelAnimationFrame(this.rafId);

// Right: cancel while observerWindow still points to the popout
(this.observerWindow ?? window).cancelAnimationFrame(this.rafId);
this.rafId = null;
this.observerWindow = null;
```

## Main-window RAF does not run before popout paint

Each `BrowserWindow` renders on its own frame schedule, even though all windows share one renderer process. A `requestAnimationFrame` callback queued on the main window runs before the main window's paint — but does **not** block the popout's paint. If a `MutationObserver` detects a DOM change in a popout and defers work to main-window RAF, the popout renders one frame without the update, causing visible flicker.

**Workaround**: For popout-visible DOM updates triggered by MutationObserver, process synchronously in the MO callback instead of deferring to RAF. MO already batches mutations internally, so the frequency is equivalent to RAF. Alternatively, use the popout's own RAF via `doc.defaultView.requestAnimationFrame`.

```ts
// Wrong: main-window RAF — popout paints a frame without the update
const observer = new MutationObserver(() => {
  requestAnimationFrame(() => updateDOM());
});

// Right: process synchronously — update is visible on the popout's next paint
const observer = new MutationObserver(() => {
  updateDOM();
});

// Also right: use the popout's own RAF
const win = doc.defaultView ?? window;
const observer = new win.MutationObserver(() => {
  win.requestAnimationFrame(() => updateDOM());
});
```

**Observed**: 2026-04-14, Obsidian 1.12.7

## Enumerating popout documents via `floatingSplit`

`app.workspace.floatingSplit.children` is an array of popout window containers. Each child has a `doc` property (the popout's `Document`) and a `win` property (its `Window`). Use this to iterate all open documents:

```ts
function getAllDocuments(): Document[] {
  const docs: Document[] = [document];
  const floating = (app.workspace as any).floatingSplit?.children;
  if (floating) {
    for (const child of floating) {
      if (child.doc?.defaultView) docs.push(child.doc);
    }
  }
  return docs;
}
```

- **`defaultView` guard**: Filter by `child.doc?.defaultView` to exclude documents from already-closed windows. Without this, `doc.body` may be null, causing observer setup to throw.
- **Cross-context `instanceof`**: `child.doc instanceof Document` returns `false` because the popout's `Document` comes from a different realm, with its own `Document` constructor. The object is a real `Document` — use duck typing or skip the check.
- **Undocumented API**: `floatingSplit` is not in Obsidian's public type definitions. It has been stable across Obsidian 1.8–1.12.

**Observed**: 2026-04-14, Obsidian 1.12.7

## renderHash must be invalidated on document change

When a view moves between windows, `handleDocumentChange` tears down observers, but the underlying data hasn't changed. If the render pipeline uses a hash-based early return to skip redundant re-renders, the hash must be invalidated — otherwise the pipeline hits the early return, skipping observer re-creation in the new window context. CSS Grid views survive because they auto-reflow without JS; absolutely-positioned views (masonry) require explicit observer-driven layout.

```ts
// In handleDocumentChange:
this.teardownObservers();
this.renderState.lastRenderHash = ''; // Force pipeline to fall through
```

## No re-hit-test after overlay removal

After removing a DOM overlay (`cloneEl.remove()`), Electron popout windows do **not** recalculate `:hover` or dispatch `mouseenter`/`mouseleave` on elements that were underneath. In the main window, `:hover` updates after a `requestAnimationFrame`; in popouts, it stays stale indefinitely.

**Impact**: Any code that checks `element.matches(":hover")` or waits for `mouseenter` after removing an overlaying element will get incorrect results in popouts.

**Workaround**: Apply state changes (e.g., class additions) directly rather than depending on browser re-hit-testing.

## Panzoom `isAttached` check

The `@panzoom/panzoom` library's `isAttached` check walks up the DOM to find `document` (module scope). In popouts, the element is in a different document, so the check fails. Workaround: temporarily reparent the container to `document.body` during init, then move it back.

## Event listener binding

Libraries that bind event listeners to module-scope `document` (e.g., `pointermove`, `pointerup` for drag handling) will miss events in popouts since pointer events fire on the popout's document. Must rebind to the popout's document after init.

## `defaultView` is null after window close

When a popout's `BrowserWindow` is closed, `ownerDocument.defaultView` returns `null` for elements that were in that document. Cleanup code that derives the window via `containerEl.ownerDocument.defaultView` must null-check — otherwise a `?? fallback` pattern may trigger unintended global behavior (e.g., cleaning up all windows' observers instead of just the closed one).

```ts
// Wrong: falls through to global cleanup when window is gone
cleanupObserver(el.ownerDocument.defaultView ?? undefined);

// Right: skip cleanup — observer dies with its window
const win = el.ownerDocument.defaultView;
if (win) cleanupObserver(win);
```

## A popout closed mid-load leaves its loads unsettled

**Observed**: 2026-09-25 to 2026-09-26, Electron 43 (Chromium 150)

- **An `Image` never settles**: when a popout closes while an `Image` built on its window is still downloading, Chromium aborts the download, and the `Image` fires neither `load` nor `error` (none within 9s). It then reads `complete` true with `naturalWidth` 0. Code that awaits `load`/`error` hangs — bound the wait with a timeout, or abort it together with the view.
- **Its `requestAnimationFrame` never fires**: a closed popout's `requestAnimationFrame` still returns an id and does not throw, but the callback never runs. Schedule teardown work that must complete on the main window.

## Style Settings classes in popout windows

Style Settings plugin syncs `class-toggle` and `class-select` settings to ALL open documents (main + popouts). Changes made after popout creation DO reflect in popout windows — Style Settings handles this internally.

However, module-scope `document.body` (main window) remains the canonical source for reading configuration classes, because:

- It's always available (popout may not exist yet)
- It avoids coupling to a specific popout's document lifecycle

```ts
// Right: reads from main window body (canonical source)
const hasSettingClass = document.body.classList.contains('my-plugin-setting');

// Right: creates node in correct document context
const textNode = cardEl.ownerDocument.createTextNode(title);
```

**Observed**: 2026-03-12, Obsidian 1.8.9, Style Settings 1.0.9

## Obsidian recreates `.view-content` during popout move

When a leaf is moved to a popout window (right-click → "Open in new window"), Obsidian destroys and recreates the `.view-content` element. Any inline styles set on `.view-content` before the move are lost. The view instance itself survives — only the DOM subtree is rebuilt.

Additionally, Obsidian mirrors the main window's body classes to the popout's body at creation time. This means body-class-gated CSS rules apply immediately once stylesheets load, even before any plugin JS runs in the popout context.

**Observed**: 2026-03-12, Obsidian 1.8.9

## Workspace event timing in popouts

All workspace events (`window-open`, `layout-change`) fire at ~300-380ms after popout creation. The first paint happens before any JS event fires, so workspace events cannot prevent FOUC. Style injection must happen synchronously during popout creation (e.g., via `window-open` event on the main window's `workspace`) or through CSS-only solutions.

**Observed**: 2026-03-12, Obsidian 1.8.9

## ResizeObserver doesn't fire for minimized windows

Electron does not dispatch `ResizeObserver` callbacks for minimized `BrowserWindow` instances. The window must be restored/shown before testing RO-based behavior. This applies to both main and popout windows.
