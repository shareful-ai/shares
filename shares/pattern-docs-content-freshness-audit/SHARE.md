---
title: "Automated Content Freshness Auditing for Documentation"
slug: "pattern-docs-content-freshness-audit"
solution_type: pattern
tags:
  - content-audit
  - freshness
  - stale-docs
  - automation
created: "2026-02-08"
---

## Problem

Documentation silently becomes outdated because there is no systematic way to detect which pages have not been reviewed since the code they describe changed.

## Solution

Combine Git history analysis with frontmatter metadata to automatically identify stale documentation pages, generate audit reports, and integrate freshness checks into CI.

### 1. Frontmatter `last_reviewed` Field

Add a `last_reviewed` date to every documentation page's frontmatter. This date represents the last time a human confirmed the content is accurate, which is distinct from the last Git commit (which could be a typo fix):

```yaml
---
title: "Configure Database Connections"
last_reviewed: "2026-01-15"
review_owner: "@backend-team"
---
```

Update `last_reviewed` only when a substantive accuracy review occurs, not on every edit.

### 2. Freshness Audit Script

Create `scripts/docs-freshness-audit.sh`:

```bash
#!/bin/bash
set -euo pipefail

DOCS_DIR="${1:-docs}"
STALE_DAYS="${2:-90}"
CUTOFF_DATE=$(date -v-${STALE_DAYS}d +%Y-%m-%d 2>/dev/null || date -d "$STALE_DAYS days ago" +%Y-%m-%d)

echo "Documentation Freshness Audit"
echo "=============================="
echo "Stale threshold: $STALE_DAYS days (before $CUTOFF_DATE)"
echo ""

TOTAL=0
STALE=0
NO_REVIEW=0
FRESH=0

# CSV output for reporting
REPORT_FILE="docs-freshness-report.csv"
echo "file,last_reviewed,last_git_commit,review_owner,status" > "$REPORT_FILE"

for FILE in $(find "$DOCS_DIR" -name "*.md" -type f | sort); do
  TOTAL=$((TOTAL + 1))

  # Extract last_reviewed from frontmatter
  LAST_REVIEWED=$(sed -n '/^---$/,/^---$/p' "$FILE" | grep 'last_reviewed' | head -1 | sed 's/.*: *"\{0,1\}\([0-9-]*\)"\{0,1\}/\1/')

  # Extract review_owner from frontmatter
  REVIEW_OWNER=$(sed -n '/^---$/,/^---$/p' "$FILE" | grep 'review_owner' | head -1 | sed 's/.*: *"\{0,1\}\(.*\)"\{0,1\}/\1/')

  # Get last Git modification date
  LAST_GIT_COMMIT=$(git log -1 --format="%Y-%m-%d" -- "$FILE" 2>/dev/null || echo "unknown")

  if [ -z "$LAST_REVIEWED" ]; then
    STATUS="MISSING_REVIEW_DATE"
    NO_REVIEW=$((NO_REVIEW + 1))
    echo "  [NO DATE]  $FILE (last commit: $LAST_GIT_COMMIT)"
  elif [[ "$LAST_REVIEWED" < "$CUTOFF_DATE" ]]; then
    STATUS="STALE"
    STALE=$((STALE + 1))
    DAYS_AGO=$(( ($(date +%s) - $(date -j -f "%Y-%m-%d" "$LAST_REVIEWED" +%s 2>/dev/null || date -d "$LAST_REVIEWED" +%s)) / 86400 ))
    echo "  [STALE]    $FILE (reviewed: $LAST_REVIEWED, ${DAYS_AGO}d ago, owner: ${REVIEW_OWNER:-unassigned})"
  else
    STATUS="FRESH"
    FRESH=$((FRESH + 1))
  fi

  echo "$FILE,$LAST_REVIEWED,$LAST_GIT_COMMIT,${REVIEW_OWNER:-unassigned},$STATUS" >> "$REPORT_FILE"
done

echo ""
echo "Summary"
echo "-------"
echo "  Total pages:       $TOTAL"
echo "  Fresh:             $FRESH"
echo "  Stale (>$STALE_DAYS days): $STALE"
echo "  Missing date:      $NO_REVIEW"
echo ""
echo "Report written to: $REPORT_FILE"

# Exit with error if stale percentage exceeds threshold
STALE_COMBINED=$((STALE + NO_REVIEW))
if [ "$TOTAL" -gt 0 ]; then
  STALE_PCT=$((STALE_COMBINED * 100 / TOTAL))
  echo "  Stale + undated:   ${STALE_PCT}%"
  if [ "$STALE_PCT" -gt 30 ]; then
    echo ""
    echo "WARNING: ${STALE_PCT}% of docs are stale or unreviewed (threshold: 30%)"
    exit 1
  fi
fi
```

### 3. Node.js Audit Script (Cross-Platform)

For teams that prefer a cross-platform solution, `scripts/docs-freshness-audit.mjs`:

