---
title: "Configure MkDocs Material with all recommended plugins"
slug: config-mkdocs-material-full-setup
tags: [mkdocs, material, setup, plugins]
problem: "MkDocs Material has dozens of optional plugins that require a well-structured mkdocs.yml"
solution_type: config
created: "2026-02-08"
---

## Problem

MkDocs Material is one of the most feature-rich documentation themes available, but configuring it from scratch means assembling settings from dozens of separate documentation pages. Teams either under-configure and miss useful features like social cards and search suggestions, or they over-configure and end up with a bloated `mkdocs.yml` full of options they do not understand. Getting the combination of plugins, extensions, and theme features right on the first try is rare.

## Solution

Use the following `mkdocs.yml` and `requirements.txt` as a starting point. Every section is annotated so you can understand what each setting does and remove what you do not need.

### requirements.txt

```txt title="requirements.txt"
mkdocs>=1.6.0
mkdocs-material>=9.6.0
mkdocs-material-extensions>=1.3.0
mkdocs-minify-plugin>=0.8.0
mkdocs-git-revision-date-localized-plugin>=1.3.0
mkdocs-glightbox>=0.4.0
pymdown-extensions>=10.12
pillow>=10.4.0
cairosvg>=2.7.0
```

### mkdocs.yml

```yaml title="mkdocs.yml"
# ==============================================================================
# Project metadata
# ==============================================================================
site_name: My Docs
site_url: https://docs.example.com
site_description: Documentation for My Product
site_author: My Org
repo_url: https://github.com/my-org/my-docs
repo_name: my-org/my-docs
edit_uri: edit/main/docs/
copyright: Copyright &copy; 2026 My Org

# ==============================================================================
# Theme configuration
# ==============================================================================
theme:
  name: material
  language: en
  logo: assets/logo.svg
  favicon: assets/favicon.ico

  palette:
    # Light mode
    - media: "(prefers-color-scheme: light)"
      scheme: default
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode
    # Dark mode
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-4
        name: Switch to light mode

  font:
    text: Inter
    code: JetBrains Mono

  features:
    # Navigation
    - navigation.instant         # XHR-based page loads (SPA feel)
    - navigation.instant.progress # Show loading progress bar
    - navigation.tracking        # Update URL hash on scroll
    - navigation.tabs            # Top-level sections as tabs
    - navigation.tabs.sticky     # Tabs stay visible on scroll
    - navigation.sections        # Render top-level sections as groups
    - navigation.expand          # Auto-expand sections in sidebar
    - navigation.top             # Back-to-top button
    - navigation.footer          # Previous/next page links in footer

    # Table of contents
    - toc.follow                 # TOC follows scroll position

    # Search
    - search.suggest             # Show search suggestions while typing
    - search.highlight           # Highlight matches on target page
    - search.share               # Shareable search URLs

    # Content
    - content.action.edit        # Show edit button on each page
    - content.action.view        # Show view source button
    - content.code.copy          # Copy button on code blocks
    - content.code.annotate      # Code annotation support
    - content.tabs.link          # Linked content tabs

  icon:
    repo: fontawesome/brands/github

# ==============================================================================
# Plugins
# ==============================================================================
plugins:
  # Built-in search (required, do not remove)
  - search:
      separator: '[\s\-\.]+'
      lang:
        - en

  # Git revision date on each page footer
  - git-revision-date-localized:
      enable_creation_date: true
      type: timeago
      fallback_to_build_date: true

  # Image lightbox
  - glightbox:
      touchNavigation: true
      loop: false
      effect: zoom
      slide_effect: slide
      width: 100%

  # HTML minification for production builds
  - minify:
      minify_html: true
      minify_js: true
      minify_css: true
      htmlmin_opts:
        remove_comments: true

  # Social cards (generates og:image cards automatically)
  - social:
      cards_layout_options:
        background_color: "#4f46e5"
        color: "#ffffff"

# ==============================================================================
# Markdown extensions
# ==============================================================================
markdown_extensions:
  # Python Markdown built-in extensions
  - abbr                        # Abbreviation definitions
  - admonition                  # Callout boxes (note, tip, warning)
  - attr_list                   # Add HTML attributes to Markdown elements
  - def_list                    # Definition lists
  - footnotes                   # Footnote references
  - md_in_html                  # Markdown inside HTML blocks
  - tables                      # Standard tables
  - toc:
      permalink: true
      permalink_title: Link to this section
      toc_depth: 4

  # PyMdown Extensions
  - pymdownx.arithmatex:        # LaTeX math rendering
      generic: true
  - pymdownx.betterem:          # Improved emphasis handling
      smart_enable: all
  - pymdownx.caret              # Superscript with ^^text^^
  - pymdownx.critic             # Critic markup for suggestions
  - pymdownx.details            # Collapsible admonitions
  - pymdownx.emoji:             # Emoji support
      emoji_index: !!python/name:material.extensions.emoji.twemoji
      emoji_generator: !!python/name:material.extensions.emoji.to_svg
  - pymdownx.highlight:         # Code block syntax highlighting
      anchor_linenums: true
      line_spans: __span
      pygments_lang_class: true
  - pymdownx.inlinehilite       # Inline code highlighting
  - pymdownx.keys               # Keyboard key rendering (++ctrl+c++)
  - pymdownx.mark               # Highlight text with ==marks==
  - pymdownx.smartsymbols       # Smart quotes, dashes, arrows
  - pymdownx.snippets:          # Include external files
      auto_append:
        - includes/abbreviations.md
  - pymdownx.superfences:       # Enhanced fenced code blocks
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tabbed:            # Content tabs
      alternate_style: true
      slugify: !!python/object/apply:pymdownx.slugs.slugify
        kwds:
          case: lower
  - pymdownx.tasklist:          # Checkbox task lists
      custom_checkbox: true
  - pymdownx.tilde              # Subscript and strikethrough

# ==============================================================================
# Extra configuration
# ==============================================================================
extra:
  social:
    - icon: fontawesome/brands/github
      link: https://github.com/my-org
    - icon: fontawesome/brands/discord
      link: https://discord.gg/example
  generator: false  # Remove "Made with Material for MkDocs" footer text

  # Analytics (uncomment and configure)
  # analytics:
  #   provider: google
  #   property: G-XXXXXXXXXX

  # Version selector (requires mike for versioning)
  # version:
  #   provider: mike

extra_css:
  - stylesheets/extra.css

extra_javascript:
  # MathJax for LaTeX rendering
  - javascripts/mathjax.js
  - https://unpkg.com/mathjax@3/es5/tex-mml-chtml.js

# ==============================================================================
# Navigation structure
# ==============================================================================
nav:
  - Home: index.md
  - Getting started:
      - Installation: getting-started/installation.md
      - Quick start: getting-started/quickstart.md
      - Configuration: getting-started/configuration.md
  - Guides:
      - guides/index.md
  - Reference:
      - reference/index.md
  - Blog:
      - blog/index.md
```

