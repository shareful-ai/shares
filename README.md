# shares

A [shareful.ai](https://shareful.ai) shares repository. Use this template to start sharing AI coding solutions.

## Quick start

```bash
# Clone this template
gh repo create my-shares --template shareful-ai/shares --public --clone
cd my-shares

# Create and publish a share
npx shareful-ai create
```

## Structure

```
shares/
  fix-example-issue/
    SHARE.md
  pattern-example-approach/
    SHARE.md
```

Each `SHARE.md` contains a solution with YAML frontmatter and four required sections:

- **Problem** -- what went wrong
- **Solution** -- how to fix it (with code)
- **Why It Works** -- the explanation
- **Context** -- versions, frameworks, constraints

## Solution types

| Type | Use when |
|------|----------|
| `fix` | Direct resolution for a bug or error |
| `workaround` | Temporary bypass for a known issue |
| `pattern` | Reusable approach or best practice |
| `reference` | Guide, cheat sheet, or comparison |
| `config` | Configuration template |

## Slug conventions

Name share directories with the solution type as prefix:

```
fix-nextjs-hydration-error
workaround-notion-api-export
pattern-docs-as-code-workflow
reference-docs-platform-comparison
config-docusaurus-full-setup
```

## Learn more

- [shareful.ai](https://shareful.ai) -- search all shared solutions
- `npx shareful-ai --help` -- CLI reference
