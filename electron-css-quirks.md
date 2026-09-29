---
title: Electron CSS quirks
description: Blink/Electron CSS rendering quirks affecting selectors, text truncation, overflow clipping, container queries, GPU compositing, scroll anchoring, sub-pixel scroll offsets, and `filter` transitions from `none`.
author: 🤖 Generated with Claude Code
updated: 2026-09-29
---
# Electron CSS quirks

## Nested `:has()` inside `:has()` is broken

**Discovered**: 2026-02-25 on Electron 39.5.

`:has()` containing another `:has()` — even through `:not()` — silently fails. The inner `:has()` is treated as invalid, and forgiving selector parsing discards that argument.

### What works

| Pattern                     | Example                            | Status |
| --------------------------- | ---------------------------------- | ------ |
| Simple `:has()`             | `.card-body:has(> .card-previews)` | Works  |
| `:not(:has())` at top level | `.property:not(:has(.label))`      | Works  |
| `:has(:is())`               | `.card-body:has(:is(.a, .b))`      | Works  |

### What breaks

| Pattern                  | Example                                      | Status |
| ------------------------ | -------------------------------------------- | ------ |
| `:has()` inside `:has()` | `:has(> .foo:not(:has(> .bar)))`             | Broken |
| Nested via `:not()`      | `:has(> .el:not(:has(> .child:only-child)))` | Broken |

### Workaround

Replace nested `:has()` with adjacent sibling combinator `+` plus a simple `:has()` override.

**Before** (broken):

```scss
.card-body:has(
    > .card-previews:not(:has(> .card-thumbnail-placeholder:only-child))
  )
  > .card-properties-bottom {
  padding-top: var(--size-4-2);
}
```

**After** (working):

```scss
// Positive match via adjacent sibling — no :has() needed
.card-body > .card-previews + .card-properties-bottom {
  padding-top: var(--size-4-2);
}

// Override for the excluded case — simple :has(), not nested
.card-body
  > .card-previews:has(> .card-thumbnail-placeholder:only-child)
  + .card-properties-bottom {
  padding-top: 0;
}
```

## `-webkit-line-clamp` preserves trailing whitespace

`-webkit-line-clamp` preserves trailing whitespace at the truncation point before appending its ellipsis. When a word boundary falls at the truncation point (common, since `word-break: break-word` prefers word boundaries), the result is `word …` instead of `word…`.

The `-webkit-` prefix is historical (Blink forked from WebKit in 2013 and kept all prefixed properties). No CSS-only fix exists — `-webkit-line-clamp` offers zero control over its ellipsis behavior, and `text-overflow` only applies to single-line truncation.

### JS fix considered and rejected

A binary-search approach was prototyped and confirmed working:

1. Detect clamped overflow via `scrollHeight > clientHeight`
2. Binary search for max text prefix where `prefix + "…"` fits within clamped height
3. `trimEnd()` the prefix, set `textContent = trimmed + "…"`

**Rejected because the tradeoffs outweigh the cosmetic benefit:**

- ~10 forced layout reflows per card (setting `textContent` then reading `scrollHeight` in a loop)
- Reimplements browser truncation in JS — fights the platform instead of using it
- Mutates DOM outside the normal render pipeline
- Doesn't survive container resize without re-render

## Stuck `:hover` after drag

**Discovered**: 2026-03-03 on Electron 39.5.

After a drag operation ends, Chromium does **not** re-hit-test the element under the pointer. The dragged element retains `:hover` state until the next mouse move event. This causes visible artifacts when hover styles include transitions — e.g., a card background-color transition animates back over ~1s after drop.

This is a longstanding Chromium behavior, not Electron-specific.

### Mitigation

Gate visible hover effects behind a JS-managed class rather than pure `:hover`. Set the class on mousemove-after-mouseenter, remove on mouseleave/dragend. CSS hover styles use the class alone (class is the gate), so the stuck `:hover` has no visual effect because the class is removed in `dragend`.

