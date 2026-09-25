---
title: Doc index
description: Index of all cross-plugin knowledge files — when and why to read each one.
author: 🤖 Generated with Claude Code
updated: 2026-09-25
---
# Doc index

> [!important]
> While an effort is made to keep these docs continuously up to date, all content is generated with Claude Code and may contain inaccuracies — verify important information.

| Doc | Read before |
|---|---|
| `undocumented-obsidian-apis.md` | Using or looking up undocumented Obsidian properties and methods — documents useful internal APIs discovered through runtime inspection. |
| `ios-webkit-quirks.md` | Modifying content-visibility, IntersectionObserver, scroll-state container queries, touch handling on `position: fixed` elements, suppressing long-press image drag, or click synthesis behavior — iOS WebKit has platform-specific bugs including compositor hit-test divergence. |
| `electron-popout-quirks.md` | Using `document`, `window`, `ResizeObserver`, `IntersectionObserver`, `requestAnimationFrame`, `floatingSplit`, or `:hover` checks in code that may run in popout windows — covers silent observer failures, RAF paint timing, stale hit-testing, document enumeration, and module-scope binding issues. |
| `obsidian-api-quirks.md` | Using `vault.process()`, `vault.modify()`, `new Notice()`, Bases `config.get()`/`config.set()` with dynamic schema defaults, how an open `.base` file saves and reloads across its three open forms, or any file I/O that could race with Obsidian's debounced writes. |
| `declarative-settings-api.md` | Building a settings tab with `getSettingDefinitions()` (Obsidian 1.13+) — covers when definitions are re-evaluated, `refreshDomState()` vs `update()`, the focused row `update()` skips, definitions that silently render nothing, sub-settings that split a section into two boxes, cascading side effects without an `onChange`, row reuse that duplicates hand-mounted DOM, what breaks search indexing, and the `renderTab` name collision that blanks the settings pane. |
| `android-chromium-quirks.md` | Working with CSS environment variables, DOM timing, scroll behavior, or animations that run in Android's Chromium WebView under Capacitor. |
| `webkit-compositor-constraints.md` | Doing scroll-concurrent DOM work or touch gesture handling on iOS/iPadOS — covers momentum-killing APIs, `scrollTop` write behavior, the double-rAF pattern, layer promotion, and touch-action evaluation timing. |
| `obsidian-review-bot.md` | Fixing bot-reported issues, adding/modifying eslint-disable comments, or preparing a PR for the Obsidian plugin review. |
| `electron-css-quirks.md` | Writing nested `:has()` selectors, clamping text with `-webkit-line-clamp` (especially on or inside a flex item), relying on `hyphens: auto`, using `opacity` transitions on text-heavy elements, or setting `z-index` on `.cm-line` elements — documents Blink/Electron CSS rendering quirks including GPU compositing antialiasing, per-platform hyphenation support, and CM6 click-to-position breakage. |
| `obsidian-css-specificity.md` | Overriding CSS on an Obsidian-owned element, or removing `!important` from plugin styles — covers `app.css` state-class selectors that specificity cannot beat, how to identify the winning rule at runtime, and when `!important` is the correct answer. |
