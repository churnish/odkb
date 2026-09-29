---
title: iOS WebKit quirks
description: Platform-specific bugs in iOS WebKit (WKWebView) affecting content-visibility, IntersectionObserver, CSS scroll-state(), compositor layer shifts, touch hit-testing, long-press image drag, and click synthesis — plus hover media queries on iPad and the native requestUrl transport.
author: 🤖 Generated with Claude Code
updated: 2026-09-29
---

# iOS WebKit quirks

## content-visibility: hidden + IntersectionObserver loop

Toggling `content-visibility: hidden` via IntersectionObserver causes an infinite reflow loop on iOS WebKit (WKWebView). The cycle: IO callback toggles class → geometry changes → IO re-fires → class toggles back → repeat. Flickering is persistent and never stops, even after all user interaction ceases.

**Root cause**: iOS WebKit re-evaluates IntersectionObserver entries when `content-visibility: hidden` changes an observed element's geometry. Chromium does not.

**Fix**: Use `content-visibility: auto` (browser-managed) on mobile instead of IO-driven toggling. Guarded via `Platform.isMobile` at the call site.

**Observed**: 2026-02-11, iOS.

## content-visibility: auto + masonry measurement

`content-visibility: auto` causes iOS WebKit to return the `contain-intrinsic-height` fallback (e.g., 300px) from `offsetHeight` for off-screen cards, even after an initial accurate measurement. This breaks masonry layout — positions are calculated from wrong heights, producing large gaps between cards. Chromium returns accurate heights regardless.

**Root cause**: iOS WebKit reports intrinsic fallback height for off-screen elements with `content-visibility: auto` when read via `offsetHeight`/`getBoundingClientRect()`. A ResizeObserver feedback loop compounds the issue — height changes from `auto` kicking in trigger re-measurement with wrong values.

**Fix**: Add `.masonry-measuring` class (which forces `content-visibility: visible !important`) around height reads in both the full layout and incremental (infinite scroll) layout paths.

**Observed**: 2026-02-11, iOS.

## CSS scroll-state() container queries

`CSS.supports('container-type', 'scroll-state')` returns `false` on iOS WebKit (tested 2026-02-11, Obsidian early access via TestFlight). The `@container scroll-state(stuck: top)` rule is silently ignored — progressive enhancement, no errors.

## Opacity on `<a>` inside absolute-positioned elements

Changing `opacity` on an `<a>` element inside a `position: absolute` container causes iOS WKWebView to momentarily shift the container horizontally before snapping back. The shift is visible to users but undetectable by JavaScript (`getBoundingClientRect()` reports no change) — it occurs at the compositor layer.

**Trigger**: Obsidian's `a.mobile-tap { opacity: 0.5 }` rule (applied on touchstart, removed on touchend) on card title `<a>` elements inside masonry cards.

**Fix**: Override with `.card-title a.mobile-tap { opacity: 1; filter: opacity(0.5); }` — `filter: opacity()` is visually identical but uses a different compositor pipeline that doesn't trigger the shift. Do NOT add `transition: filter` — animating the filter also triggers the same compositor shift.

**Observed**: 2026-02-21, iOS.

## iOS text selection loupe blocks custom long-press interactions

iOS's native text selection loupe (magnifier) is driven by `UILongPressGestureRecognizer` at the UIKit layer — it operates BEFORE web events reach JavaScript. Any custom long-press interaction (hold-to-drag, long-press menus) on or near text content will trigger the loupe after ~300ms.

**What does NOT work**:
1. `selectstart` `preventDefault()` — the event never fires on iOS for touch selections.
2. `user-select: none` on parent containers — iOS checks the element directly under the finger, ignoring parent rules.
3. `getSelection().removeAllRanges()` on pointermove — loupe is already shown by the time JS runs.
4. `pointermove` `preventDefault()` in capture phase — pointer events are too late in the event chain.
5. Full-screen overlay with `pointer-events: auto` — blocks all input.
6. Overlay with `pointer-events: none` — iOS ignores it.

**Fix**: `touchmove` `preventDefault()` in **capture phase** during the interaction. Touch events operate closer to UIKit's native gesture recognizer layer than pointer events — this is the only mechanism that reaches the native layer to block the loupe.

