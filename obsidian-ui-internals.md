---
title: Obsidian UI internals
description: How native Obsidian UI works under the hood — the image lightbox's drag panning, double-click zoom, loading and focus, the test that classes mouse events as touch-made, mobile press feedback, safe-area insets under mobile emulation, and Cards view's failed-cover retry.
author: 🤖 Generated with Claude Code
updated: 2026-09-27
---
# Obsidian UI internals

How some of Obsidian's own UI works internally, read from `app.js` and `app.css` and measured at runtime. Useful when a plugin replicates one of these surfaces and wants native parity, or needs to predict how native UI reacts next to its own. None of this is public API, and all of it can change without notice. For the history stack a back press steps through, see `undocumented-obsidian-apis.md`.

## Image lightbox

Obsidian's own image viewer, opened by clicking an image in Reading view.

### Mouse drag panning

**Observed**: 2026-09-24, Obsidian 1.14.2 (installer 1.14.2), read from `app.js`

- **Zoomed only, from the image only**: a primary-button press on the displayed image starts a pan only while the zoom level is above 1. It first stops any glide still running.
- **A shared drag helper with a 5px threshold**: Obsidian's generic pointer-drag helper ignores a non-primary pointer, listens on the event's window for `pointermove`, `pointerup`, `pointercancel`, `dragstart`, `drop` and `contextmenu` (and for `keydown`, where Escape cancels, and `keyup`), and starts the drag only once the pointer has moved 5px from where it went down. It never captures the pointer.
- **The first move carries the whole threshold**: at start, the lightbox adds `is-grabbing` to its document's `body` and seeds its last pointer position and move time from the `pointerdown` event. The first applied move therefore covers the full distance traveled so far, and its velocity sample spans the whole wait before the threshold.
- **Release glides and swallows one click**: on release it starts its momentum glide and adds a capture-phase `click` listener on its media container that cancels the next `click` and removes itself one task later. A click on the container anywhere but an image closes the lightbox, and with no pointer capture, the `click` after a drag lands on whatever is under the pointer.

Pointer capture makes the swallow unnecessary for that case: in Chromium, the `click` that follows a captured drag fires on the capture target, not on the element under the pointer at release (measured, Electron 43, trusted CDP input, a drag from the image released over the backdrop).

### Double-click zoom

**Observed**: 2026-09-24, Obsidian 1.14.2 (installer 1.14.2), read from `app.js`

- **Touch-made double-clicks are rejected**: a `dblclick` on the displayed image zooms, unless the lightbox opened under 300ms ago or the touch-synthesis test (see "Touch-synthesized mouse events") classes the event as touch.

### Loading: the lightbox never streams

**Observed**: 2026-09-25, Obsidian 1.14.2 (installer 1.14.2), Electron 43

- **Every image starts loading at open**: opening the lightbox builds one `img` per image in the note, each with `decoding="async"` and its `src` already set.
- **The opening image waits for `load`**: when the lightbox opens from an image on the page, it holds that image, its backdrop and its title bar at opacity 0, and starts its open animation only once the image is complete. A picture still downloading therefore shows nothing there, while the same picture in the note behind paints its rows as they arrive — notes stream, and so do Cards view cards (observed by eye), but the lightbox does not.
- **A step to a downloading image paints nothing either**: an arrow step to a still-streaming picture updated the title and showed an `img` reading `naturalWidth` 2400, `complete` false, opacity 1 and visible, yet nothing painted, while the dimmed note behind showed that picture's rows.

Precondition: the step case was measured only in a desktop popout, where the lightbox builds its `img` elements with the main window's `createEl`, `src` included, so their downloads start in the main window's document (see `electron-popout-quirks.md`). The main window was not measured.

### Keyboard and focus: no trap

**Observed**: 2026-09-26, Obsidian 1.14.2 (installer 1.14.2), Electron 43 — desktop popout, Reading view, trusted CDP key input

- **Tab leaves the lightbox**: it focuses its own container (`tabindex="-1"`) on open. Tab moves focus to the focusable controls behind it — the view header's buttons in the measured case — and further Tabs cycle through those, never returning to the lightbox, which stays open.
- **Rings show under the dim**: nothing traps focus, so the control focused behind the lightbox draws its focus ring under the dimmed backdrop.
- **Its keys work only while focus is inside it**: Escape, `=`/`-`, the arrow keys and Space are handled by a `keydown` listener on the lightbox container. Space closes it on `keyup`, not `keydown`.

## Touch-synthesized mouse events

**Observed**: 2026-09-24, Obsidian 1.14.2 (installer 1.14.2), read from `app.js`; the `dblclick` shape measured in Electron 43 (Chromium 150) and in Obsidian on iPadOS (version not recorded)

Obsidian decides whether a mouse event was made by touch with one module-private test. The lightbox's double-click zoom, `aria-label` tooltips and the `mobile-tap` hover marking all skip events it classes as touch.

