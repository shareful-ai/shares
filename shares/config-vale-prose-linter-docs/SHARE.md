---
title: "Configure Vale prose linter for technical documentation CI"
slug: config-vale-prose-linter-docs
tags: [vale, linting, prose, ci]
problem: "Documentation has inconsistent style and readability because there is no automated prose linting"
solution_type: config
created: "2026-02-08"
---

## Problem

Code gets linted. Prose does not. The result is documentation with passive voice in one section and active voice in the next, jargon that confuses new users, sentences that run to 60 words, and inconsistent heading styles across pages. Manual review catches some of these issues, but reviewers are inconsistent and tire quickly. Without automated prose linting, style guide rules exist on paper but not in practice.

## Solution

Use Vale, an open source prose linter, to enforce style rules in CI. The configuration below includes the `.vale.ini` config, custom vocabulary, example rules, and a GitHub Actions workflow.

### .vale.ini

```ini title=".vale.ini"
# ==============================================================================
# Vale configuration
# https://vale.sh/docs/reference/config
# ==============================================================================

StylesPath = .vale/styles
MinAlertLevel = suggestion

# Use the Google and Microsoft style packages as a base
Packages = Google, Microsoft

# Vocabulary defines accepted terms and rejected terms
Vocab = MyProject

# ==============================================================================
# Format-specific settings
# ==============================================================================

# Markdown files in the docs directory
[docs/**.md]
BasedOnStyles = Vale, Google, Microsoft, MyProject
# Disable specific rules that conflict with our style
Google.Parens = NO
Google.Spacing = NO
Microsoft.Adverbs = suggestion

# Blog posts (slightly relaxed rules)
[blog/**.md]
BasedOnStyles = Vale, Google
Vale.Spelling = NO

# Changelogs (minimal linting)
[CHANGELOG.md]
BasedOnStyles = Vale
```

### Custom vocabulary

Create accepted and rejected term lists:

```txt title=".vale/styles/Vocab/MyProject/accept.txt"
Docusaurus
MkDocs
GitHub
TypeScript
JavaScript
PostgreSQL
Redis
OAuth
API
APIs
CLI
SDK
YAML
JSON
frontmatter
```

```txt title=".vale/styles/Vocab/MyProject/reject.txt"
utilise
leverage
[Bb]asically
[Ss]imply
[Jj]ust # often a filler word in docs
[Oo]bviously
[Ee]asily
```

### Custom style rules

#### Passive voice detection

```yaml title=".vale/styles/MyProject/PassiveVoice.yml"
extends: existence
message: "Avoid passive voice. Rewrite '%s' in active voice."
level: warning
ignorecase: true
tokens:
  - 'is (\w+ed|built|made|given|known|seen|done)'
  - 'are (\w+ed|built|made|given|known|seen|done)'
  - 'was (\w+ed|built|made|given|known|seen|done)'
  - 'were (\w+ed|built|made|given|known|seen|done)'
  - 'been (\w+ed|built|made|given|known|seen|done)'
  - 'being (\w+ed|built|made|given|known|seen|done)'
```

#### Sentence length

```yaml title=".vale/styles/MyProject/SentenceLength.yml"
extends: metric
message: "Sentence too long (%s words). Aim for under 25 words."
level: warning
metric: words
condition: ">"
threshold: 30
scope: sentence
```

#### Heading style (sentence case)

```yaml title=".vale/styles/MyProject/HeadingCase.yml"
extends: capitalization
message: "Use sentence case for headings: '%s'."
level: error
match: $sentence
scope: heading
indicators:
  - ":"
exceptions:
  - API
  - APIs
  - CLI
  - URL
  - OAuth
  - GitHub
  - JavaScript
  - TypeScript
  - PostgreSQL
  - MkDocs
  - Docusaurus
```

#### Inclusive language

```yaml title=".vale/styles/MyProject/InclusiveLanguage.yml"
extends: substitution
message: "Use inclusive language. Replace '%s' with '%s'."
level: error
ignorecase: true
swap:
  whitelist: allowlist
  blacklist: blocklist
  master: main
  slave: replica
  sanity check: confidence check
  dummy: placeholder
  grandfathered: legacy
  man-hours: person-hours
  manpower: workforce
```

#### No jargon without explanation

```yaml title=".vale/styles/MyProject/Jargon.yml"
extends: existence
message: "Avoid jargon without explanation: '%s'. Define the term or use a simpler alternative."
level: suggestion
ignorecase: true
tokens:
  - 'idempotent'
  - 'orthogonal'
  - 'reify'
  - 'hydrate'
  - 'serialize'
  - 'marshalling'
  - 'bikeshed'
  - 'greenfield'
  - 'brownfield'
  - 'yak shaving'
```

### GitHub Actions workflow

```yaml title=".github/workflows/prose-lint.yml"
name: Prose lint

on:
  pull_request:
    paths:
      - 'docs/**'
      - 'blog/**'
      - '*.md'
      - '.vale.ini'
      - '.vale/**'

permissions:
  contents: read
  pull-requests: write

jobs:
  vale:
    name: Lint prose with Vale
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install Vale
        uses: errata-ai/vale-action@v2
        with:
          version: '3.9.1'
          files: docs
          reporter: github-pr-review
          fail_on_error: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Local development

```sh
# Install Vale
brew install vale   # macOS
# or: snap install vale  # Linux

# Sync style packages (Google, Microsoft)
vale sync

# Lint all docs
vale docs/

# Lint a single file
vale docs/getting-started/installation.md

# Lint with JSON output for tooling integration
vale --output JSON docs/
```

### Editor integration

Vale integrates with most editors for real-time feedback:

- **VS Code**: Install the "Vale VSCode" extension by Chris Chinchilla. Set `vale.valeCLI.path` to your Vale binary.
- **Neovim**: Use `null-ls` or `nvim-lint` with the Vale source.
- **Sublime Text**: Use the SublimeLinter-vale plugin.

## Why It Works

Automated prose linting catches style issues at the same point in the workflow where code linting catches code issues: before the PR is merged. This removes the burden from human reviewers, who can focus on technical accuracy and clarity instead of spotting passive voice and inconsistent capitalization. The combination of established style packages (Google, Microsoft) with custom rules means you get broad coverage out of the box and can layer on project-specific requirements incrementally.

## Context

Vale style packages (Google, Microsoft) are community-maintained and update periodically. Run `vale sync` after updating `.vale.ini` to fetch the latest rules. The `errata-ai/vale-action` GitHub Action uses the `github-pr-review` reporter to post inline comments on the PR diff, which gives authors precise feedback. If you use a different CI system, run `vale --output line docs/` and parse the output. The `fail_on_error: true` setting blocks merge when errors are found; set to `false` during initial rollout to avoid blocking the entire team while you tune the rules.
