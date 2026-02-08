---
title: "Fix Mermaid diagrams invisible in dark mode docs"
slug: fix-mermaid-rendering-dark-mode-docs
tags: [mermaid, dark-mode, diagrams, docs]
problem: "Mermaid diagrams use hardcoded light-mode colors making text invisible on dark backgrounds"
solution_type: fix
created: "2026-02-08"
---

## Problem

Mermaid diagrams render with dark text on transparent backgrounds. In dark mode documentation themes, the text becomes invisible against the dark background. Lines and arrows also disappear because they use dark colors by default.

## Solution

Configure Mermaid to use the `dark` theme when the docs site is in dark mode. In Docusaurus:

```js
// docusaurus.config.js
module.exports = {
  markdown: {
    mermaid: true,
  },
  themes: ['@docusaurus/theme-mermaid'],
  themeConfig: {
    mermaid: {
      theme: {
        light: 'default',
        dark: 'dark',
      },
    },
  },
};
```

For MkDocs Material, add to `mkdocs.yml`:

```yaml
markdown_extensions:
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format

extra_javascript:
  - https://unpkg.com/mermaid/dist/mermaid.min.js

extra_css:
  - stylesheets/mermaid-dark.css
```

```css
/* stylesheets/mermaid-dark.css */
[data-md-color-scheme="slate"] .mermaid {
  --mermaid-font-family: inherit;
  --mermaid-theme: dark;
}
```

For manual initialization:

```js
const isDark = document.documentElement.getAttribute('data-theme') === 'dark';
mermaid.initialize({
  theme: isDark ? 'dark' : 'default',
  startOnLoad: true,
});
```

## Why It Works

Mermaid ships with built-in theme support including `dark`, `neutral`, `forest`, and `default`. The `dark` theme uses light-colored text and lines designed for dark backgrounds. By tying the Mermaid theme to the docs site's color scheme, diagrams remain readable in both modes.

## Context

- Framework: Docusaurus 3.x or MkDocs Material 9.x
- Library: Mermaid 10.x+
- Also works with custom docs sites that initialize Mermaid manually