```javascript
import { readdir, readFile } from 'fs/promises';
import { join, relative } from 'path';
import { execSync } from 'child_process';

const DOCS_DIR = process.argv[2] || 'docs';
const STALE_DAYS = parseInt(process.argv[3] || '90', 10);

async function* walkMarkdown(dir) {
  const entries = await readdir(dir, { withFileTypes: true });
  for (const entry of entries) {
    const fullPath = join(dir, entry.name);
    if (entry.isDirectory()) yield* walkMarkdown(fullPath);
    else if (entry.name.endsWith('.md')) yield fullPath;
  }
}

function extractFrontmatter(content) {
  const match = content.match(/^---\n([\s\S]*?)\n---/);
  if (!match) return {};
  const fields = {};
  for (const line of match[1].split('\n')) {
    const [key, ...rest] = line.split(':');
    if (key && rest.length) {
      fields[key.trim()] = rest.join(':').trim().replace(/^["']|["']$/g, '');
    }
  }
  return fields;
}

function getGitLastModified(filePath) {
  try {
    return execSync(`git log -1 --format="%Y-%m-%d" -- "${filePath}"`, {
      encoding: 'utf-8',
    }).trim();
  } catch {
    return null;
  }
}

function daysBetween(dateStr) {
  return Math.floor((Date.now() - new Date(dateStr).getTime()) / 86_400_000);
}

const results = { fresh: [], stale: [], missing: [] };

for await (const filePath of walkMarkdown(DOCS_DIR)) {
  const content = await readFile(filePath, 'utf-8');
  const fm = extractFrontmatter(content);
  const gitDate = getGitLastModified(filePath);
  const relPath = relative('.', filePath);

  const entry = {
    file: relPath,
    lastReviewed: fm.last_reviewed || null,
    lastGitCommit: gitDate,
    owner: fm.review_owner || 'unassigned',
  };

  if (!fm.last_reviewed) {
    results.missing.push(entry);
  } else if (daysBetween(fm.last_reviewed) > STALE_DAYS) {
    entry.daysAgo = daysBetween(fm.last_reviewed);
    results.stale.push(entry);
  } else {
    results.fresh.push(entry);
  }
}

// Print report
console.log('\nDocumentation Freshness Audit');
console.log('==============================');
console.log(`Stale threshold: ${STALE_DAYS} days\n`);

if (results.stale.length) {
  console.log('STALE PAGES:');
  results.stale
    .sort((a, b) => b.daysAgo - a.daysAgo)
    .forEach((r) => console.log(`  [${r.daysAgo}d] ${r.file} (owner: ${r.owner})`));
}

if (results.missing.length) {
  console.log('\nMISSING REVIEW DATE:');
  results.missing.forEach((r) => console.log(`  ${r.file}`));
}

const total = results.fresh.length + results.stale.length + results.missing.length;
const staleCount = results.stale.length + results.missing.length;
console.log(`\nSummary: ${results.fresh.length} fresh, ${results.stale.length} stale, ${results.missing.length} missing date (${total} total)`);

if (total > 0 && (staleCount / total) > 0.3) {
  console.error(`\nERROR: ${Math.round((staleCount / total) * 100)}% of docs need review.`);
  process.exit(1);
}
```

### 4. CI Integration

Add a weekly freshness check in `.github/workflows/docs-freshness.yml`:

```yaml
name: Docs Freshness Audit

on:
  schedule:
    - cron: "0 9 * * 1"  # Every Monday at 9am UTC
  workflow_dispatch:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history needed for git log dates

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run freshness audit
        run: node scripts/docs-freshness-audit.mjs docs 90

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: freshness-report
          path: docs-freshness-report.csv

      - name: Create issue if stale docs found
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('docs-freshness-report.csv', 'utf-8');
            const staleLines = report.split('\n').filter(l => l.includes('STALE') || l.includes('MISSING'));
            const body = `## Docs Freshness Audit Failed\n\n${staleLines.length} pages need review.\n\n\`\`\`\n${staleLines.join('\n')}\n\`\`\``;
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Docs freshness audit: ${staleLines.length} pages need review`,
              body,
              labels: ['documentation', 'maintenance'],
            });
```

### 5. Review Workflow

When a page is reviewed and confirmed accurate, update the frontmatter:

```bash
# Quick command to mark a page as reviewed today
sed -i '' "s/last_reviewed: .*/last_reviewed: \"$(date +%Y-%m-%d)\"/" docs/guides/configure-sso.md
```

Integrate this into your team's sprint process: assign 5-10 stale pages per sprint for review, rotating ownership so no single person bears the full burden.

## Why It Works

Freshness auditing makes documentation decay visible and actionable. Without measurement, stale docs accumulate silently until a user reports an error. The `last_reviewed` field in frontmatter is critical because it separates "someone touched this file" (Git history) from "someone verified this is accurate" (human review). The CI integration ensures the audit runs automatically, and creating GitHub issues from failures routes stale pages into the team's existing work tracking system.

## Context

This pattern works for any documentation stored in Git with markdown frontmatter. The 90-day stale threshold is a reasonable default for actively developed products; adjust based on your release cadence. Stable, rarely-changing reference documentation might use a 180-day threshold, while API docs for rapidly evolving endpoints might use 30 days. The scripts assume a Unix-like environment; the Node.js version provides cross-platform compatibility.
