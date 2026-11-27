---
title: WebKit compositor constraints
description: Momentum-killing APIs, scrollTop write behavior, double-rAF pattern, scrollend timing, layer promotion, scrolling-tree stale geometry during container resize, touch event suppression during momentum, touch-action railing differences, non-passive touchmove scroll blocking, mid-gesture overflow immunity, and touch-action evaluation timing — constraints for any plugin doing scroll-concurrent DOM work or touch gesture handling on iOS/iPadOS.
author: 🤖 Generated with Claude Code
updated: 2026-04-07
---
# WebKit compositor constraints

WebKit on iOS and iPadOS runs scroll animations on the compositor thread (UIScrollView momentum). Any operation that forces the compositor to sync with the main thread pauses momentum scroll. This doc catalogs the specific APIs and patterns that cause sync, their root causes, and known workarounds. These constraints apply to all plugins that mutate the DOM during scroll — not just full screen bar hide/show.

## Main-thread JS kills momentum

ANY main-thread JavaScript execution during compositor scroll (UIScrollView momentum) causes a compositor-to-main-thread sync that pauses momentum. This is architectural, not a bug — native iOS apps use `UIScrollViewDelegate.scrollViewDidScroll` at the UI thread level, synchronized with the compositor. Web apps in WKWebView cannot access this.

**Affected APIs** (all cause sync): `classList`, inline styles, WAAPI `.animate()`, WAAPI `.play()`, `requestAnimationFrame` callbacks.

No JS-based workaround exists. All major PWAs and JS libraries (headroom.js, Twitter/X) accept this limitation. headroom.js iOS issue #100 documents it as unsolvable.

**Observed**: 2026-03-24, iOS/iPadOS.

## Instant layout exception

JS during momentum kills momentum ONLY if it triggers continuous relayout. Instant layout changes (`transition: none`) do NOT kill momentum — the compositor sync is brief enough that UIScrollView resumes.

**Root cause**: A single synchronous style recalc causes a brief compositor pause. UIScrollView resumes after the pause if no further recalcs follow. Continuous relayout (e.g., `transition: margin-top 0.3s` relayouting for 300ms after each class toggle) keeps the main thread busy long enough for UIScrollView to fully decelerate.

**Fix**: When mutating layout properties during scroll, ensure `transition: none` is active on the target element. Batch all changes into one synchronous tick.

**Observed**: 2026-03-24, iOS/iPadOS.

## `scrollTop` writes kill scroll unconditionally

`scrollTop` writes kill scroll in ALL states — during momentum, active touch, AND idle. Not just momentum. Touch-gated `scrollTop` writes (finger on screen, no momentum) still kill the scroll.

**Root cause**: `scrollTop` assignment forces an immediate compositor sync and scroll position override that cancels the active scroll gesture at the UIScrollView level.

**Fix**: True scroll-idle must be detected via scroll debounce (no scroll events for N ms). `scrollTop` writes are only safe when the scroll has fully stopped.

**Observed**: 2026-03-24, iOS/iPadOS.

## Layer promotion cost

The first `transform` write on an element that has never been transformed forces WebKit to create a new compositing layer (layer promotion). This one-time cost causes a compositor sync that kills momentum. Subsequent `transform` writes reuse the existing layer and are momentum-safe.

**Fix**: Pre-promote elements with `transform: translateY(0)` at initialization, before any scroll interaction begins.

**Caveat**: Do NOT use `will-change: transform` on scroll containers — it breaks scroll event detection on child scroll containers. The child container stops receiving scroll events entirely. Use `transform: translateY(0)` for pre-promotion instead.

**Observed**: 2026-03-24, iOS/iPadOS.

## Double-rAF pattern

WebKit's passive scroll listener optimization collapses inline `style.setProperty()` transition + target value into a single style recalculation when both are set in the same execution context. The transition never fires — the element jumps directly to the target value.

**Root cause**: WebKit batches style mutations from passive scroll listeners into one composite, so setting `transition` and `transform` in the same tick produces only the final computed value.

