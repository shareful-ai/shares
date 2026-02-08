---
title: "Colocating Documentation with Code in Monorepos"
slug: "pattern-monorepo-docs-colocation"
solution_type: pattern
tags:
  - monorepo
  - colocation
  - organization
  - docs-as-code
created: "2026-02-08"
---

## Problem

In monorepos, documentation either lives in a single top-level folder divorced from the packages it describes, or is scattered across packages with no unified way to browse it.

## Solution

Colocate documentation alongside the code it describes within each package, then use a build step to aggregate all package docs into a single, navigable documentation site.

### 1. Monorepo Directory Structure

Each package owns its own `docs/` folder. A top-level `docs-site/` project aggregates everything at build time:

```
monorepo/
  packages/
    auth/
      src/
      docs/
        overview.md
        guides/
          configure-sso.md
          api-keys.md
        reference/
          auth-api.md
      package.json
      README.md

    billing/
      src/
      docs/
        overview.md
        guides/
          setup-stripe.md
          invoicing.md
        reference/
          billing-api.md
      package.json

    ui-components/
      src/
      docs/
        overview.md
        guides/
          theming.md
          custom-components.md
        reference/
          component-props.md
      package.json

  docs-site/                # Aggregated documentation site
    .vitepress/
      config.ts
      theme/
    public/
    index.md                # Landing page
    architecture.md         # Cross-cutting docs live here
    package.json

  scripts/
    aggregate-docs.sh
  turbo.json
  package.json
```

### 2. Documentation Aggregation Script

Create `scripts/aggregate-docs.sh` to symlink or copy package docs into the docs site:

```bash
#!/bin/bash
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel)"
DOCS_SITE="$REPO_ROOT/docs-site"
PACKAGES_DIR="$REPO_ROOT/packages"

echo "Aggregating docs from packages into docs-site..."

# Clean previous aggregated docs (only the package-specific dirs)
for PACKAGE_DIR in "$PACKAGES_DIR"/*/; do
  PACKAGE_NAME=$(basename "$PACKAGE_DIR")
  rm -rf "$DOCS_SITE/packages/$PACKAGE_NAME"
done

mkdir -p "$DOCS_SITE/packages"

# Copy docs from each package that has a docs/ folder
for PACKAGE_DIR in "$PACKAGES_DIR"/*/; do
  PACKAGE_NAME=$(basename "$PACKAGE_DIR")
  PACKAGE_DOCS="$PACKAGE_DIR/docs"

  if [ -d "$PACKAGE_DOCS" ]; then
    echo "  Aggregating: $PACKAGE_NAME"
    cp -r "$PACKAGE_DOCS" "$DOCS_SITE/packages/$PACKAGE_NAME"

    # Rewrite relative image paths to work from the new location
    find "$DOCS_SITE/packages/$PACKAGE_NAME" -name "*.md" -exec \
      sed -i '' "s|](\.\.\/|](\/packages\/$PACKAGE_NAME\/|g" {} \;
  else
    echo "  Skipping:    $PACKAGE_NAME (no docs/ folder)"
  fi
done

echo "Done. Aggregated docs are in $DOCS_SITE/packages/"
```

### 3. Node.js Aggregation with Sidebar Generation

For a more robust solution that also auto-generates the sidebar, create `scripts/aggregate-docs.mjs`:

```javascript
import { cpSync, existsSync, mkdirSync, readdirSync, readFileSync, writeFileSync, rmSync } from 'fs';
import { join, basename } from 'path';

const ROOT = process.cwd();
const PACKAGES_DIR = join(ROOT, 'packages');
const DOCS_SITE = join(ROOT, 'docs-site');
const OUTPUT_DIR = join(DOCS_SITE, 'packages');

// Clean and recreate
if (existsSync(OUTPUT_DIR)) rmSync(OUTPUT_DIR, { recursive: true });
mkdirSync(OUTPUT_DIR, { recursive: true });

const sidebarItems = [];

for (const pkg of readdirSync(PACKAGES_DIR).sort()) {
  const pkgDocsDir = join(PACKAGES_DIR, pkg, 'docs');
  if (!existsSync(pkgDocsDir)) continue;

  console.log(`Aggregating: ${pkg}`);
  const destDir = join(OUTPUT_DIR, pkg);
  cpSync(pkgDocsDir, destDir, { recursive: true });

  // Read package.json for display name
  const pkgJson = JSON.parse(readFileSync(join(PACKAGES_DIR, pkg, 'package.json'), 'utf-8'));
  const displayName = pkgJson.displayName || pkg.charAt(0).toUpperCase() + pkg.slice(1);

  // Build sidebar section for this package
  const items = buildSidebarItems(destDir, `/packages/${pkg}`);
  sidebarItems.push({
    text: displayName,
    collapsed: true,
    items,
  });
}

function buildSidebarItems(dir, urlPrefix) {
  const items = [];
  const entries = readdirSync(dir, { withFileTypes: true }).sort();

  // Files first
  for (const entry of entries) {
    if (entry.isFile() && entry.name.endsWith('.md')) {
      const slug = entry.name.replace('.md', '');
      const content = readFileSync(join(dir, entry.name), 'utf-8');
      const titleMatch = content.match(/^#\s+(.+)$/m) || content.match(/title:\s*["']?(.+?)["']?\s*$/m);
      const title = titleMatch ? titleMatch[1] : slug;

      items.push({
        text: title,
        link: `${urlPrefix}/${slug}`,
      });
    }
  }

  // Subdirectories
  for (const entry of entries) {
    if (entry.isDirectory()) {
      const subItems = buildSidebarItems(join(dir, entry.name), `${urlPrefix}/${entry.name}`);
      if (subItems.length > 0) {
        items.push({
          text: entry.name.charAt(0).toUpperCase() + entry.name.slice(1),
          collapsed: true,
          items: subItems,
        });
      }
    }
  }

  return items;
}

// Write generated sidebar config
const sidebarConfig = JSON.stringify(sidebarItems, null, 2);
const outputPath = join(DOCS_SITE, '.vitepress', 'sidebar-packages.json');
writeFileSync(outputPath, sidebarConfig);
console.log(`\nSidebar config written to: ${outputPath}`);
console.log(`Aggregated ${sidebarItems.length} packages.`);
```

### 4. VitePress Configuration

Wire the auto-generated sidebar into `docs-site/.vitepress/config.ts`:

```typescript
import { defineConfig } from 'vitepress';
import packagesSidebar from './sidebar-packages.json';

export default defineConfig({
  title: 'Acme Platform Docs',
  description: 'Documentation for all Acme platform packages',

  themeConfig: {
    sidebar: [
      {
        text: 'Getting Started',
        items: [
          { text: 'Overview', link: '/' },
          { text: 'Architecture', link: '/architecture' },
          { text: 'Quick Start', link: '/quick-start' },
        ],
      },
      {
        text: 'Packages',
        items: packagesSidebar,
      },
    ],

    search: {
      provider: 'local',
    },
  },
});
```

### 5. Turborepo Integration

Add the docs aggregation and build to `turbo.json`:

```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "docs:aggregate": {
      "inputs": ["packages/*/docs/**"],
      "outputs": ["docs-site/packages/**", "docs-site/.vitepress/sidebar-packages.json"]
    },
    "docs:build": {
      "dependsOn": ["docs:aggregate"],
      "inputs": ["docs-site/**"],
      "outputs": ["docs-site/.vitepress/dist/**"]
    }
  }
}
```

And in the root `package.json`:

```json
{
  "scripts": {
    "docs:aggregate": "node scripts/aggregate-docs.mjs",
    "docs:build": "npm run docs:aggregate && cd docs-site && vitepress build",
    "docs:dev": "npm run docs:aggregate && cd docs-site && vitepress dev"
  }
}
```

### 6. Package-Level Doc Ownership

Each package's `package.json` can declare documentation metadata:

```json
{
  "name": "@acme/auth",
  "displayName": "Authentication",
  "docs": {
    "category": "Core Services",
    "order": 1,
    "owner": "@backend-team"
  }
}
```

The aggregation script can read these fields to sort packages in the sidebar, group by category, and assign review ownership.

## Why It Works

Colocation ensures that documentation changes happen in the same PR as code changes, making it natural for developers to update docs when modifying package behavior. The aggregation step solves the discoverability problem -- readers get a single, unified site while authors work in the context of their package. Turborepo's input tracking means the docs site only rebuilds when docs actually change, keeping CI fast. Package ownership metadata ensures every doc page has a clear responsible team.

## Context

This pattern is designed for TypeScript/JavaScript monorepos using Turborepo or Nx, with VitePress as the documentation framework. The same approach works with other static site generators (Docusaurus, Astro) by adapting the sidebar generation step. For very large monorepos (50+ packages), consider generating a top-level index page that groups packages by domain or team rather than listing all packages in a flat sidebar. The symlink approach (`ln -s` instead of `cp -r`) is faster but can cause issues with some bundlers; the copy approach is more portable.