- **A `PointerEvent`** counts as touch when its `pointerType` is not `mouse`, or when the main document has seen fewer than 2 mouse `pointermove` events since the last `touchstart` (or since startup).
- **Any other `MouseEvent`** counts as touch while a touch is down, for 400ms after the last touch ends, and in the iOS/iPadOS app (`Platform.isIosApp`) for 100ms after any scroll.
- **Its inputs live on the main window**: capture-phase `touchstart`, `touchend`, `touchcancel` and `scroll` listeners on the main window, plus a capture-phase `pointermove` counter on the main document.
- **Apple Pencil counts as touch**: WebKit fires `touchstart`/`touchend` for Pencil contact too, with `touchType: 'stylus'`, so a Pencil tap updates the same state a finger does.
- **`dblclick` only ever reaches the `MouseEvent` branch**: in Chromium, and in WebKit for a trackpad double-click on iPadOS (a Mac trackpad through Universal Control), `dblclick` is a plain `MouseEvent` with no `pointerType`. Only the `MouseEvent` conditions above — a touch still down, one that ended under 400ms ago, or a recent scroll in the iOS/iPadOS app — can reject it.
- **A double-tap on a `touch-action: none` surface fires no `dblclick` at all**: neither a finger nor a Pencil double-tap produced one on iPadOS, measured on a full-screen `touch-action: none` overlay.

## Mobile press feedback (`mobile-tap`)

**Observed**: 2026-09-24, Obsidian 1.14.2 (installer 1.14.2), read from `app.js` and `app.css`

On mobile, and under desktop mobile emulation, Obsidian marks a pressed control with the `mobile-tap` class, and `app.css` styles that class control by control.

- **Target**: a capture-phase `touchstart` on the main document, for a single finger, adds `mobile-tap` to the touched element or its nearest ancestor matching `a, button, .tappable, .is-clickable, .clickable-icon, .text-icon-button, .suggestion-item` — or, with no match, to the touched element itself.
- **Removal**: at once when the finger moves more than 5px or the touch is canceled; otherwise after the lift, once the press has lasted 300ms in all — `max(10, 300 − press duration)` ms after `touchend`.
- **Mouse hover marks it too**: a `pointerover` on those selectors adds the class, and `pointerout` removes it, when the touch-synthesis test classes the pointer as a real mouse — so a trackpad or mouse hover on a tablet dims a control the same way a press does. A Pencil hover arrives as `pointerType: 'pen'`, which the test classes as touch, so it marks nothing.
- **The icon dim**: `.clickable-icon.mobile-tap svg` takes `--icon-opacity-hover`, which `.is-mobile` sets to `0.65` against `--icon-opacity: 1`.
- **`.tappable` carries no styles of its own**: it only opts an element into `mobile-tap`. Under `body.emulate-mobile`, it and the other tappable selectors get `touch-action: manipulation`.

A plugin control gets native press feedback by matching one of those selectors — `.clickable-icon` on an icon button brings the icon dim with it. See `ios-webkit-quirks.md` for a WebKit layer shift that the opacity change on `a.mobile-tap` triggers.

## Safe-area insets and mobile emulation

**Observed**: 2026-09-24, Obsidian 1.14.2 (installer 1.14.2), read from `app.js` and `app.css`; the popout inset measured in Electron 43

- **Variables over `env()`**: `app.css` declares `:root { --safe-area-inset-top|bottom|left|right: env(safe-area-inset-*) }`, and native styles read the variables.
- **Mobile emulation fakes a notched phone**: `body.emulate-mobile` overrides them to 59px top, 34px bottom and 0 on both sides, so desktop emulation exercises safe-area layout without a device. `app.emulateMobile(true)` sets a local storage flag and reloads the window; `app.emulateMobile(false)` clears it and reloads.
- **Emulation on macOS reports iOS**: under emulation, `Platform.isMobile` is true, `Platform.isDesktop` and `Platform.hasPhysicalKeyboard` are false, and `Platform.isIosApp` takes the value of `Platform.isMacOS` — so on a Mac, code gated on `isIosApp` runs.
- **Nothing zeroes them in popouts**: the only place `app.js` writes them is the body of a Markdown editor it hosts in an `iframe`, where it sets all four to `0` inline. A popout body carrying `emulate-mobile` computes `--safe-area-inset-top` as 59px.

See `android-chromium-quirks.md` for Android, where `env()` resolves to `0px`.

## Cards view: a failed cover retries only through item recycling

**Observed**: 2026-09-26, Obsidian 1.14.2 (installer 1.14.2), read from `app.js`; the CSS behavior measured in Electron 43

Bases' Cards view has no retry for a cover that failed to load. It retries only as a side effect of how it virtualizes cards:

- **Visible cards keep their item**: when the visible range updates, an entry that stays visible keeps the DOM item it already had.
- **Entering cards take a pooled item**: an entry scrolling into view is drawn into an item from the unused pool — oldest released first — or into a new one. The pool keeps at most max(10, 2 × cards per row) items.
- **The cover is written only when it changes**: an item sets its cover's `background-image` only when the value differs from the one it last showed.
- **Chromium does not refetch on a same-value write**: re-setting a failed `background-image` to the same value fetches nothing, while going through `none` or another URL first fetches it again (see `image-loading-quirks.md`).

So a card that scrolls out and back, and lands in an item that last showed another card, retries its picture. A card that stays on screen, or lands back in its own item, keeps the failure.