```scss
// Wrong: visible artifact from stuck :hover
.card:hover {
  background: var(--hover-bg);
}

// Right: class is removed in dragend, so stuck :hover is invisible
.card.interact {
  background: var(--hover-bg);
}
```

## `overflow-clip-margin` ignored with per-axis `overflow-y: clip`

**Discovered**: 2026-03-05 on Electron 39.5.

`overflow-clip-margin` only takes effect when using the `overflow` shorthand (`overflow: clip`). When `overflow-y: clip` is set independently (e.g., `overflow-x: visible; overflow-y: clip`), the clip margin is silently ignored.

### Impact

Masonry's container used `overflow-x: visible; overflow-y: clip` so that cards could extend horizontally (e.g., sticky header borders with negative margins) while clipping vertical scroll overflow. Adding hover transforms (scale, translate) on edge-row cards caused them to be clipped at the container boundary because the clip margin wasn't applied.

### Fix

Switch to the shorthand `overflow: clip` + `overflow-clip-margin: <value>`. The clip margin then applies to both axes uniformly, accommodating both vertical hover transforms and horizontal negative-margin extensions.

```scss
// Broken: clip margin ignored
.masonry-container {
  overflow-x: visible;
  overflow-y: clip;
  overflow-clip-margin: var(--bases-view-padding); // silently ignored
}

// Working: shorthand enables clip margin
.masonry-container {
  overflow: clip;
  overflow-clip-margin: var(--bases-view-padding); // applied to both axes
}
```

## `@container scroll-state(stuck)` cannot style the container element

**Discovered**: 2026-03-08 on Electron 39.5.

CSS `@container scroll-state(stuck: top)` container queries can only style **descendants** of the container element, not the container itself. This is per spec — the container query matches on the container, but only descendant selectors inside the `@container` block are valid targets.

### Impact

Sticky group headings need elevated `z-index` when stuck (to paint above hovered cards with `z-index: 11`). A pure CSS approach using `container-type: scroll-state` on the heading and `@container scroll-state(stuck: top)` to set `z-index: 20` on child elements (`.bases-group-collapse-region`, `.bases-group-count`) was attempted. However, child `z-index` inside a flex container creates per-child stacking contexts — these cannot compete with cards in the parent grid stacking context. Only `z-index` on the heading element itself works, which `@container` cannot target.

### Fix

JS `IntersectionObserver` + zero-height sentinel approach. A sentinel div at each group section's top is observed — when it exits the scroll viewport upward, the heading is stuck. The observer toggles a `stuck` class on the heading, which carries `z-index: 20`. The `@container scroll-state(stuck: top)` rule is retained for the bottom border (progressive enhancement on descendant `::after`).

## `opacity` transitions trigger GPU compositing → grayscale antialiasing

**Discovered**: 2026-04-02.

When an element has an active `opacity` transition (even a brief 70ms one), the browser promotes it to a GPU compositing layer. Compositing layers use grayscale antialiasing instead of subpixel, making text appear blurred for the transition duration. This is most visible on the first paint of text-heavy elements.

The problem compounds when many invisible elements stack at the same position. Even with `opacity: 0` (no visible output), the browser still *paints* each element into its compositing layer. With 170+ masonry cards stacked at `(0,0)` before positioning, the compositor overhead from painting transparent layers affected text rendering of nearby visible elements.

### Fix

1. **Remove `opacity` from `transition` shorthand** — use CSS `@keyframes` animation for intentional fades instead. Animations override transitions in the cascade, so a `card-fade-in` animation plays correctly even without opacity in the transition list.
2. **Add `visibility: hidden` to invisible elements** — unlike `opacity: 0` (paints but transparent), `visibility: hidden` prevents painting entirely. Safe for measurement — `offsetHeight` works correctly, unlike `display: none`.

