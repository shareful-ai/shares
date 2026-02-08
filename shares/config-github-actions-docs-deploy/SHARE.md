---
title: "Configure GitHub Actions to build and deploy docs on merge"
slug: config-github-actions-docs-deploy
tags: [github-actions, ci-cd, deployment, automation]
problem: "Docs deployment is manual or fragile causing stale published docs after content changes"
solution_type: config
created: "2026-02-08"
---

## Problem

Documentation deployments are often an afterthought. Teams either deploy manually (leading to stale published docs that lag behind the source by days or weeks), use a fragile script that breaks silently, or skip deployment validation entirely so broken links and build errors reach production. A proper CI/CD pipeline should build on every PR for preview, validate links and prose, and deploy automatically on merge to main.

## Solution

Use the GitHub Actions workflows below. Two complete configurations are provided: one for Docusaurus and one for MkDocs Material. Both follow the same pattern: validate on PR, deploy on merge.

### Workflow for Docusaurus

```yaml title=".github/workflows/docs.yml"
name: Docs

on:
  push:
    branches: [main]
    paths:
      - 'docs/**'
      - 'blog/**'
      - 'src/**'
      - 'static/**'
      - 'docusaurus.config.js'
      - 'sidebars.js'
      - 'package.json'
      - 'package-lock.json'
  pull_request:
    paths:
      - 'docs/**'
      - 'blog/**'
      - 'src/**'
      - 'static/**'
      - 'docusaurus.config.js'
      - 'sidebars.js'
      - 'package.json'
      - 'package-lock.json'

permissions:
  contents: read
  pages: write
  id-token: write
  pull-requests: write

concurrency:
  group: docs-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ============================================================================
  # Build the docs site
  # ============================================================================
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for git-based features

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build site
        run: npm run build
        env:
          NODE_OPTIONS: '--max-old-space-size=4096'

      - name: Upload build artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: build

  # ============================================================================
  # Check links in the built output
  # ============================================================================
  link-check:
    name: Check links
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: github-pages
          path: artifact

      - name: Extract artifact
        run: |
          mkdir -p build
          tar -xf artifact/artifact.tar -C build

      - name: Check links
        uses: lycheeverse/lychee-action@v2
        with:
          args: >-
            --no-progress
            --exclude-loopback
            --exclude 'linkedin\.com'
            --exclude 'twitter\.com'
            --exclude 'x\.com'
            --suggest
            --max-retries 3
            --timeout 30
            'build/**/*.html'
          fail: true

  # ============================================================================
  # Prose linting
  # ============================================================================
  prose-lint:
    name: Lint prose
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Vale linter
        uses: errata-ai/vale-action@v2
        with:
          files: docs
          reporter: github-pr-review
          fail_on_error: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ============================================================================
  # Deploy to GitHub Pages (only on push to main)
  # ============================================================================
  deploy:
    name: Deploy
    needs: [build, link-check]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### Workflow for MkDocs Material

```yaml title=".github/workflows/docs.yml"
name: Docs

on:
  push:
    branches: [main]
    paths:
      - 'docs/**'
      - 'mkdocs.yml'
      - 'requirements.txt'
  pull_request:
    paths:
      - 'docs/**'
      - 'mkdocs.yml'
      - 'requirements.txt'

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: docs-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ============================================================================
  # Build the docs site
  # ============================================================================
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for git-revision-date-localized plugin

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip

      - name: Install system dependencies
        run: sudo apt-get install -y libcairo2-dev libfreetype6-dev

      - name: Install Python dependencies
        run: pip install -r requirements.txt

      - name: Build site
        run: mkdocs build --strict

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: docs-site
          path: site
          retention-days: 7

  # ============================================================================
  # Check links in the built output
  # ============================================================================
  link-check:
    name: Check links
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: docs-site
          path: site

      - name: Check links
        uses: lycheeverse/lychee-action@v2
        with:
          args: >-
            --no-progress
            --exclude-loopback
            --exclude 'linkedin\.com'
            --exclude 'twitter\.com'
            --exclude 'x\.com'
            --suggest
            --max-retries 3
            --timeout 30
            'site/**/*.html'
          fail: true

  # ============================================================================
  # Prose linting
  # ============================================================================
  prose-lint:
    name: Lint prose
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Vale linter
        uses: errata-ai/vale-action@v2
        with:
          files: docs
          reporter: github-pr-review
          fail_on_error: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ============================================================================
  # Deploy to GitHub Pages (only on push to main)
  # ============================================================================
  deploy:
    name: Deploy
    needs: [build, link-check]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip

      - name: Install system dependencies
        run: sudo apt-get install -y libcairo2-dev libfreetype6-dev

      - name: Install Python dependencies
        run: pip install -r requirements.txt

      - name: Configure Git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Deploy with mkdocs gh-deploy
        run: mkdocs gh-deploy --force --strict
```

### Repository settings for GitHub Pages

Before the workflows will deploy successfully, configure the repository:

1. Go to **Settings > Pages**.
2. Under **Source**, select **GitHub Actions** (for Docusaurus) or **Deploy from a branch** and select the `gh-pages` branch (for MkDocs).
3. Optionally configure a custom domain under **Custom domain**.

### Adding a PR preview comment

For Docusaurus deployments using GitHub Pages, you can add a preview URL comment to PRs by adding this step after the build job:

```yaml
  preview-comment:
    name: Preview comment
    needs: build
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - name: Find existing comment
        uses: peter-evans/find-comment@v3
        id: find-comment
        with:
          issue-number: ${{ github.event.pull_request.number }}
          comment-author: 'github-actions[bot]'
          body-includes: 'Docs preview'

      - name: Create or update comment
        uses: peter-evans/create-or-update-comment@v4
        with:
          comment-id: ${{ steps.find-comment.outputs.comment-id }}
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            ## Docs preview

            Build succeeded. Download the artifact from the [Actions run](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}) to preview locally.

            To preview: download the artifact, extract it, and serve with `npx serve build`.
          edit-mode: replace
```

### Monitoring deployment health

Add a badge to your README to surface deployment status:

```markdown
[![Docs](https://github.com/my-org/my-docs/actions/workflows/docs.yml/badge.svg)](https://github.com/my-org/my-docs/actions/workflows/docs.yml)
```

## Why It Works

This pipeline catches problems at three levels before they reach production. First, the build step fails on broken Markdown, missing assets, and configuration errors. Second, the link checker catches dead internal and external links. Third, the prose linter enforces style guide rules. Only after all three pass does the deploy step run. The `concurrency` setting ensures that rapid successive pushes do not create conflicting deployments, and the `paths` filter avoids running the pipeline for unrelated changes.

## Context

The Docusaurus workflow uses the `actions/deploy-pages` action, which requires the GitHub Pages source to be set to "GitHub Actions" in repository settings. The MkDocs workflow uses `mkdocs gh-deploy`, which pushes to a `gh-pages` branch. These are different deployment mechanisms so do not mix them. The link checker excludes common social media sites that aggressively block automated requests. Add additional exclusions for URLs that consistently fail due to rate limiting or authentication requirements. The `cancel-in-progress: true` setting in the concurrency group stops running workflows when a new commit is pushed to the same branch, saving Actions minutes.