```js
const blockTouchMove = (ev) => ev.preventDefault();
document.addEventListener('touchmove', blockTouchMove, {
  passive: false,
  capture: true,
});
// Remove on interaction end
```

**Observed**: 2026-03-09, iPadOS 18.

## Long-press image drag retargets on `draggable="false"` rather than cancelling

WebKit's long-press-to-drag affordance on `<img>` is a separate mechanism from HTML5 drag-and-drop, and it answers to none of the properties that switch the desktop path off. `draggable="false"` is the one that does reach it — as a redirect, not an off switch.

| Property | Governs | Effect on long press |
|---|---|---|
| `draggable="false"` on the image | The desktop HTML5 DnD path | **Retargets** — the drag moves to the nearest draggable ancestor |
| `-webkit-user-drag: none` | The same desktop path, via CSS | None observed |
| `touch-action` on the image | That element's own default touch behaviour | None — neither `pan-y` nor `none` stopped `dragstart` |
| `touch-action: none` on a container | Default touch behaviour over the container's own box | The only case the suppression was ever demonstrated for, and it does not reach descendants |

**`draggable="false"` retargets, it does not cancel.** WebKit drops the image as the drag source and hands the press to the nearest draggable ancestor, which then lifts with its own payload. An image inside a draggable card therefore drags the card, and that retargeting is the fix for card images that previously lifted a bare image with nothing attached.

**`touch-action` is not inherited.** A rule on a container never lands on the image the finger is actually pressing, so every container-level `touch-action` written to suppress this missed the pressed element. The container demonstration below is therefore not evidence about the image.

**`setDragImage()` is accepted but ignored.** On a WebKit system-initiated drag, calling it on the event's `dataTransfer` neither throws nor changes anything — the preview stays WebKit's own snapshot of the pressed element. A custom drag ghost has to come from somewhere else.

**Symptom when nothing opts out**: a long press starts a system drag and fires the drag-lift haptic, then aborts with no payload if the element has no draggable data. The user feels a vibration for a drag that never happens.

**Fix**: set `draggable="false"` on the image and give the ancestor that should be dragged the payload you want lifted. `touch-action` on the image is not a substitute — measured at both `pan-y` and `none`, neither stopped `dragstart`. Obsidian's own image lightbox sets `touch-action: none` on its media container *and* `draggable="false"` at element creation; only the second of those reaches the image.

**Diagnostic trap**: an ancestor's `touch-action` reads like coverage and is not. Because the property does not inherit, a grep that finds `touch-action: none` on the container and stops there reports an opt-out the image never receives. Check the rule that actually matches the pressed element.

**Scope carefully.** `touch-action: none` disables *all* default touch behaviour over the box it applies to, including any long-press drag you actually want. Where a long press is a real affordance — dragging an image out into a document, for instance — the opt-out must be scoped to the modes that do not offer it.

**Observed**: 2026-08-10, corrected 2026-08-28 on iOS. The original reading — `draggable` inert, `touch-action` the only property that reaches the affordance — was inferred from the host app's own suppression rather than measured. The corrections above are measured.

## WKWebView compositor touch routing bypasses main-thread hit-test

WKWebView's compositor/scrolling thread maintains its own hit-test tree for touch routing that is **disconnected** from the main thread's hit-test tree (used by `elementFromPoint()` and synthesized clicks). A `position: fixed` element can pass `elementFromPoint()` checks on the main thread while the compositor routes the actual touch to a different element underneath.

**Example**: `elementFromPoint(200, 20)` returns `.view-header-title` (inside a fixed header), but the actual `touchstart` event lands on `.masonry-container` (scroll content below). The compositor's stale hit-test tree doesn't reflect recent `classList` or style changes on fixed elements.

**Implications**:
- **`elementFromPoint()` is unreliable** for verifying touch reachability on iOS — it tests the main-thread tree, not the compositor tree.
- **`classList` changes on `position: fixed` elements** (e.g., adding a class that changes `pointer-events`, `transform`, `opacity`) do NOT synchronously update the compositor hit-test regions. Neither `offsetHeight` flush nor `elementFromPoint()` reliably forces a compositor hit-test tree rebuild.
- **Inline style changes** (`element.style.pointerEvents = 'auto'`) are more reliable than class-based changes for affecting compositor touch routing, though not guaranteed.