```scss
// Wrong: opacity transition triggers GPU compositing on every state change
.card.positioned {
  transition: opacity 70ms ease, top 200ms ease, left 200ms ease;
}

// Right: no opacity in transition, use animation for intentional fade
.card.positioned {
  transition: top 200ms ease, left 200ms ease;
}
.card.card-fade-in {
  animation: cardFadeIn 300ms ease-out;
}

// Wrong: invisible cards still painted into compositor layers
.card:not(.positioned) {
  opacity: 0;
}

// Right: prevents painting entirely
.card:not(.positioned) {
  opacity: 0;
  visibility: hidden;
}
```

## A `filter` transition from `none` runs `invert()` through gray

**Observed**: 2026-09-25, Electron 43.7.1; also desktop Safari, version not recorded

A `filter` transition from `none` pads `none` with the identity of each function in the other list, and the identity of `invert` is `invert(0)`. So `none` → `invert(1) brightness(0.5) invert(1)` animates both inverts through `invert(0.5)` — a gray flash — while an explicit `invert(1) brightness(1) invert(1)` rest state keeps them at 1 and animates only `brightness`.

- **Measured** halfway through a 400ms linear transition, in desktop Safari and in Chromium, in a hidden window opened from Obsidian.
- **`none` is safe without `invert`**: `brightness` and `blur` pad to their rest values, so a chain without `invert` can start from `none`.
- **Precondition for sampling a transition**: Safari throttles a background tab's animation clock, and its mid-transition value never advanced there. Sample in a hidden Electron window with `backgroundThrottling: false`, or in a foreground tab.

```scss
// Wrong: both inverts animate up from invert(0), through gray
.photo {
  filter: none;
  transition: filter 400ms;
}

// Right: the rest state lists the same functions, so only brightness moves
.photo {
  filter: invert(1) brightness(1) invert(1);
  transition: filter 400ms;
}

.photo.is-dimmed {
  filter: invert(1) brightness(0.5) invert(1);
}
```

## `-webkit-line-clamp` ignores block margins

**Discovered**: 2026-03-09 on Electron 39.5.

`-webkit-line-clamp` counts only text lines — block-level margins between elements (e.g., `<p>` with `margin-top`) do not consume any line budget. A container with `-webkit-line-clamp: 4` and six `<p>` children separated by `1lh` margins shows 4 lines of text plus all inter-paragraph margins, not 4 "visual lines" including margins.

No CSS-only mechanism exists to make `-webkit-line-clamp` count block margins as lines. The unprefixed `line-clamp: auto` with `max-height` is the future spec answer, but CSSWG reverted the dual-clamping resolution in January 2026 due to 4% page-load compatibility impact.

Related findings:

- **`<br>` elements behave identically** — they produce the same result as `\n` with `white-space: pre-line`. Neither consumes a clamp line.
- **`gap` is not an alternative** — `gap` does not work on `display: -webkit-box` (legacy flexbox). Only modern `flex`/`grid` support it.

### Workaround

JS per-paragraph clamping: measure each `<p>` height against a line budget, hide overflow paragraphs with `display: none`, and apply `-webkit-line-clamp` only to the last visible paragraph.

## `display: -webkit-box` computes as `flow-root`

**Discovered**: 2026-03-09 on Chrome 142 (Electron 36).

`display: -webkit-box` computes to `flow-root` in Chrome 142+. Despite the different computed value, `-webkit-line-clamp` still functions correctly — the truncation and ellipsis behavior is unchanged. This is a DevTools display quirk, not a functional regression.

## A flex item cannot host a `-webkit-box` line clamp

**Observed**: 2026-08-08, Electron 43.1.1 (Chromium 150).

Applying `display: -webkit-box` + `-webkit-box-orient: vertical` + `-webkit-line-clamp` to an element that is itself a **flex item** collapses it to **zero height**. The element disappears; the clamp never engages.

