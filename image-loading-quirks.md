---
title: Image loading quirks
description: What an img reports and paints while its picture is still downloading, and when changing, removing or sharing its source aborts, joins or restarts the download — measured in Chromium (Electron) and WebKit (Safari).
author: 🤖 Generated with Claude Code
updated: 2026-09-27
---
# Image loading quirks

How `img` elements report and paint a picture that is still downloading, and what happens to a download when an element's source changes, is removed or is shared with another element. Chromium facts were measured in Obsidian's Electron and WebKit facts in macOS Safari; none has been re-measured in an iOS or iPadOS WKWebView. For how downloads behave across Obsidian's windows, see `electron-popout-quirks.md`.

**Test setup behind these measurements**: a local server streams a large baseline (non-progressive) JPEG in small chunks, with an optional stall, and logs how each response ended — completed, or aborted after how many bytes. A log that records only request starts cannot show an abort. Every trial uses a fresh URL (a unique query string): within one document, an in-flight or memory-cached request for the same URL is shared, so a repeated trial reads the earlier download.

## Readiness signals while a picture streams

**Observed**: 2026-09-24 to 2026-09-25, Electron 43 (Chromium 150) and macOS Safari 27

| Case | Reads |
|---|---|
| Body arriving | `naturalWidth` is non-zero from the first body bytes, long before `load` — Chromium at 9ms against `load` at 14.4s, Safari at ~520ms against `load` at 70s, for a 2.3MB file. A fresh element has painted its top rows by then |
| Headers sent, body stalled | `naturalWidth` 0 and a blank box until the first chunk arrives |
| SVG with no intrinsic size | Loads as 300×150 |
| Broken image | `naturalWidth` 0 with `complete` true |
| After `img.src = ''` | `complete` true (Chromium) |

- **`naturalWidth > 0` means the picture has started arriving, not that it has loaded**: use it to reveal a picture early; use `load`, or `complete && naturalWidth > 0`, for a finished one.
- **`complete` alone is not success**: a broken image and an emptied element both read `complete` true.
- **Chromium does not give up on a stalled body**: with the headers sent and the body stalled for 200s, an element read `naturalWidth` 0 and `complete` false throughout, painted at 210s and fired `load` at 286s — no `error` at any point. Anything that waits on `load`/`error` needs its own timeout.

Probe preconditions: keep Safari's window frontmost — occluded, it screenshots blank white and its timings drift. Set `src` and start sampling in the same script run; a probe split across separate evaluations can miss a short stall window entirely.

## A new `src` shows only once complete, unless the element was emptied

**Observed**: 2026-09-26, Electron 43 (Chromium 150) and macOS Safari 27

- **The old picture stays until the new one completes**: an `img` already showing a picture keeps painting it after its `src` changes, until the new picture has fully loaded. It never streams the new one.
- **The readiness signals move on at once**: `naturalWidth` and `currentSrc` report the new request from its first bytes, so neither can tell that the old picture is still on screen.
- **Why**: both engines' image loaders hand the renderer a new image only when the renderer holds none, or the new image is complete — to avoid flicker when one image replaces another (Blink `ImageLoader::UpdateLayoutObject`, WebKit `ImageLoader::updateRenderer`).
- **Empty the element to stream the replacement**: call `removeAttribute('src')` right before assigning the new `src`. The emptied element then paints the new picture's rows as they arrive. The removal fires no `error` in either engine, unlike `src = ''`. A memory-cached URL still reads `complete` true synchronously after the pair.

```ts
img.removeAttribute('src'); // no `error` event, unlike `img.src = ''`
img.src = nextUrl; // paints nextUrl's rows as they arrive
```

Probe precondition: record whether the element held a picture before the new `src`. A streaming measurement on a fresh or emptied element says nothing about one that was already showing a picture. Safari's screenshots of an emptied element can lag the download, showing fewer rows than have arrived.

## Changing `src` mid-download

**Observed**: 2026-09-26, Electron 43 (Chromium 150) and macOS Safari 27

- **Chromium aborts the abandoned download**: once `src` moves to another URL, the old download stops at once — for an attached `img` and for a detached `Image` alike. It continues only while another attached `img` in the document still uses that URL, and a later return to the URL starts again from zero.
- **WebKit never aborts it**: in all three cases above, the old download ran to the end of the file. Every URL an element is pointed at and then moved off costs its whole download.

## One download shared by a detached `Image` and a visible `img`

**Observed**: 2026-09-26, Electron 43 (Chromium 150) and macOS Safari 27

- **One request**: a detached `new Image()` given a URL, then an attached `img` given the same URL 60ms later — the server saw one request, and the `img` streamed it, with a non-zero `naturalWidth` at 1.5s while the file was still arriving.
- **The `Image` holds the download on its own**: after the `img` moved on to another URL, the download still ran to the end, held by the `Image`.
- **So a prefetch through `new Image()` is never wasted**: a visible element that asks for the same URL streams the prefetch's download instead of starting its own, and the download is not aborted when that element moves on.

Preconditions: the identical URL string, in the same document (see `electron-popout-quirks.md` for windows), with the `Image` keeping its `src` until the download settles.

## Removing `src` and `srcset` mid-download

**Observed**: 2026-09-26, Electron 43 (Chromium 150) and macOS Safari 27

| After removing `src` and `srcset` mid-download | Chromium | WebKit |
|---|---|---|
| The download | Aborted at once | Never aborted |
| A detached `img` that keeps its `src` | Downloads on | Downloads on |
| A same-URL `img` created in the same task as the removal | Joins the download in flight, for `no-store` and cacheable responses alike | Joins it if the response is cacheable (`max-age=3600`); a second full download if it is `no-store` |
| A same-URL `img` created 100ms later | A second request, from zero | Same as the row above |

- **Releasing a download a successor will want**: stripping `src` the moment an element is discarded restarts the picture from zero in Chromium when its replacement asks for the same URL in a later task, and downloads an uncacheable picture twice in WebKit. Delay the strip until any successor has made its own request; strip at once only when nothing will ask for that URL again.

Probe precondition: Chromium opens at most six connections per host, so with six slow downloads open, a seventh request to the same probe server waits and is never sent within the trial.

## A failed CSS `background-image` refetches only after a different value

**Observed**: 2026-09-26, Electron 43 (Chromium 150)

- **Same value, no retry**: setting a `background-image` that failed to load to the same value again fetches nothing.
- **Through `none` or another URL, a retry**: setting `none`, or another URL, and then the original value fetches it again.

Measured in a visible window on a 2px element, with the server down for the first load and back up for the retry, the intermediate value held for two animation frames before the original was restored, and a fresh URL per trial. See `obsidian-ui-internals.md` for how Bases' Cards view depends on this.
