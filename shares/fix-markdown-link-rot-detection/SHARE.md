---
title: "Fix undetected link rot in large documentation sites"
slug: fix-markdown-link-rot-detection
tags: [markdown, links, linkrot, ci]
problem: "External links in docs silently break over time with no notification or CI check"
solution_type: fix
created: "2026-02-08"
---

## Problem

Documentation sites accumulate hundreds of external links over time. When linked pages move, get deleted, or change their URL structure, the documentation links break silently. Users encounter 404 errors and lose trust in the docs. There's no automated way to detect these broken links.

## Solution

Add `lychee` as a CI step to check all links on every push:

```yaml
# .github/workflows/link-check.yml
name: Link Check
on:
  push:
    branches: [main]
  schedule:
    - cron: '0 8 * * 1'  # Weekly on Monday

jobs:
  linkcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check links
        uses: lycheeverse/lychee-action@v2
        with:
          args: >
            --verbose
            --no-progress
            --accept 200,204,301,302
            --exclude-loopback
            --exclude 'example\.com'
            --exclude 'localhost'
            './docs/**/*.md'
            './docs/**/*.mdx'
          fail: true
```

Create a `.lychee.toml` config for persistent settings:

```toml
# .lychee.toml
exclude_path = ["node_modules"]
max_retries = 3
timeout = 30
accept = [200, 204, 301, 302]
exclude = [
  "example\\.com",
  "localhost",
  "127\\.0\\.0\\.1",
]
```

## Why It Works

`lychee` is a fast, Rust-based link checker that validates both internal and external links. Running it weekly via cron catches external link rot before users do. Running it on push catches internal broken links from doc restructuring. The `--accept` flag tolerates redirects, and the `--exclude` patterns skip placeholder URLs.

## Context

- Tool: lychee (Rust-based link checker)
- CI: GitHub Actions
- Also available as a CLI: `brew install lychee` or `cargo install lychee`
- Alternative tools: markdown-link-check, linkinator