**Workaround**: Control touch interception via structural properties (e.g., `min-height` to expand/collapse the layout box) rather than `pointer-events` class toggles. The compositor respects layout geometry more reliably than style-only changes.

**Observed**: 2026-04-02, iOS 26.4.

## WKWebView ignores `preventDefault()` on touchstart for click synthesis

Calling `preventDefault()` on a non-passive `touchstart` listener (even with `capture: true`) does **NOT** prevent WKWebView from synthesizing a compatibility click event. This contradicts the spec and Chromium behavior, where `preventDefault()` on `touchstart` suppresses the entire click sequence.

**Trigger**: Touch a `position: fixed` element → JS handler calls `showBarsUI()` which changes layout → WKWebView synthesizes a click ~300ms later on the original touch target (now potentially repositioned or behind other content).

**Workaround**: Use `preventDefault()` on `touchend` (not `touchstart`) via a non-passive listener — this reliably suppresses click synthesis on WKWebView. Alternatively, register a one-shot capture click listener that calls `stopPropagation()` + `preventDefault()` to eat the synthesized click.

**Observed**: 2026-04-02, iOS 26.4.

## iPadOS CSS cursor property

iPadOS does not support the CSS `cursor` property (except `text`). Apple locks cursor appearance at the system level — no web API workaround exists. Custom cursors (e.g., `cursor: col-resize` on drag handles) have no effect.

**Observed**: 2026-03-09, iPadOS 18.

## An iPad with a hover-capable input matches `any-hover`

**Observed**: 2026-09-25, iPadOS, version not recorded — an iPad Air M3, in Obsidian

| Media query | iPad with a hover-capable input |
|---|---|
| `(any-hover: hover)` | Matches |
| `(any-pointer: fine)` | Matches |
| `(hover: hover)` | No match |
| `(pointer: fine)` | No match |
| `(pointer: coarse)` | Matches |

The probe does not say which input made the `any-*` queries match — a hovering Apple Pencil or a trackpad — so the claim that either one makes `any-hover` match on iPadOS is confirmed for at least one of the two. An iPhone matches none of the hover or fine queries.

- **`any-hover` does not rule out touch**: a rule or check that gates touch behavior on `any-hover` also governs finger input on such an iPad. A script check that reads the same two `any-*` queries through `matchMedia()` is true there too.
- **Name touch states in every hover lock**: when an `@media (any-hover: hover)` block locks behavior behind a desktop hover state, a state the touch tap path sets must also be named inside it, or the behavior stays locked on such an iPad.

See `obsidian-ui-internals.md` for how Obsidian's own press feedback treats a trackpad hover and a Pencil hover on a tablet.

## `requestUrl` is a native call on iOS, not WKWebView networking

**Observed**: 2026-09-25, iOS, version not recorded; the `app.js` names are from Obsidian 1.14.2

On iOS, Obsidian's `requestUrl` is the Capacitor `App.requestUrl` native call (`Sw = registerPlugin('App')` in `app.js`), not a WKWebView fetch.

- **Outside the page's cache and tools**: it neither shares the page's HTTP cache nor appears in the Web Inspector network panel.
- **Several full-size copies per response**: the body crosses the native bridge as base64 text, which `app.js`'s `wb` turns back into bytes through `atob`, a char-by-char `Uint8Array` fill and its `ArrayBuffer`. An image therefore passes through several full-size copies in the JS heap before the caller can even build a `Blob` from it.
- **Failure messages**: `Request failed. The request timed out.` after about 60s, and `Request failed. The network connection was lost.`
- **No patching from the console**: a Capacitor plugin object is a Proxy whose get trap returns a fresh wrapper on every read, so its methods cannot be monkey-patched from the console. Timing iOS requests needs instrumentation built into the plugin itself.

On desktop, `requestUrl` is an IPC call to Electron's main process instead — see `undocumented-obsidian-apis.md`.

### Telling a memory kill from a crash

A memory kill leaves a `JetsamEvent-…ips` file under Settings → Privacy & Security → Analytics & Improvements → Analytics Data, naming the killed process. A genuine crash leaves a report named after the app instead.
