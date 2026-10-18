---
title: Declarative settings API quirks
description: Runtime behavior of Obsidian 1.13's getSettingDefinitions() API that the official docs don't state. Covers definition caching, when render callbacks re-run, the keys that match rows across renders, refresh semantics, the re-render skip that applies only to focused control rows, which definitions without a control render, where `disabled` applies, row reuse, the row-batching rule that decides every box boundary, where a row's icon renders, search indexing, which rows keyboard navigation stops at and how a text-only row stays out of it, and a name collision that blanks the settings pane.
author: 🤖 Generated with Claude Code
updated: 2026-10-18
---
# Declarative settings API quirks

Obsidian 1.13 added a declarative settings API: a `PluginSettingTab` overrides `getSettingDefinitions()` and returns a tree of definitions instead of building rows imperatively in `display()`. The framework renders the tree, binds each `control` to a settings key, and indexes every row for global settings search.

The findings below were established by probing a live Obsidian instance, not read from the docs. Several contradict what the documentation implies.

**Observed**: 2026-08-03, Obsidian 1.13.4 (installer 1.13.4)

## Definitions are captured once, not per render

`getSettingDefinitions()` runs **once**, when `addSettingTab()` registers the tab. It does **not** re-run when the user opens the tab, and it does **not** re-run when the user navigates into a sub-page. Only `update()` re-runs it.

The documentation's phrasing ("called on every `display()`") is misleading, because `display()` is bypassed entirely once `getSettingDefinitions()` returns a non-empty array.

Consequences:

- **Anything computed at definition time is frozen** until `update()`. A `type: 'list'` whose `items` are mapped from an array renders the array as it was at registration. Every mutation — add, delete — must call `update()` or the list silently shows stale contents.
- **Closures stay live.** `visible` / `disabled` predicates and `render` callbacks capture references, so they read current state whenever they are invoked. It is the surrounding *structure* that is frozen, not the values the callbacks read.
- **`render` callbacks do re-run on every tab display.** Only the definition tree is cached, not the work its callbacks do: switching away from the tab and back twice ran `render` twice and `getSettingDefinitions()` zero times. So state probed *inside* `render` — whether another plugin is installed, say — is current every time the user opens the tab, while the same probe evaluated at definition time stays frozen until `update()`.

## Rows are matched across renders by key

**Observed**: 2026-10-15, Obsidian 1.14.4 (read from `app.js`)

When `update()` rebuilds a page, the renderer matches each new definition to an already-rendered row by a key, and a match reuses that row's element (see "The row element is reused across renders" below).

| Definition | Key, first present wins |
|---|---|
| Row | `id`, `control.key`, `name`, its index among its rendered siblings |
| Page row | `id`, `name`, its index |
| Group or list | `id`, `heading`, its index |

