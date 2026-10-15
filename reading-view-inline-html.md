---
title: Reading view's inline HTML
description: Which text of a line's inline HTML Reading view shows. Covers the render order and the sanitizer, raw-text elements and where they end, elements hidden by CSS or element semantics, elements the sanitizer drops or unwraps, how markup inside `svg` and `math` parses and reads, block start tags that close the paragraph, and lines that open an HTML block, where no Markdown is parsed.
author: 🤖 Generated with Claude Code
updated: 2026-10-15
---
# Reading view's inline HTML

Reading view turns a line into visible text in two steps: Obsidian's Markdown parser renders the line to HTML, then its sanitizer parses that HTML with the browser's own parser and cleans the resulting tree, which the browser styles with CSS. Which part of a line's inline HTML shows follows from that order. It matters to a plugin that reproduces what Reading view shows for a line — a title, a preview, a search snippet — without rendering it.

The sanitizer is a bundled DOMPurify 3.0.1, configured with `FORBID_TAGS: ['style']` and `ADD_TAGS: ['iframe']`.

**Observed**: 2026-10-12 to 2026-10-15, Obsidian 1.14.4 (Electron 43) — each example is the `innerText` of an attached container rendered with `MarkdownRenderer.render`, a technique with a precondition described in "MarkdownRenderer from the DevTools console" in `undocumented-obsidian-apis.md`

## What hides text

| Construct | What shows |
|---|---|
| `script`, `style`, `xmp`, `template`, `noscript`, `title`, `iframe`, `noembed`, `noframes` outside `svg` and `math` | Nothing from the open tag to the first literal close tag of the same element |
| `plaintext` outside `svg` and `math` | Nothing from the open tag to the end of the line, a `</plaintext>` included |
| The `hidden` attribute | Nothing, unless its value is `until-found`; SVG elements ignore it |
| An inline `display` | Nothing when the winning declaration is `none`: the last valid `!important` one, else the last valid one |
| An inline `visibility` | Nothing under `hidden` or `collapse` |
| `details` without `open` | Its first child `summary` only |
| `dialog` without `open` | Nothing |
| `video`, `audio`, `canvas`, `meter`, `progress`, `datalist`, `rp` | Nothing |
| `svg` | Its `text` elements only, without their children that are not text content elements, such as `desc` |
| `math` | Its token elements only, such as `mi`, which shows in mathematical italic |
| A name the sanitizer allows only as SVG or MathML, such as `text`, `g`, `feBlend` or `mi`, outside `svg` and `math` | Nothing: the sanitizer's namespace check drops it with its content. `a` and `font`, which HTML shares, keep their HTML behavior, and so does `image`, which the parser reads as `img` |
| `object`, `embed`, `slot`, `search`, `listing`, `animate`, `use`, `semantics`, and unknown names such as `foo` | Their text, whatever their attributes say: the sanitizer disallows the element, so it drops the tag with its attributes and keeps its content |
| `foreignObject`, `annotation-xml` | Nothing: the sanitizer drops them with their content |

Examples: `x <span hidden>a</span> b` shows `x b`, `x <span style="display:none; display:inline">a</span> b` shows `x a b`, `x <details><summary>s</summary>a<summary>t</summary></details> b` shows `x`, `s`, `b`, and `x <search hidden>a</search> b` shows `x`, `a b`.

From the SVG and sanitizer rows: `x <svg><text>a<desc>b</desc></text></svg> c` shows `x`, `a`, `c`, `x <mi>a</mi> b` shows `x b`, and `x <foo hidden>a</foo> b` shows `x a b`.

## Raw-text elements end at the first literal close tag

The Markdown is rendered first, so a close tag counts where the rendered HTML has it:

- **Inside a construct, it still ends the element**: bold, italic and link text are rendered around it, not escaped, so `x <script>a **</script>** b</script> c` shows `x b c`.
- **A code span or an escape hides it**: both reach the browser as escaped text, so `` x <script>a `</script>` b</script> c `` and `x <script>a \</script> b</script> c` show `x c`.
- **An attribute value or a comment does not**: the browser's raw-text state ends at the first literal close tag wherever it sits, so `x <script>a <!-- </script> --> b</script> c` shows `x --> b c`, and `x <script>a <i title="</script>"> b</script> c` shows `x "> b c`.
- **A link destination does not either**: the close tag ends the element inside Obsidian's own link markup, whose attribute tail then shows — `x <script>a [l](u</script>) b</script> c` shows `x " data-href="u" class="internal-link">l b c`.
- **Hidden either way**: the sanitizer removes `script`, `style`, `xmp`, `noscript`, `noembed` and `noframes` with their content, while `template`, `title` and `iframe` stay in the DOM with nothing displayed.

## Block start tags close the paragraph

A block-level start tag inside a paragraph — `div`, `details`, `dialog`, `search`, `listing`, `xmp` and the like — closes the paragraph and every element open inside it, so an inline ancestor stops hiding there: `x <span hidden>a <div>b</div> c</span> d` shows `x`, `b`, `c d`.

## Inside `svg` and `math`

The parser follows foreign-content rules inside `svg` and `math`, which change where elements end:

- **Raw-text elements are ordinary elements**: `script`, `style`, `title` and the other raw-text elements in "What hides text" end like any other element, so `x <svg><title>a</svg> b` shows `x b`.
- **An end tag closes up to the element it names**: an end tag naming an open SVG or MathML element closes every element open inside it, unless one of them is an HTML element — `x <svg><desc>a</svg> b` shows `x b`, and `x <math><mtext>a</math> b</mtext> c` shows `x`, `a`, `b c`.
- **`</p>` and `</br>` break out**: they first close every foreign element above the nearest HTML element or integration point, such as SVG `desc` or MathML `mtext`, then act as in HTML — `x <svg><g>a</p> b</g></svg> c` closes the paragraph and shows `x`, `b c`.
- **`innerText` breaks lines around drawn foreign text**: it puts a line break on each side of the text an SVG `text` element or a MathML token element draws, and between a token element's own text and an HTML child — `x <math><mtext>a<span>b</span> c</mtext></math> d` shows `x`, `a`, `b`, `c`, `d`, so `a` and `b` stay separate words once whitespace collapses.

## Lines that open an HTML block

A line whose content, after any quote, list or callout marker, opens an HTML block reaches the browser as typed.

- **What opens one**: a `script`, `pre` or `style` tag, a comment, a processing instruction, a declaration, CDATA, a block-level tag such as `div`, `details` or `hgroup` in any case, or one complete tag alone on the line (`<span>`).
- **What does not**: a heading's content, which is parsed as inline Markdown; an inline or custom tag followed by more text (`<span> a`, `<custom-el> a`); and `<search>`, which Obsidian's parser does not count as block-level.
- **No Markdown is parsed**: escapes keep their backslash, a block ID stays visible, and bold and wikilinks stay literal — `<div>a</div> **b** [[l]] \*e\* ^blk` shows `a`, then `**b** [[l]] \*e\* ^blk`.
- **Code spans protect nothing**: with no Markdown, a raw-text element ends at its first literal close tag anywhere — `` <script>a `</script>` b</script> c `` shows `` ` b c ``.
- **The metadata cache indexes no tags there**: see "Which body tags the metadata cache indexes" in `obsidian-api-quirks.md`.
