---
title: "Docs-as-Code Workflow with PR Reviews, Linting, and CI"
slug: "pattern-docs-as-code-workflow"
solution_type: pattern
tags:
  - docs-as-code
  - workflow
  - git
  - review
created: "2026-02-08"
---

## Problem

Documentation quality degrades over time because docs lack the same review, testing, and deployment rigor that application code receives.

## Solution

Treat documentation exactly like source code: store it in Git, review changes through pull requests, enforce quality with linters, and deploy through CI/CD pipelines.

### 1. GitHub Actions CI Workflow

Create `.github/workflows/docs.yml` to lint and build docs on every PR:

```yaml
name: Docs CI

on:
  pull_request:
    paths:
      - "docs/**"
      - ".vale/**"
      - ".github/workflows/docs.yml"

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Vale
        uses: errata-ai/vale-action@v2
        with:
          files: docs/
          vale_flags: "--config=.vale.ini"

      - name: Check for broken links
        uses: lycheeverse/lychee-action@v1
        with:
          args: --verbose --no-progress "docs/**/*.md"
          fail: true

  build:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npm run docs:build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: docs-site
          path: docs/.vitepress/dist

  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: build
    permissions:
      pages: write
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: docs-site
          path: ./site

      - uses: actions/deploy-pages@v4
```

### 2. Vale Linter Configuration

Create `.vale.ini` at the repo root:

```ini
StylesPath = .vale/styles
MinAlertLevel = warning

Packages = Google, write-good, alex

[docs/*.md]
BasedOnStyles = Vale, Google, write-good, alex
Google.Passive = warning
Google.We = warning
Google.WordList = error
write-good.Weasel = warning
write-good.TooWordy = warning
alex.Condescending = error
```

Create custom rules in `.vale/styles/Custom/Acronyms.yml`:

```yaml
extends: existence
message: "Define acronym '%s' on first use."
level: warning
scope: sentence
tokens:
  - '\b[A-Z]{2,5}\b'
exceptions:
  - API
  - URL
  - HTTP
  - HTML
  - CSS
  - JSON
  - YAML
  - CI
  - CD
```

### 3. Pull Request Template

Create `.github/PULL_REQUEST_TEMPLATE/docs.md`:

```markdown
## What changed

<!-- Describe the documentation change -->

## Why

<!-- Link to issue, user feedback, or reason for the change -->

## Checklist

- [ ] Content is technically accurate (verified against source code or SME)
- [ ] Vale linter passes with no errors
- [ ] All links resolve (no 404s)
- [ ] Images have alt text
- [ ] Frontmatter metadata is complete (title, description, last_reviewed)
- [ ] Tested locally with `npm run docs:dev`
- [ ] If API change: corresponding API reference updated
```

### 4. Pre-commit Hook

Add a local pre-commit hook via `.husky/pre-commit` to catch issues before CI:

```bash
#!/bin/sh
# Only lint changed markdown files
CHANGED_DOCS=$(git diff --cached --name-only --diff-filter=ACM | grep '\.md$' | grep '^docs/')

if [ -n "$CHANGED_DOCS" ]; then
  echo "Linting changed docs..."
  echo "$CHANGED_DOCS" | xargs vale --config=.vale.ini
  if [ $? -ne 0 ]; then
    echo "Vale found issues. Fix them before committing."
    exit 1
  fi
fi
```

## Why It Works

This workflow catches problems at three stages: locally before commit (pre-commit hooks), during review (PR checks and human review), and before deploy (CI build verification). The same Git-based collaboration that makes code review effective -- diff visibility, inline comments, approval gates -- works equally well for prose. Vale enforces consistent style without manual effort, and the CI pipeline guarantees that broken links and build errors never reach production.

## Context

This pattern is most valuable for teams where documentation lives alongside code in the same repository. It assumes familiarity with GitHub Actions (adaptable to GitLab CI or other platforms), Node.js-based static site generators like VitePress or Docusaurus, and markdown as the authoring format. Teams using a separate CMS or wiki should adapt the CI steps to their toolchain but can still apply the review and linting principles.