**Fix**: Frame 1 sets the transition property. Frame 2 (nested `requestAnimationFrame`) sets the target value. This forces two separate style recalcs, ensuring the transition fires.

```js
// Frame 1: set transition
el.style.setProperty('transition', 'transform 0.3s ease-out');
requestAnimationFrame(() => {
  // Frame 2: set target value — separate style recalc
  el.style.setProperty('transform', 'translateY(-91px)');
});
```

**Observed**: 2026-03-24, iOS/iPadOS.

## Scroll container resize during momentum — stale scrolling tree

WebKit runs momentum scroll on a dedicated scrolling thread (UIScrollView on iOS). The scrolling thread and main thread synchronize geometry via a commit handshake during display refresh. When the main thread changes scroll container geometry (height, margins, content size) during active deceleration, the scrolling thread continues operating on **stale bounds** until the next synchronization commit.

WebKit Bug 218676 (changeset r269558, Simon Fraser) fixed this for **programmatic scrolls** by immediately committing geometry via `requestScrollPositionUpdate()`. But passive momentum deceleration has no equivalent — no `requestScrollPositionUpdate()` is triggered, so the scrolling tree retains stale geometry.

**Impact**: Any plugin that resizes a scroll container during momentum (hiding bars, virtual scroll height changes, dynamic content insertion) may see intermittent visual jumps. The jump occurs in the window between the main thread's layout commit and the scrolling tree's geometry update.

**No CSS mitigation**: `contain: layout` limits layout recalculation scope but does not prevent scrolling-tree geometry propagation. `content-visibility: auto` skips rendering for off-screen content but does not affect scroll container bounds. `overflow-anchor: auto` would solve this at the compositor level but is not shipped in Safari through 26.5 (only in Technology Preview, with active bug fixes in STP 239).

**References**: WebKit Bug 218676, WebKit Bug 139245, WebKit scrolling thread architecture (`trac.webkit.org/wiki/Scrolling`).

**Observed**: 2026-04-02, iOS 26.4.

## `scrollend` fires at finger-lift

On WebKit, `scrollend` fires when the user lifts their finger — BEFORE momentum begins. Layout mutations at `scrollend` still kill momentum because the scroll is actively decelerating.

On Chromium, `scrollend` fires at true scroll-idle (~1ms after the last scroll event), making it safe for layout mutations.

**Root cause**: WebKit interprets `scrollend` as the end of the user's active scroll gesture, not the end of all scroll motion. This matches the "scroll ended from the user's perspective" mental model but diverges from Chromium's "scroll motion fully stopped" interpretation.

**Fix**: Use a scroll debounce (no scroll events for N ms) for true idle detection on WebKit. `scrollend` is not a reliable idle signal.

**Re-test candidate**: MDN spec says `scrollend` fires "when scrolling definitively completes" — after momentum, not at finger-lift. If WebKit aligned with the spec in 26.2+, this could replace scroll debounce for idle detection. The v87 empirical finding predates Safari 26.2. Worth re-testing.

**Timeline**: `scrollend` shipped in Safari 26.2 (December 2025). Safari 26.0-26.1 needs a debounced scroll fallback regardless.

**Observed**: 2026-03-24, iOS/iPadOS.

## Touch events suppressed during momentum scroll

iOS does NOT fire `touchstart` on the touch that stops momentum scroll. The user must lift and re-touch to receive touch events. This is a `UIScrollView.delaysContentTouches` behavior inherited by WKWebView — the scroll view claims the touch to stop scrolling and never forwards it to content.

On Chromium (Android), the same touch-to-stop-momentum gesture fires `touchstart` and `touchmove` normally, allowing immediate gesture evaluation.

**Implication**: JS-based gesture recognizers (multi-image swipe, drag handlers) cannot detect gestures on the touch that stops momentum. `touch-action` and `capture: true` do not help — the event is never dispatched.

Apple's Safari Web Content Guide: "One-finger panning doesn't generate any events until the user stops panning."

