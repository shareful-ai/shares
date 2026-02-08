# Shares Repository

This is a shareful.ai shares repository containing coding solutions as SHARE.md files.

## Structure

- Each share lives in `shares/{slug}/SHARE.md`
- Slugs are kebab-case, prefixed by solution type: `fix-`, `workaround-`, `pattern-`, `reference-`, `config-`
- The `shares/` directory is the only content directory

## SHARE.md format

Every SHARE.md must have:

1. YAML frontmatter with: `title`, `slug`, `tags`, `problem`, `solution_type`, `created`
2. Four required markdown sections: `## Problem`, `## Solution`, `## Why It Works`, `## Context`

### Frontmatter constraints

- `title`: max 128 characters
- `slug`: max 64 characters, lowercase, hyphens and numbers only (`[a-z0-9-]`)
- `tags`: 1-10 tags, each lowercase, max 32 characters
- `problem`: max 256 characters, one sentence
- `solution_type`: one of `fix`, `workaround`, `pattern`, `reference`, `config`
- `created`: ISO date string (YYYY-MM-DD)

### Valid solution types

- **fix** -- direct resolution for a bug or error
- **workaround** -- temporary bypass for a known issue
- **pattern** -- reusable approach or best practice
- **reference** -- guide, cheat sheet, or comparison
- **config** -- configuration template

## Commands

```bash
npx shareful create     # Create a new SHARE.md interactively
npx shareful publish    # Validate, commit, push, and index
npx shareful list       # List shares in this repo
```

## Code style

- Markdown content uses standard GitHub-flavored markdown
- Code blocks must specify a language for syntax highlighting
- Keep solutions focused and actionable
