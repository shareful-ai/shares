---
title: "Fix Docusaurus build failing on broken internal links"
slug: fix-docusaurus-broken-links-build-fail
tags: [docusaurus, links, build, validation]
problem: "Docusaurus build fails with broken link errors after restructuring docs folder hierarchy"
solution_type: fix
created: "2026-02-08"
---

## Problem

After moving or renaming documentation files, `docusaurus build` fails with errors like:

```
Error: Docs markdown link couldn't be resolved: ./old-page.md
Broken link found in /docs/guide/intro.md
```

The build is configured to reject broken links by default, and the existing cross-references still point to the old file paths.

## Solution

1. Find all broken references using the build output:

```bash
npx docusaurus build 2>&1 | grep "Broken link"
```

2. Fix references by updating the link targets. For markdown links, use relative paths from the current file:

```markdown
<!-- Before (broken after moving files) -->
[Setup Guide](./setup.md)

<!-- After (correct relative path) -->
[Setup Guide](../getting-started/setup.md)
```

3. To prevent future breaks, configure the broken links behavior in `docusaurus.config.js`:

```js
module.exports = {
  onBrokenLinks: 'warn',        // 'throw' | 'warn' | 'ignore'
  onBrokenMarkdownLinks: 'warn', // 'throw' | 'warn' | 'ignore'
};
```

4. Add a CI check that catches broken links before merge:

```yaml
# .github/workflows/docs.yml
- name: Check for broken links
  run: npx docusaurus build 2>&1 | tee build.log
  env:
    NODE_ENV: production
```

## Why It Works

Docusaurus resolves markdown links at build time by mapping file paths. When files move, the relative paths break. Setting `onBrokenLinks` to `warn` during development lets you iterate without hard failures, while CI enforcement catches issues before merge.

## Context

- Framework: Docusaurus 3.x
- Language: JavaScript/TypeScript
- The `onBrokenLinks` config accepts `throw`, `warn`, or `ignore`