Blockification is the cause — a flex item's `display` is blockified, and the legacy box cannot act as a clamp container once that happens.

- **Clamp a child instead.** Move the declarations to a descendant whose parent is an ordinary block. The child is not a flex item, so the box survives and clamps normally.
- **Forcing the child's `display` does not rescue it.** Setting the flex item's own child to `display: block` leaves the collapse in place; it is the clamp element's status as a flex item that matters, not what it contains.
- **Nested flex descendants keep their own layout.** A flex row inside the clamped box still lays out horizontally — only the box's own line boxes are counted.
- **A flex descendant escapes clamping entirely.** A `display: flex` child counts as a single box rather than line boxes, so the clamp has nothing to count and the element renders at full height. Give it `display: inline` to make its content participate. `inline-flex` does NOT work — it is still one box.
- **Measure rendered height, never the computed value.** `display` reports `flow-root` either way (see above), so it cannot distinguish a working clamp from a collapsed one.

Watch for a second failure mode nearby: if the clamped element is a flex item whose container is height-constrained, `flex-shrink` compresses it below the clamp — a two-line clamp renders one line. `flex-shrink: 0` fixes it.

## `hyphens: auto` support varies by platform, and skips capitalized words

**Observed**: 2026-08-09, Electron 43.1.1 (Chromium 150).

Automatic hyphenation is not uniformly available, and where it is available it deliberately skips some words.

| Platform | Mechanism | Works? |
|---|---|---|
| macOS | CoreText, no dictionary needed | Yes |
| iOS | Same CoreText path via WebKit | Yes |
| Android WebView | AOSP dictionaries at a hardcoded system path | Yes |
| Windows, Linux | Dictionaries delivered by Chromium's component updater | **No** |

- **Electron has no component updater**, and does not override the browser-client hook that supplies the dictionary directory, so Windows and Linux silently never hyphenate. The property parses and computes to `auto` regardless.
- **Bundling the dictionary files does not help.** The directory fallback that would load them is compiled only into Chrome-for-Testing builds. No command-line switch or feature flag exists. The only route is a patched Electron build.
- **Blink refuses to hyphenate capitalized words in `en*` locales.** This is a deliberate UA heuristic, and it makes hyphenation appear broken on Title Case headings while sentence case works. WebKit has no such rule, so the same capitalized word hyphenates on iOS but not on macOS.
- **Probe with a lowercase, real, multi-syllable word.** A capitalized word or a synthetic string such as `aaaaaaaa` has no break points and yields a false negative. Blink also requires a minimum word length of 5 with 2-character prefix and suffix.
- **`hyphens: manual` with soft hyphens works everywhere**, since it needs no dictionary. It is the only portable route, at the cost of soft-hyphen characters surviving into copied text.

## `-webkit-line-clamp` ellipsis eats characters on forced line breaks

**Observed**: 2026-04-10, Electron 39.8.3 (Chromium 142).

When `-webkit-line-clamp` truncates text that contains forced line breaks (`\n` with `white-space: pre-line`, or `<br>`), the ellipsis replaces trailing characters on the last visible line instead of appending after the text. This only occurs when the text on the last visible line is shorter than the container width — the ellipsis glyph-removal algorithm operates on the physical line box end regardless of actual text length.

**Example**: With `-webkit-line-clamp: 2` and text `"Title line one\nTitle line two\nTitle line three"`, the second line renders as `"Title line tw…"` instead of `"Title line two…"` despite ample horizontal space remaining.

### Platform scope

| Platform | Behavior |
|---|---|
| **Desktop Electron** (Blink) | Characters eaten — ellipsis replaces trailing glyphs |
| **iOS/iPadOS** (WebKit) | Correct — ellipsis appends after text |
| **Android** (Chrome/WebView) | Correct — ellipsis appends after text |

### Root cause