### Supporting files

Create the MathJax configuration file:

```js title="docs/javascripts/mathjax.js"
window.MathJax = {
  tex: {
    inlineMath: [['$', '$'], ['\\(', '\\)']],
    displayMath: [['$$', '$$'], ['\\[', '\\]']],
    processEscapes: true,
    processEnvironments: true,
  },
  options: {
    ignoreHtmlClass: '.*|',
    processHtmlClass: 'arithmatex',
  },
};
```

Create the abbreviations include file:

```md title="docs/includes/abbreviations.md"
*[HTML]: Hyper Text Markup Language
*[CSS]: Cascading Style Sheets
*[JS]: JavaScript
*[API]: Application Programming Interface
*[CLI]: Command Line Interface
*[CI]: Continuous Integration
*[CD]: Continuous Deployment
```

### Build and serve commands

```sh
# Install dependencies
pip install -r requirements.txt

# Local development with live reload
mkdocs serve

# Production build
mkdocs build --strict

# Deploy to GitHub Pages
mkdocs gh-deploy --force
```

## Why It Works

This config file consolidates the most commonly needed MkDocs Material features into a single reference. The annotations explain the purpose of each setting, making it easy to remove features you do not need without accidentally breaking dependent configurations. For example, removing the `pymdownx.superfences` custom fence entry disables Mermaid diagrams but does not affect other code block features. The `requirements.txt` pins minimum versions to avoid compatibility issues between plugins.

## Context

This config targets MkDocs Material 9.6+ (Insiders features are not included). The social cards plugin requires `pillow` and `cairosvg` as dependencies, and Cairo must be installed on the system (`brew install cairo` on macOS, `apt install libcairo2-dev` on Ubuntu). If social card generation fails in CI, install these system dependencies in your workflow. For versioning, install `mike` separately and configure the `extra.version` block. The `--strict` flag on `mkdocs build` catches broken links and missing pages at build time.