- **`id` is the explicit key**: the 1.14.0 changelog adds an optional `SettingDefinitionItem#id`, "like `<div key={key} />` in React", for sibling settings that share a name. The npm typings may not declare it yet (1.14.4's do not), so its absence from `obsidian.d.ts` is no sign it is unsupported.
- **Colliding siblings log an error**: two siblings with one key log `duplicate setting key "…"`, which the message says "may cause unexpected behavior when re-rendering". Two rows with the same `name` and no control collide this way until each gets an `id`.
- **A page's path is its `id ?? name`**: navigating into a sub-page matches each step of the path by it, and two sibling pages sharing one log `Settings: duplicate page id …`. An `id` keeps the path stable when the page's `name` changes.

## `refreshDomState()` does not re-read control values

`refreshDomState()` re-evaluates `visible` and `disabled` predicates and applies the result to existing DOM. It does **not** re-read control values through `getControlValue()`.

The failure this produces is quiet and easy to miss: a cascade force-writes a sibling setting to `false`, the stored value is correctly `false`, and the sibling's toggle keeps rendering **on** — correctly greyed out, but showing the wrong state.

| Situation | Correct call |
|---|---|
| A predicate's inputs changed (show/hide, enable/disable) | `refreshDomState()` |
| A **value** on another control was written | `update()` |
| The set of definitions changed (rows added/removed) | `update()` |

A useful nuance: controls on a **freshly mounted sub-page** do re-read `getControlValue()` even though definitions aren't rebuilt. So a cascade that writes to a control on a *different* page self-corrects when the user navigates there; only same-page writes strictly require `update()`. Using `update()` for all value-writing cascades is simpler and safe.

## `update()` skips a focused **control** row

**Observed**: 2026-08-18, Obsidian 1.13.7 (installer 1.13.4)

`update()` re-runs `getSettingDefinitions()` and rebuilds rendered rows from the result — but a row is **left untouched** while it holds DOM focus *and* its definition carries a `control`, presumably so a re-render can't clobber a control the user is interacting with.

The skip is conditional, not universal. The predicate is `"control" in def && !!def.control` combined with the focused element sitting inside the row's control container. A **`render` row is therefore never skipped**: it is torn down (its cleanup function runs), rebuilt, and its control container refocused afterwards — so a button inside it survives `update()` with focus intact, and a `render` row may safely rebuild itself from its own click handler.

This is invisible until a control definition's `name` or `desc` depends on state, because that is the only part `update()` would have rebuilt:

| What triggered `update()` | Rebuilt? |
|---|---|
| A sibling row's `desc` | Yes |
| The focused **control** row's own `desc` | **No** |
| The focused **`render`** row's own `desc` | Yes |
| The focused control row's `desc`, after focus moves away and `update()` runs again | Yes |

The trap is that a user clicking a control **leaves that control focused**, and the natural place to call `update()` is that control's own `setControlValue`. So a state-dependent `desc` on the very control row the user just toggled is the one case that never refreshes — it stays stale for the rest of the settings session, since nothing else moves focus and re-runs `update()`.

Reproduction: give a toggle a `desc` built from its own key's value, click it, and read the row's description. The stored value flips, the group `visible` predicates re-apply, and the description does not change. That repro uses a control row, which is why the finding first read as universal.

- **Don't make a control row's own `name`/`desc` depend on that row's control value.** State the text unconditionally, move the conditional part to a sibling row or a group the predicate can hide, or make it a `render` row and mutate the text directly.
- **A sibling row's text is safe** — cascades that reword a *different* row do rebuild.
- Nothing here affects `visible`/`disabled`; those re-apply on the focused row normally.

## A definition with no `control`, `render`, or `action` renders only with a `name`

**Observed**: 2026-10-15, Obsidian 1.14.4

The documentation lists "a setting with none of the above" as a valid shape, "useful for headings or static informational rows", and the typings declare a matching `SettingDefinitionEmpty`. Before rendering a group's or list's rows, the renderer keeps only definitions with a truthy `name`, `render`, `control`, or `action`, so such a definition shows or not by its `name` alone:

- **A named row renders as a static row**: `{ name, desc }` shows its name and description over an empty control area.
- **A description-only row is silently dropped**: with `name: ''` and only a `desc`, the row is filtered out before rendering, and nothing is logged.

An informational row with no visible name (a call-to-action, a link to another plugin, an explanatory paragraph) therefore needs a `render` callback even when it mounts nothing but text:

```ts
{
  name: '',
  render: (setting) => {
    setting.setDesc(
      createFragment((frag) => {
        frag.createEl('p').appendText('…');
      })
    );
  },
}
```

`setDesc()` writes into `descEl`, which the framework does clear between renders — so this avoids the accumulation problem that hand-appending to `settingEl` causes (see the render-row constraints below).

## Controls have no `onChange` — `setControlValue` is the only hook

`SettingControlBase` exposes exactly `key`, `defaultValue`, `validate`, and `disabled`. There is no per-control change callback.

Every `control` write is routed through the tab's `setControlValue(key, value)`, which makes that method the single place for side effects — cross-setting cascades, cache invalidation, re-initialising a subsystem after a value changes.

```ts
async setControlValue(key: string, value: unknown): Promise<void> {
  writeValue(key, value);
  runCascadeFor(key);          // mutate other settings directly
  await this.plugin.saveData(/* … */);
  needsRerender(key) ? this.update() : this.refreshDomState();
}
```

**Re-entrancy is not a concern** provided cascades mutate the settings object directly rather than calling `setControlValue` again. That keeps it to one save per user action.

Reaching for `render` instead — to get an `onChange` — is the wrong trade: it costs the automatic persistence and gains nothing that the `setControlValue` funnel doesn't already provide.

## `render` rows: `disabled` and row reuse

### `disabled` works at runtime, but the typings reject it

**Observed**: 2026-10-15, Obsidian 1.14.4

The 1.13.6 changelog adds `SettingDefinitionBase#disabled`. The renderer reads `disabled` from any definition that has the key, falls back to `control.disabled` only when the key is absent, and passes the value, or the predicate's result, to `setting.setDisabled()` after every render and on every `refreshDomState()`.

- **A `render` row disables like any other**: `disabled: true` adds `is-disabled` to the row and disables the buttons its callback added with `setting.addButton()`, and a predicate re-applies on `refreshDomState()`.
- **The definition's `disabled` beats the control's**: a control row with `disabled: true` and `control.disabled: false` renders disabled.
- **The 1.14.4 typings lag behind**: `SettingDefinitionBase` still declares no `disabled`, so a `render` definition carrying it fails to compile (`'disabled' does not exist in type 'SettingDefinitionRender'`) without a cast.
- **`setting.setDisabled()` inside the render body avoids the cast** but re-applies only on `update()`, since `refreshDomState()` does not re-run `render`.

### The row element is reused across renders

The framework reuses the same `.setting-item` element between renders and only rebuilds its info and control children. Anything appended **directly to `settingEl`** survives teardown and accumulates one copy per render — a table doubles on every `update()`: 14 rows, then 28, then 56.

Controls added via `setting.addButton()` / `addToggle()` are unaffected, because those live in `controlEl`, which the framework does clear. Only hand-appended DOM leaks.

The obvious fix — `settingEl.empty()` — is a trap: it removes the name element and **silently drops the row from global settings search**. The row still renders; it just stops being findable.

Correct pattern: remove only your own previously-mounted container, then create a fresh one.

```ts
function mountHost(settingEl: HTMLElement): HTMLElement {
  settingEl
    .querySelectorAll(':scope > .my-plugin-host')
    .forEach((stale) => stale.remove());
  return settingEl.createDiv({ cls: 'my-plugin-host' });
}
```

## Search indexing

- **`render` rows are indexed** by their `name`, exactly like `control` rows. Choosing `render` to keep an icon or a labelled button does not cost searchability.
- **Search results navigate into sub-pages.** A setting buried in a `type: 'page'` is findable from the top-level search field, and clicking the result opens its page.
- **A row with `name: ''` is effectively unsearchable** — useful for decorative call-to-action rows that aren't settings, and a silent defect anywhere else.
- **`searchable: false`** keeps user-data rows (list entries, for example) out of the index while leaving the section heading findable.

## Keyboard navigation stops only at shown rows

**Observed**: 2026-10-15 to 2026-10-18, Obsidian 1.14.4 — measured, with the navigation read from `app.js`

- **What the arrow keys visit**: the settings keyboard navigation (`getNavigableSettingItems` and `getTabSequence`) visits every `.setting-item:not(.setting-item-heading)`, `.setting-group-search` and heading `.extra-setting-button` for which `isShown()` is true.
- **A hidden row is skipped**: a row hidden with `display: none` — which Obsidian's own `visible` toggling writes inline — is not visited.
- **An empty group hides whole**: a group with no visible rows is hidden along with its heading.
- **Shown but not visited**: give a row of text meant to be read, not stepped to, `display: contents`. `isShown()` is `!!this.offsetParent`, which is false for an element without a box, so the arrow keys skip the row, while its children lay out in its place as block children of `.setting-items` and its text stays real DOM text.
  - **`visible` still hides it**: the inline `display: none` that `visible` writes outranks the class.
  - **Its empty parts take no space**: Obsidian hides the empty control, and the icon in a `mod-no-icon` group, and the empty name lays out 0 px tall; setting all three to `display: none` is a guard only.
  - **Keep it first in its box**: the row's own divider `::before` is still generated, positioned against an outer box, and Obsidian suppresses it only on `:first-child`.
- **Accessibility**: a screen reader reads such text only when it is real DOM text, which `display: contents` keeps and a pseudo-element drawing the text elsewhere gives up.

## `renderTab` is a reserved method name

`SettingTab.prototype.renderTab()` is what the core settings modal calls to draw a tab; its base implementation is roughly `this.settingItems.length > 0 ? renderDeclarative(this) : this.display()`.

Defining a method called `renderTab` on a `PluginSettingTab` subclass — for example a private helper that renders an internal sub-view — **shadows it by name**, so the framework's version never runs and `display()` is never reached.

The symptom is a **completely blank settings pane**, with no console error. Avoid the name entirely.

## Styling: the row is a flex container

`.setting-item` is `display: flex; flex-wrap: nowrap`. A container mounted into it becomes a third flex sibling alongside the info and control children, so wide content is squeezed into whatever width is left over — a full-width table can end up at roughly 40% of the row.

```css
.setting-item:has(> .my-plugin-host) { flex-wrap: wrap; }
.setting-item > .my-plugin-host { flex-basis: 100%; }
```

Two related migration hazards when moving imperative settings UI to the declarative API:

- **Descendant rules scoped to a page-level wrapper stop matching.** Declarative rows have no such ancestor, so a rule like `.my-page-wrapper .some-icon { … }` silently dies and the element falls back to default block layout. Prefer plugin-owned classes with no ancestor dependency.
- **Sibling combinators break.** Rules of the form `.some-table + .setting-item` no longer match, because the table now sits *inside* a `.setting-item` rather than beside one.

## `ConfirmationModal`

`ConfirmationModal extends Modal`, so it inherits `setTitle()` and `setContent(string | DocumentFragment)`. Buttons auto-close the modal on click; return a truthy value from the handler to keep it open.

`ButtonComponent.setWarning()` is **deprecated**. The replacement is `setDestructive()` for a destructive button, or `setDestructive().setCta()` for a destructive *primary* action — the latter produces the filled treatment. Both resolve to the same classes the deprecated call produced.

`addCancelButton()` supplies its own localized label; pass no argument.

## Nesting rules

- `SettingDefinitionGroup.items` is typed `SettingGroupItem[]` = `SettingDefinition | SettingDefinitionPage`. **A group cannot contain another group or a list.** Pages *can* nest inside groups.
- A page's `items` is `SettingDefinitionItem[]`, which does admit groups and lists.

### A row's `icon` renders only inside a list

**Observed**: 2026-10-15, Obsidian 1.14.4 (read from `app.js`)

The typings declare no `icon` on a setting definition, yet the renderer reads one in a single place:

- **List rows only**: for every row of a `type: 'list'`, the renderer sets the row's icon to `def.icon`, which counts only on a row with no `control` and no `render` — an `action` row or a plain named row.
- **Empty slots hide together**: every row of a list gets an icon slot, and a list where no row carries an icon takes `mod-no-icon`, which hides them all.
- **Ignored everywhere else**: a top-level row, a row in a `type: 'group'`, and a `control` or `render` row in any container never show `def.icon`. Such a row still needs a `render` callback that inserts the icon into `setting.nameEl`.

### Consecutive plain rows fold into one box

The renderer batches as it walks a page's `items`. Each run of consecutive plain rows is collected into a **synthetic headingless group**, and any explicit `type: 'group'` or `type: 'list'` closes that run and starts a new batch.

This one rule decides every box boundary on a settings page:

- A group or list **ends the current box** regardless of what its `visible` predicate returns — a hidden group still closes the batch.
- A lone row's box **depends on its neighbours, not on itself**. Wrapping it in an explicit `type: 'group'` with no `heading` is what guarantees it a box of its own, so a later reorder cannot silently merge it into the rows above. The wrapper is load-bearing, not decorative.
- Rows meant to read as belonging to a master toggle must be placed **before the next group**, not after it. Placement decides box membership; there is no indentation that would override it.

A headingless group is legal and renders correctly. `heading` is optional on `SettingDefinitionGroup`, and the re-render path calls `setHeading("")`, which detaches the element rather than leaving an empty one behind.

### Indented sub-settings split the section they sit in

There is no "child row" concept. A cluster of settings that only applies while a parent toggle is on has to be its own sibling `type: 'group'` with a `visible` predicate, styled through `cls` to read as a continuation of the row above it.

The consequence is structural, and follows from the batching rule above: when the sub-group is hidden, the rows before and after it stay in two separate boxes with a visible gap between them.

```
Row A ─┐
Row B  │ box 1
Row C ─┘
[sub-group, hidden]     ← still ends box 1
Row D ─── box 2         ← visually orphaned
```

Put the row that gates the sub-group **last** in its section, so nothing follows the split.

## Native lists

`type: 'list'` provides drag-to-reorder handles, per-row delete buttons, a platform-appropriate add affordance (tooltipped from `addItem.name`), and an `emptyState` shown at zero entries — all working out of the box.

`onReorder` fires after the DOM is already reordered, so it only needs to persist; it does not need `update()`. `onDelete` and `addItem` both do, because they change the `items` array (see the caching section above).

When row handlers need to address a specific entry, **close over the entry object rather than its index**. Reorder mutates the array without re-rendering, so a captured index points at the wrong row afterwards.
