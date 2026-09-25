---
title: Obsidian CSS specificity
description: How Obsidian's app.css uses high-specificity stateful selectors that plugin overrides cannot beat, how to identify the winning rule at runtime, when !important is the correct answer, and generic class names that app.css already styles.
author: 🤖 Generated with Claude Code
updated: 2026-09-25
---
# Obsidian CSS specificity

Plugin stylesheets load after `app.css`, so an override with *equal* specificity wins on source order. That makes it tempting to assume a single extra class is always enough to beat Obsidian. It is not — several `app.css` rules that control visibility are gated on stacked **state classes**, reaching specificity a plugin cannot practically match.

## The trap

A plugin toggles its own class on an Obsidian-owned element and expects `display` to follow:

```scss
/* (0,2,0) — loses */
.metadata-container.my-hidden {
  display: none;
}
```

**Observed**: 2026-08-06, Obsidian 1.13.5 (installer 1.13.4).

`app.css` reveals the editor's property container with:

```css
.markdown-source-view.is-live-preview.show-properties
  .metadata-container:not(.mod-error) {
  display: var(--metadata-display-editing);
}
```

That is **(0,5,0)**: four classes plus `:not(.mod-error)`. Adding one ancestor class to the plugin rule gets to (0,3,0) and still loses. The element keeps rendering, and — because the plugin's own class *is* applied and `matches()` returns true — the failure looks like a JS bug rather than a cascade loss.

## Diagnose before guessing

Do not infer the winner from reading `app.css`. Enumerate the rules that actually match the live element:

```js
const el = document.querySelector('.metadata-container.my-hidden');
const hits = [];
for (const sheet of document.styleSheets) {
  let rules;
  try { rules = sheet.cssRules; } catch { continue; }
  const walk = (list) => {
    for (const r of list) {
      if (r.cssRules && !r.selectorText) { walk(r.cssRules); continue; }
      if (!r.selectorText || !r.style) continue;
      const v = r.style.getPropertyValue('display');
      if (!v) continue;
      for (const sel of r.selectorText.split(',')) {
        try {
          if (el.matches(sel.trim())) hits.push({
            sel: sel.trim(),
            value: v,
            important: r.style.getPropertyPriority('display') === 'important',
            from: sheet.ownerNode?.id || sheet.href?.split('/').pop(),
          });
        } catch { /* unsupported selector */ }
      }
    }
  };
  walk(rules);
}
```

Two notes on this walk:

- **Recurse only into conditional groups** (`@media`, `@supports`, `@layer`). Guard with `!r.selectorText`, or a plain style rule inside a group gets skipped.
- **`sheet.cssRules` can throw**; skip those sheets rather than aborting.

A faster triage that distinguishes *specificity loss* from *`!important` loss* — inject candidates and read the computed value back:

| Test rule | Result means |
|---|---|
| Same selector, no `!important` | Still wrong → something outranks it |
| Same selector + `!important` | Correct → the competitor is beatable by importance |
| Higher-specificity selector, no `!important` | Correct → plain specificity bump is enough |
| Higher specificity fails, `!important` works | Competitor is a high-specificity **non**-important rule |

That last row is the common case, and it is easy to misread as "the competitor must be `!important`" when it isn't.

## Choosing the fix

Prefer, in order:

1. **Qualify with the element's own class or tag.** `input.my-input` (0,1,1) beats `input[type='text']` (0,1,1) on source order; `.setting-item.my-row` (0,2,0) beats `.setting-item` (0,1,0) outright. This is the right fix for the majority of cases and costs nothing.
2. **Scope to a stable structural ancestor** — a container class that is part of the DOM contract, not a state flag.
3. **`!important`, documented** — when the competing selector is gated on state classes.

Reach for (3) only when (1) and (2) genuinely cannot win. The signal is that matching the competitor requires naming Obsidian's *state* classes (`.is-live-preview`, `.show-properties`, `.is-collapsed`, `.is-active`). Such a selector is not merely verbose — it is **fragile in a silent direction**: if Obsidian renames one class, the override stops applying and the plugin's intent (usually hiding something) fails without any error.

```scss
/* REQUIRES !important: Obsidian reveals this container via
   `.markdown-source-view.is-live-preview.show-properties .metadata-container:not(.mod-error)`
   at (0,5,0). Matching that means chaining Obsidian's state classes, so the
   override would fail silently if any were renamed. */
.metadata-container.my-hidden {
  display: none !important;
}
```

## Interaction with the review bot

`eslint-plugin-obsidianmd`'s CSS lint flags every `!important` as a warning and suggests raising specificity instead. That advice is correct for most cases and wrong for this one. Warnings are advisory, not approval-blocking — keep the declaration with a comment naming the competing selector and its specificity. See `obsidian-review-bot.md`.

## A generic class name can collide with `app.css`

**Observed**: 2026-09-25, Obsidian 1.14.2.

Not a specificity loss: a plugin element picks up an unrelated `app.css` rule because its own class name already means something there, so it renders with Obsidian's styling from the moment it mounts.

`.list-item` is one such name. `app.css` styles it as a flex row (`display: flex`) with vertical margins (`margin: var(--size-4-2) 0`), a `gap` and centered items (`align-items: center`), so a plugin element reusing the class inherits all four. Overriding `display` with `inline` neutralizes them together — `gap` and `align-items` need a flex container, and vertical margins do not apply to inline boxes — while a namespaced class avoids the collision outright. Check `app.css` before reusing a name that looks generic.

## Related

- **Verify per element, not per rule.** Bulk computed-style diffs over a settings pane will not cover editor chrome (`.metadata-container`, `.cm-editor` descendants). Check those separately; a specificity loss confined to the editor is invisible in a settings-only sweep.
- **Same-specificity ties depend on load order**, which a *theme* can win. Where the property matters (hiding something), prefer an outright specificity win over a tie, even when the tie currently works.
- See `electron-css-quirks.md` for Blink-level rendering bugs, which are a separate layer from stylesheet cascade.
