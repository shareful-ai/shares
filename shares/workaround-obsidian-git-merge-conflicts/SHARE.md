---
title: "Work around Obsidian vault merge conflicts in team repos"
slug: workaround-obsidian-git-merge-conflicts
tags: [obsidian, git, merge-conflicts, collaboration]
problem: "Multiple team members editing Obsidian vault files cause frequent git merge conflicts"
solution_type: workaround
created: "2026-02-08"
---

## Problem

When a team shares an Obsidian vault via git, merge conflicts arise constantly. Obsidian updates `.obsidian/workspace.json` on every UI interaction (pane positions, open files). Multiple people editing notes with wikilinks causes path-based conflicts. The `.obsidian/plugins/` directory generates conflicts when plugin settings change independently.

## Solution

1. Add a comprehensive `.gitignore` to exclude volatile files:

```gitignore
# .gitignore for shared Obsidian vaults
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache
.trash/
.DS_Store
```

2. Use a `.gitattributes` file to handle remaining conflicts gracefully:

```gitattributes
# Treat JSON config as union merge (keep both sides)
.obsidian/*.json merge=union
# Treat markdown as standard merge
*.md merge=default
```

3. Set up automatic pull-before-edit with the Obsidian Git plugin:

```json
// .obsidian/plugins/obsidian-git/data.json
{
  "autoSaveInterval": 5,
  "autoPullInterval": 5,
  "autoPullOnBoot": true,
  "disablePush": false,
  "pullBeforePush": true,
  "commitMessage": "vault: {{date}}",
  "autoCommitMessage": "vault: auto {{date}}"
}
```

4. Adopt a note-naming convention that reduces conflicts:

```
notes/
  alice/         # Each person has a personal directory
    2026-02-08-standup.md
  bob/
    2026-02-08-standup.md
  shared/        # Shared notes edited by one person at a time
    architecture-decisions.md
    team-handbook.md
```

## Why It Works

The main source of conflicts is `workspace.json`, which changes every time someone opens a pane or switches files. Ignoring it eliminates 80% of conflicts. The `merge=union` strategy for remaining JSON files keeps changes from both sides. Frequent auto-pulls reduce the window for concurrent edits, and personal directories eliminate multi-editor conflicts on daily notes.

## Context

- Tool: Obsidian with Obsidian Git plugin
- Requirement: Shared git repository (GitHub, GitLab, etc.)
- Team size: Works best for 2-10 people; larger teams should consider Confluence or Notion
