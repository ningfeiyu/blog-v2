---
title: "Markdown Rendering Test"
date: 2026-01-15
draft: false
tags: ["test", "markdown"]
---

<!--
  REGRESSION FIXTURE for hugo-noir Issue #12 (Markdown/typography rendering).
  This page intentionally exercises every Markdown element the theme styles
  via the Tailwind Typography plugin and the table render hook
  (layouts/_default/_markup/render-table.html): headings, paragraphs,
  ordered/unordered lists, blockquotes, inline code, fenced code blocks
  (including tab characters and long lines for overflow), tables, and links.
  It is not real content — keep it so future Tailwind/config changes can be
  smoke-tested in light and dark mode, desktop and mobile widths.
-->

## Heading level two

This is **bold** and *italic*.

### Heading level three

- item one
- item two

1. first step
2. second step

> A blockquote

`inline code`

```text
some code
that should scroll
A	B
1	2
```

| Column A | Column B | Column C |
|----------|----------|----------|
| a        | b        | c        |
| 1        | 2        | 3        |

Here is a [link](https://example.com) to test visibility in both themes.