Blink's legacy `-webkit-line-clamp` implementation reuses the `text-overflow: ellipsis` glyph-removal algorithm. That algorithm measures from the physical line box end inward to make room for the `…` glyph, removing characters as needed. When a forced `\n` break ends the line early, the algorithm still removes characters from the text end rather than recognizing that space already exists.

### Attempted workarounds (all failed)

- **`word-break: normal` + `overflow-wrap: normal`**: No effect — the algorithm operates on the line box end regardless of break rules.
- **`<br>` instead of `\n`**: Same result — both are forced breaks that trigger the same code path.
- **No CSS-only workaround exists**.

### Resolution timeline

The unprefixed `line-clamp` spec uses block-ellipsis placement which does not reuse the glyph-removal algorithm. Chromium tracks this behind the `css-line-clamp-line-breaking-ellipsis` flag. When shipped, the bug auto-resolves for code using `-webkit-line-clamp`.

- **Tracking**: https://issues.chromium.org/issues/40336192
- **CSSWG spec**: https://drafts.csswg.org/css-overflow-4/#propdef-line-clamp

## `-webkit-line-clamp` loses its ellipsis under right-aligned text

**Observed**: 2026-09-25, Electron 43.7.1.

When a `-webkit-line-clamp` truncation leaves room on the last visible line, Chromium places the ellipsis after that line's content without re-aligning the line. Under `text-align: right` the content already ends at the right edge, so the ellipsis lands past it, where the element's clip hides it: the text truncates with no visible `…`. A last line that fills the full width truncates normally and shows it.

- **Measured** with plain text and with inline list items.
- **Unmeasured**: centered text.

## `z-index: 0` on `.cm-line` breaks CM6 click-to-position

**Observed**: 2026-04-14, Electron 39.8.3.

Setting `z-index: 0` on a `.cm-line` element (to create a stacking context for child pseudo-elements) causes `caretRangeFromPoint()` to return incorrect positions when clicking past the end of text on an active (`.cm-active`) heading line. `posAtCoords` returns `lineFrom` (line start) instead of `lineTo` (line end), causing the cursor to jump to position 0.

- **Clicks on text** are unaffected — correct character positions are returned.
- **Clicks past text edge** (right of the last character) return the wrong position.
- **The bug only manifests when the line is `.cm-active`** — first click works, second click (on the now-active line) fails.
- **Interaction with `::before`/`::after`**: The bug is exacerbated when themes add absolutely-positioned pseudo-elements (e.g., active-line highlighting via `::before`). The combination of `z-index: 0` stacking context + pseudo-elements confuses Blink's hit-testing for `caretRangeFromPoint()`.

### Fix

Remove `z-index` from `.cm-line` elements entirely. If the pseudo-element is positioned in the padding area below text (no vertical overlap with text content), `z-index` layering is unnecessary — the pseudo-element and text occupy different vertical spaces regardless of stacking order.

If vertical overlap IS needed (e.g., an underline overlapping text), there is no CSS-only fix. The stacking context breaks click-to-position. Alternatives:
- **`pointer-events: none`** on the pseudo-element helps but does not fix the `caretRangeFromPoint` issue.
- **JS-injected child elements** instead of pseudo-elements (avoids the stacking context requirement).

## Native scroll anchoring double-counts a model-based `scrollTop` correction

**Observed**: 2026-09-27, Electron 43.7.1.

Chromium's scroll anchoring (`overflow-anchor: auto`, the default) moves `scrollTop` during layout when content above the visible area changes size. A script that then corrects `scrollTop` from its own layout model — "the rows above grew by N, so scroll down by N" — applies the same shift a second time.

