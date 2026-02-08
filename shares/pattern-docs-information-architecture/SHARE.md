---
title: "Information Architecture for Developer Documentation Using Diataxis"
slug: "pattern-docs-information-architecture"
solution_type: pattern
tags:
  - information-architecture
  - navigation
  - taxonomy
  - ux
created: "2026-02-08"
---

## Problem

Developer documentation becomes an unnavigable maze because content types are mixed together with no consistent organizational principle.

## Solution

Apply the Diataxis framework to classify every page into one of four content types -- tutorials, how-to guides, reference, and explanation -- and build your sidebar, navigation, and content strategy around this taxonomy.

### 1. The Diataxis Framework

Diataxis organizes documentation along two axes: **learning vs. doing** and **practical vs. theoretical**.

```
                 LEARNING                    DOING
            +-----------------+    +-----------------+
 PRACTICAL  |    Tutorials    |    |  How-to Guides  |
            | "Learning-      |    | "Problem-       |
            |  oriented"      |    |  oriented"      |
            +-----------------+    +-----------------+
THEORETICAL |   Explanation   |    |    Reference    |
            | "Understanding- |    | "Information-   |
            |  oriented"      |    |  oriented"      |
            +-----------------+    +-----------------+
```

Each type has a distinct purpose and writing style:

| Type | Purpose | Structure | Example |
|------|---------|-----------|---------|
| **Tutorial** | Teach a beginner by doing | Step-by-step, always works | "Build your first app" |
| **How-to** | Solve a specific problem | Goal-oriented steps | "How to configure SSO" |
| **Reference** | Describe the machinery | Exhaustive, accurate, dry | "API endpoint reference" |
| **Explanation** | Deepen understanding | Discursive, contextual | "How authentication works" |

### 2. Concrete Directory Structure

```
docs/
  index.md                          # Landing page with paths to each quadrant
  getting-started/
    index.md                        # Quick start overview
    installation.md                 # Tutorial: install and run
    your-first-project.md           # Tutorial: end-to-end walkthrough
    your-first-deployment.md        # Tutorial: deploy to production

  guides/                           # How-to guides (task-oriented)
    authentication/
      configure-sso.md
      set-up-api-keys.md
      migrate-from-jwt-to-session.md
    deployment/
      deploy-to-aws.md
      deploy-to-vercel.md
      configure-custom-domain.md
    data/
      import-csv-data.md
      set-up-database-backups.md
      migrate-between-environments.md

  reference/                        # Reference (information-oriented)
    api/
      rest-api.md                   # Auto-generated from OpenAPI
      graphql-schema.md
      webhooks.md
    config/
      environment-variables.md
      configuration-file.md
      cli-commands.md
    sdk/
      javascript.md
      python.md

  concepts/                         # Explanation (understanding-oriented)
    architecture-overview.md
    how-authentication-works.md
    data-model.md
    permissions-and-roles.md
    event-system.md
```

### 3. Sidebar Organization

Configure the sidebar to mirror the four quadrants. Example for VitePress (`docs/.vitepress/config.ts`):

```typescript
export default defineConfig({
  themeConfig: {
    sidebar: [
      {
        text: 'Getting Started',
        icon: 'rocket',
        items: [
          { text: 'Installation', link: '/getting-started/installation' },
          { text: 'Your First Project', link: '/getting-started/your-first-project' },
          { text: 'Your First Deployment', link: '/getting-started/your-first-deployment' },
        ],
      },
      {
        text: 'Guides',
        icon: 'compass',
        collapsed: false,
        items: [
          {
            text: 'Authentication',
            collapsed: true,
            items: [
              { text: 'Configure SSO', link: '/guides/authentication/configure-sso' },
              { text: 'Set Up API Keys', link: '/guides/authentication/set-up-api-keys' },
              { text: 'Migrate from JWT to Sessions', link: '/guides/authentication/migrate-from-jwt-to-session' },
            ],
          },
          {
            text: 'Deployment',
            collapsed: true,
            items: [
              { text: 'Deploy to AWS', link: '/guides/deployment/deploy-to-aws' },
              { text: 'Deploy to Vercel', link: '/guides/deployment/deploy-to-vercel' },
              { text: 'Custom Domain', link: '/guides/deployment/configure-custom-domain' },
            ],
          },
        ],
      },
      {
        text: 'Reference',
        icon: 'book',
        collapsed: false,
        items: [
          {
            text: 'API',
            collapsed: true,
            items: [
              { text: 'REST API', link: '/reference/api/rest-api' },
              { text: 'GraphQL Schema', link: '/reference/api/graphql-schema' },
              { text: 'Webhooks', link: '/reference/api/webhooks' },
            ],
          },
          {
            text: 'Configuration',
            collapsed: true,
            items: [
              { text: 'Environment Variables', link: '/reference/config/environment-variables' },
              { text: 'Config File', link: '/reference/config/configuration-file' },
              { text: 'CLI Commands', link: '/reference/config/cli-commands' },
            ],
          },
        ],
      },
      {
        text: 'Concepts',
        icon: 'lightbulb',
        items: [
          { text: 'Architecture Overview', link: '/concepts/architecture-overview' },
          { text: 'How Authentication Works', link: '/concepts/how-authentication-works' },
          { text: 'Data Model', link: '/concepts/data-model' },
          { text: 'Permissions & Roles', link: '/concepts/permissions-and-roles' },
        ],
      },
    ],
  },
});
```

### 4. Page Classification Frontmatter

Tag every page with its Diataxis type so you can audit coverage:

```yaml
---
title: "Configure SSO"
type: how-to          # tutorial | how-to | reference | explanation
audience: developer   # developer | admin | end-user
difficulty: intermediate
last_reviewed: 2026-01-15
---
```

### 5. Audit Script for Content Coverage

```bash
#!/bin/bash
# Count pages per Diataxis type
echo "Content coverage by type:"
for TYPE in tutorial how-to reference explanation; do
  COUNT=$(grep -rl "type: $TYPE" docs/ | wc -l | tr -d ' ')
  echo "  $TYPE: $COUNT pages"
done
```

## Why It Works

The Diataxis framework eliminates the most common documentation failure mode: mixing content types on a single page (e.g., a tutorial that drifts into reference material). By giving each page a clear purpose, writers know what to include and what to leave out, and readers can predict what they will find. The sidebar mirrors the reader's intent -- "I want to learn," "I need to do X," "I need to look up Y," or "I want to understand Z" -- which reduces time-to-answer.

## Context

Diataxis was developed by Daniele Procida and is widely adopted in open-source documentation (Django, Gatsby, Cloudflare). It works best when applied consistently across the entire doc set rather than partially. Teams should classify existing pages before restructuring -- a spreadsheet audit mapping each URL to a Diataxis type often reveals gaps (e.g., "we have no explanation content") and misclassifications (e.g., "this tutorial is actually a how-to guide"). The framework is tool-agnostic and works with any static site generator or docs platform.