**Observed**: 2026-03-28, iOS/iPadOS. Confirmed iOS 11 through current.

## `touch-action: pan-y` lacks directional railing

On an element with `touch-action: pan-y`, Chromium evaluates the initial movement direction and "rails" the gesture — if horizontal, it suppresses vertical scrolling for the entire touch. WebKit does NOT rail. Any vertical deviation during a horizontal swipe causes `pointercancel` as WebKit claims the touch for vertical scrolling.

**Root cause**: WebKit does not implement directional locking based on initial gesture direction for `touch-action: pan-x`/`pan-y`. Chromium's behavior (giving up scrolling when the disallowed axis has greater displacement) is not specified by the W3C Pointer Events spec — it's an implementation choice.

**Impact**: Horizontal swipe gestures on `touch-action: pan-y` elements require a straighter finger path on iOS than on Android. In practice, this is tolerable for multi-image navigation but may cause false cancellations for less precise gestures.

**Workaround**: The `use-gesture` library's `preventScroll` option calls `preventDefault()` on early `touchmove` events to determine direction before letting the browser take over. Adds a small delay (~250ms) before the swipe begins.

**References**: W3C Pointer Events issue #303, WebKit bug #202053 (2019, still open). Confirmed broken iOS 15.4.1 per use-gesture #486.

**Observed**: 2026-03-28, iOS/iPadOS.

## Safari Timeline profiling on WKWebView

Safari Web Inspector's Timelines tab works on WKWebView over USB. Available instruments: Layout & Rendering, Memory, and Frames. No flame chart (open WebKit bug since 2012).

**What works**: `performance.mark()` and `performance.measure()` in WKWebView (iOS 11+). Timeline recordings export as JSON, parseable with Python for automated composite/frame analysis.

**Limitations**: `performance.now()` has 1ms resolution on iOS Safari — limits profiling precision for sub-millisecond operations. No programmatic CDP connection to iOS WKWebView — Safari Web Inspector console is manual only.

**Observed**: 2026-04-01, iOS/iPadOS.

## Non-passive `touchmove` blocks compositor scroll

A non-passive `touchmove` listener on an element with `touch-action: pan-y` blocks ALL compositor scroll on iOS WebKit — even when `preventDefault()` is never called. The browser waits for each handler invocation before advancing the scroll, effectively pausing it. Removing the non-passive listener restores normal scroll behavior.

On Chromium (Android), `touch-action: pan-y` allows the browser to scroll on the compositor without waiting for non-passive handlers.

**Impact**: Any element with a non-passive `touchmove` handler becomes a "scroll dead zone" on iOS — touches starting on it cannot initiate vertical scroll.

**Observed**: 2026-04-07, iOS 26.4.

## `overflow` changes ignored mid-gesture

Setting `overflowY: hidden` on a scroll container during an active touch gesture does NOT stop an ongoing scroll on iOS WebKit. The compositor has already claimed the gesture and does not re-evaluate `overflow` until the gesture ends.

On Chromium, `overflow` changes mid-gesture are respected and can stop an ongoing scroll.

**Workaround**: Use a direction lock pattern — decide the gesture direction (horizontal vs vertical) within the first few pixels of movement. Whichever axis crosses the threshold first wins the gesture for its duration. For horizontal custom gestures on `touch-action: pan-y` elements, accept minor vertical drift rather than attempting to suppress it mid-gesture.

**Observed**: 2026-04-07, iOS 26.4.

## `touch-action` evaluated at `touchstart` time

`touch-action` is evaluated at `touchstart`/`pointerdown` time, before JS event handlers run. Changing `touch-action` programmatically in a `pointerdown` handler has no effect on the current gesture — the browser has already committed to the scroll behavior.

**Implication**: `touch-action` cannot be used for conditional gesture handling (e.g., "allow scroll normally, but block it if a horizontal swipe is detected"). The decision must be made before the touch begins, not during the gesture.

**Observed**: 2026-04-07, iOS 26.4.