- **Measured**: 40px of growth above the visible area got +39 from native anchoring, then +36 from a model correction, so the content jumped 36px. In another run, native anchoring had already moved `scrollTop` before 46 of 78 model corrections, and the content jumped up 7px.
- **Measure instead of predicting**: Record an on-screen element's `getBoundingClientRect().top` before the layout change and correct by how far it actually moved. The measured delta already includes whatever native anchoring did, so it never counts the shift twice.
- **Anchor on an element that kept its size**: Holding it still holds everything below it, and a resized element above it grows upward instead of pushing the view down. When every visible element resized, the first one's top still moved only by the content above it, so its delta is still a valid correction.
- **WebKit has no native scroll anchoring** through Safari 27.2. Feature-test with `CSS.supports('overflow-anchor', 'auto')`, which is false there.

## A `scrollTop` write cancels a running smooth scroll

**Observed**: 2026-09-27, Electron 43.7.1.

Assigning `scrollTop` while a smooth scroll is animating stops the animation where it is, and the scroller never reaches the target.

- **Measured**: A smooth `scrollTo` from 6000 to 1000 stopped at 6003 after one +5 write. The control, with no write, landed at 1000.
- **Keyboard scrolling animates**: Home, End, Page Up/Down and Space scroll smoothly, as does `scrollIntoView({ behavior: 'smooth' })`. Home, End and Page Up/Down took 140–200ms (16–18 scroll events at 120Hz).
- **Workaround**: Hold corrective writes from the keydown until the scroller's `scrollend`, with a timeout as a backstop, since a key that scrolls nothing fires no `scrollend`. Skip keydowns aimed at an editor or a text field: they move the caret or the editor's own scroller, not the one being corrected.

## `scrollTop` lands on device pixels, so a fractional layout shift re-snaps text

**Observed**: 2026-09-27, Electron 43.7.1.

A box that settles at a fractional height (e.g., 316.24px) shifts everything below it by a fractional amount, but a compensating `scrollTop` write lands on the device-pixel grid: a target of 74348.64 landed at 74348.5. The residue moves the content below, and its text re-snaps by a device pixel — a visible jiggle.

- **Keep the layout on whole pixels**: Round estimated sizes before they become heights or offsets, and round content-sized boxes with `calc-size()` (see below).
- **Measured cost of fractional estimates**: Estimates that fed absolutely positioned tops produced −1.89px height corrections and tops at .83–.99, and every one of 8 correction writes landed 0.1–0.23px off target.

## `calc-size()` rounds a content-sized box up to whole pixels

**Observed**: 2026-09-27, Electron 43.7.1.

`min-height: calc-size(max-content, round(up, size, 1px))` rounds a box's content height up to the next whole pixel without disturbing its own `height`, so it composes with `height: 100%` on a grid item stretched to its row.

- **Measured** on stretched grid items with `height: 100%`: Heights went 286.28 → 287 and 371.43 → 372, rows stayed equal-height, and each item's last child kept its offset.
- **`min-height: calc-size(auto, …)` has no effect**, and `height: calc-size(auto, …)` would replace `height: 100%`, dropping the row fill.
- **Exclude boxes sized some other way**: An `aspect-ratio` box must keep its ratio over its content height. A `min-height` also beats an inline `height` lock, so a lock shorter than the content would stop clamping.
- **Round an `aspect-ratio` box with `height` instead**: On a box sized by `aspect-ratio` under `height: auto`, `height: calc-size(auto, round(nearest, size, 1px))` rounds the ratio-derived height (254.797 → 255, observed 2026-09-29). Declare it after a plain `height: auto`, which WebKit keeps. An absolutely positioned box with an inline width rounds the same way. With `object-fit: contain` and an image that exactly fills the box, the edge pixels matched with and without the rounding.
- **Cost**: Style recalc rose from 35–37ms to 44–45ms per 15,000px/s fling across a large grid, measured over three warm pairs with the stylesheet settled before each trace. Layout was unchanged.
- **WebKit has no `calc-size()`** through Safari 27.2 and drops the declaration, so the rounding applies in Chromium only.
- **Dart Sass passes it through** unchanged: `round(up, size, 1px)` inside `calc-size()` compiles as written.
