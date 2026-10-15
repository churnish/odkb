---
title: Reading view's inline HTML
description: Which text of a line's inline HTML Reading view shows. Covers the render order, raw-text elements and where they end, elements hidden by CSS or element semantics, elements the sanitizer drops, block start tags that close the paragraph, and lines that open an HTML block, where no Markdown is parsed.
author: 🤖 Generated with Claude Code
updated: 2026-10-15
---
# Reading view's inline HTML

Reading view turns a line into visible text in two steps: Obsidian's Markdown parser renders the line to HTML, which its sanitizer cleans, then the browser parses that HTML and applies CSS. Which part of a line's inline HTML shows follows from that order. It matters to a plugin that reproduces what Reading view shows for a line — a title, a preview, a search snippet — without rendering it.

**Observed**: 2026-10-12 to 2026-10-15, Obsidian 1.14.4 (Electron 43) — each example is the `innerText` of an attached container rendered with `MarkdownRenderer.render`, a technique with a precondition described in "MarkdownRenderer from the DevTools console" in `undocumented-obsidian-apis.md`

## What hides text

| Construct | What shows |
|---|---|
| `script`, `style`, `xmp`, `template`, `noscript`, `title`, `iframe`, `noembed`, `noframes` | Nothing from the open tag to the first literal close tag of the same element |
| `plaintext` | Nothing from the open tag to the end of the line, a `</plaintext>` included |
| The `hidden` attribute | Nothing, unless its value is `until-found`; SVG elements ignore it |
| An inline `display` | Nothing when the winning declaration is `none`: the last valid `!important` one, else the last valid one |
| An inline `visibility` | Nothing under `hidden` or `collapse` |
| `details` without `open` | Its first child `summary` only |
| `dialog` without `open` | Nothing |
| `video`, `audio`, `canvas`, `meter`, `progress`, `datalist`, `rp` | Nothing |
| `svg` | Its `text` elements only |
| `math` | Its token elements only, such as `mi`, which shows in mathematical italic |
| `object`, `embed`, `slot`, `search`, `listing` | Their text, whatever their attributes say: the sanitizer drops the element and keeps its content |
| `foreignObject` | Nothing: the sanitizer drops it with its content |

Examples: `x <span hidden>a</span> b` shows `x b`, `x <span style="display:none; display:inline">a</span> b` shows `x a b`, `x <details><summary>s</summary>a<summary>t</summary></details> b` shows `x`, `s`, `b`, and `x <search hidden>a</search> b` shows `x`, `a b`.

## Raw-text elements end at the first literal close tag

The Markdown is rendered first, so a close tag counts where the rendered HTML has it:

- **Inside a construct, it still ends the element**: bold, italic and link text are rendered around it, not escaped, so `x <script>a **</script>** b</script> c` shows `x b c`.
- **A code span or an escape hides it**: both reach the browser as escaped text, so `` x <script>a `</script>` b</script> c `` and `x <script>a \</script> b</script> c` show `x c`.
- **An attribute value or a comment does not**: the browser's raw-text state ends at the first literal close tag wherever it sits, so `x <script>a <!-- </script> --> b</script> c` shows `x --> b c`, and `x <script>a <i title="</script>"> b</script> c` shows `x "> b c`.
- **A link destination does not either**: the close tag ends the element inside Obsidian's own link markup, whose attribute tail then shows — `x <script>a [l](u</script>) b</script> c` shows `x " data-href="u" class="internal-link">l b c`.
- **Hidden either way**: the sanitizer removes `script`, `style`, `xmp`, `noscript`, `noembed` and `noframes` with their content, while `template`, `title` and `iframe` stay in the DOM with nothing displayed.

## Block start tags close the paragraph

A block-level start tag inside a paragraph — `div`, `details`, `dialog`, `search`, `listing`, `xmp` and the like — closes the paragraph and every element open inside it, so an inline ancestor stops hiding there: `x <span hidden>a <div>b</div> c</span> d` shows `x`, `b`, `c d`.

## Lines that open an HTML block

A line whose content, after any quote, list or callout marker, opens an HTML block reaches the browser as typed.

- **What opens one**: a `script`, `pre` or `style` tag, a comment, a processing instruction, a declaration, CDATA, a block-level tag such as `div`, `details` or `hgroup` in any case, or one complete tag alone on the line (`<span>`).
- **What does not**: a heading's content, which is parsed as inline Markdown; an inline or custom tag followed by more text (`<span> a`, `<custom-el> a`); and `<search>`, which Obsidian's parser does not count as block-level.
- **No Markdown is parsed**: escapes keep their backslash, a block ID stays visible, and bold and wikilinks stay literal — `<div>a</div> **b** [[l]] \*e\* ^blk` shows `a`, then `**b** [[l]] \*e\* ^blk`.
- **Code spans protect nothing**: with no Markdown, a raw-text element ends at its first literal close tag anywhere — `` <script>a `</script>` b</script> c `` shows `` ` b c ``.
- **The metadata cache indexes no tags there**: see "Which body tags the metadata cache indexes" in `obsidian-api-quirks.md`.
