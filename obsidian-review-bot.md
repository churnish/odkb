---
title: Obsidian review bot
description: Obsidian review bot behaviors, workarounds, and common issue solutions.
author: 🤖 Generated with Claude Code
updated: 2026-10-18
---
# Obsidian review bot

The Obsidian review bot runs `eslint-plugin-obsidianmd` with its `recommended` config against PRs. It scans independently of any local ESLint setup.

## Key behaviors

- **Ignores eslint-disable comments**: The bot strips all `eslint-disable` directives before scanning. You cannot suppress bot errors with inline comments.
- **Reports "required" and "optional" issues**: Required issues block approval. Optional issues are advisory.
- **Re-scans automatically**: After pushing changes, the bot rescans within 6 hours. Do NOT open a new PR for re-validation.
- **`/skip` for false positives**: Comment `/skip` with a reason on the PR if a required issue is incorrect.

## Local enforcement

`@eslint-community/eslint-plugin-eslint-comments` with `no-restricted-disable` is configured in all plugins to block `eslint-disable` for `obsidianmd/*` rules locally. This catches futile suppression attempts at lint time — before the bot even sees the code.

## Common issues and solutions

### `no-explicit-any` — disabling not allowed

The bot forbids `eslint-disable` for `@typescript-eslint/no-explicit-any`. Fix by adding proper type augmentations instead.

### Sentence case for UI text (`obsidianmd/ui/sentence-case`)

The bot requires sentence case for string literals passed to UI methods (`.setName()`, `.setDesc()`, `.setPlaceholder()`). To bypass for intentionally lowercase text (e.g., property name placeholders), extract the string to a `const` — the bot only checks string literals, not variable references.

### Lookbehind regex (`obsidianmd/regex-lookbehind`)

Lookbehinds (`(?<=...)`) are not supported on iOS < 16.4. Replace with capture groups. Example: `(?<=\s|^)#tag` becomes `(^|\s)#tag` with the capture group returned in the replacement.

### Undescribed directive comments

All `eslint-disable` and `eslint-enable` comments must include a description after `--`. Example: `// eslint-disable-next-line rule-name -- reason why`.

### Static style assignment (`obsidianmd/no-static-styles-assignment`)

The rule catches direct property assignments like `element.style.left = 'auto'` but does NOT catch computed property access like `element.style[prop] = value`. To avoid lint errors for static assignments, move them to CSS classes instead of setting inline.

### `prefer-create-el` — the suggested fix does not compile

The rule's message tells you to replace `doc.createElement('div')` with `doc.win.createDiv()`. That does **not** typecheck. `createEl`/`createDiv`/`createSpan` are declared on Obsidian's `interface Node` (`obsidian.d.ts`), not on `Window`.

Call them on the parent element instead — but note they **append** to that element, so they only substitute for `createElement` when the append is unconditional. Detached creation (`insertBefore`, `after`, deferred parenting, conditional append) must stay `ownerDocument.createElement`.

### `prefer-active-doc` is disabled upstream

As of `eslint-plugin-obsidianmd` 0.4.1 the `prefer-active-doc` rule ships as `off` in the recommended config. Bare `document` references are not flagged. This is an upstream default, not a local override — do not add a local disable for it.

### `eslint --fix` coverage and per-rule scoping

`--fix` autofixes a large share of `obsidianmd/*` warnings (112 of 142 in one dynamic-views pass). It is not safe to run repo-wide unscoped:

- Fixes for multiple rules land at once, so unrelated files in a dirty tree get rewritten.
- `no-global-this` autofixes are crude — it strips `as typeof globalThis` casts and rewrites `Window & typeof globalThis` to `Window & typeof window`, which compiles but reads badly.

Scope every run: `npx eslint <explicit file paths> --fix --rule '{"obsidianmd/<rule>":"warn"}'`. Always re-run `tsc --noEmit` afterwards — `prefer-window-timers` changes the return type of every fixed call from `NodeJS.Timeout` to `number`.

### Unused eslint-disable directives

The bot flags directives that suppress rules that aren't actually triggered. Remove them. Note: the bot's scanner may differ from local ESLint — a directive can be "unused" per the bot but needed locally (or vice versa). When this happens, keep the directive with a description (bot ignores it anyway) and ensure local ESLint passes.

### CSS lint warnings (`!important`, `:has`)

**Observed**: 2026-10-17, the plugin directory's CSS lint

The CSS lint reports both as warnings, which are advisory, not approval-blocking.

- **`!important`**: the lint flags every one. Prefer setting, on the element itself, a variable Obsidian's winning rule reads; keep `!important`, with a comment naming the competing selector and its specificity, only where that rule is gated on state classes and sets a literal value. See `obsidian-css-specificity.md`.
- **`:has()`**: the lint flags every one, citing broad selector invalidation, however narrowly it is anchored; Obsidian's own `app.css` uses 14 `:has()` selectors (1.14.4). Replace one where something else can stand in:
  - **A state the script already knows**: when the plugin's own code adds or removes what the `:has()` tests, set a class or attribute there instead — an attribute on a list's rows container kept only while the list has entry rows, for one.
  - **`display: contents`**: when the `:has()` only draws a hidden element's text elsewhere, give the element `display: contents` instead. It has no box, so a check for shown elements skips it, while its children render in its place as real text; see `declarative-settings-api.md`.
  - A `:has()` nested inside another `:has()` is discarded by Electron outright; see `electron-css-quirks.md`.
- **Local check**: the five TS plugins flag every `:has()` locally through `@eslint/css`'s `css/selector-complexity` rule (`disallowPseudoClasses: ['has']`), as a warning to match the directory. `@eslint/css` cannot parse SCSS, so `dynamic-views` lints its compiled, untracked `styles.css`: run `npm run css` before `eslint .`, or there is no CSS to lint.
