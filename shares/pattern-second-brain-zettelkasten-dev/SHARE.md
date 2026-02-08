---
title: "Developer Zettelkasten: A Practical Second Brain for Engineering"
slug: "pattern-second-brain-zettelkasten-dev"
solution_type: pattern
tags:
  - zettelkasten
  - note-taking
  - knowledge-management
  - pkm
created: "2026-02-08"
---

## Problem

Developers accumulate knowledge from debugging sessions, code reviews, and technical reading but lack a system to retain and retrieve it when the same problem recurs months later.

## Solution

Implement a Zettelkasten (slip-box) method adapted for software engineering: write atomic notes, link them densely, organize them through Maps of Content (MOCs), and use a consistent directory structure that works with Obsidian or plain markdown tooling.

### 1. Directory Structure

```
zettelkasten/
  inbox/                    # Quick captures, unprocessed
    2026-02-08-redis-memory-issue.md
    2026-02-07-interesting-cqrs-talk.md

  notes/                    # Permanent, atomic notes
    2026-02-08-1201-bm25-scoring.md
    2026-02-07-0945-connection-pooling-tradeoffs.md
    2026-02-05-1430-react-suspense-error-boundaries.md
    2026-01-28-1015-postgres-gin-index-jsonb.md

  mocs/                     # Maps of Content (index notes)
    databases.md
    distributed-systems.md
    frontend-architecture.md
    debugging-techniques.md
    system-design-interviews.md

  projects/                 # Project-specific notes (time-bound)
    acme-auth-migration/
      overview.md
      decisions.md
      retrospective.md

  templates/
    note.md
    moc.md
    project.md

  attachments/              # Images, diagrams, PDFs
    bm25-formula.png
```

### 2. Atomic Note Template

Create `templates/note.md`:

```markdown
---
id: {{date:YYYY-MM-DD}}-{{time:HHmm}}
title: ""
tags: []
created: {{date:YYYY-MM-DD}}
source: ""
---

# {{title}}

<!-- One idea per note. If you're writing more than ~300 words,
     split into multiple notes and link them. -->



## Links

- Related: [[]]
- Source: []
- MOC: [[]]
```

A real example note (`notes/2026-02-08-1201-bm25-scoring.md`):

```markdown
---
id: 2026-02-08-1201
title: "BM25 scoring penalizes long documents via length normalization"
tags: [search, ranking, algorithms]
created: 2026-02-08
source: "https://en.wikipedia.org/wiki/Okapi_BM25"
---

# BM25 scoring penalizes long documents via length normalization

The `b` parameter in BM25 controls document length normalization. At `b=1`,
a document twice the average length needs twice the term frequency to achieve
the same score. At `b=0`, length is ignored entirely.

For developer documentation, I've found `b=0.75` (the default) works well
because doc pages vary significantly in length. A 200-word API reference
and a 3000-word tutorial should not be treated the same.

The key formula:

```
score = IDF * (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * dl/avgdl))
```

Where `dl` is document length and `avgdl` is average document length.

**Practical implication**: If search results favor long pages, lower `b` toward 0.5.
If short pages dominate unfairly, raise `b` toward 1.0.

## Links

- Related: [[2026-01-15-0930-elasticsearch-field-boosting]]
- Related: [[2026-02-01-1045-meilisearch-ranking-rules]]
- Contradicts: [[2025-12-10-1400-tfidf-vs-bm25]] (TF-IDF has no length normalization)
- MOC: [[databases]]
```

### 3. Maps of Content (MOCs)

MOCs are index notes that organize related atomic notes into a navigable structure. They replace rigid folder hierarchies with flexible, overlapping categorizations.

Create `mocs/databases.md`:

```markdown
---
title: "Databases"
type: moc
updated: 2026-02-08
---

# Databases

## Search & Ranking
- [[2026-02-08-1201-bm25-scoring]] - BM25 length normalization
- [[2026-01-15-0930-elasticsearch-field-boosting]] - Field-weighted search
- [[2026-02-01-1045-meilisearch-ranking-rules]] - Meilisearch ranking config

## PostgreSQL
- [[2026-01-28-1015-postgres-gin-index-jsonb]] - GIN indexes for JSONB queries
- [[2025-11-20-0900-postgres-advisory-locks]] - Advisory locks for job queues
- [[2025-10-05-1130-postgres-explain-analyze]] - Reading EXPLAIN ANALYZE output

## Connection Management
- [[2026-02-07-0945-connection-pooling-tradeoffs]] - PgBouncer vs application-level pooling
- [[2025-09-15-1400-database-connection-limits]] - Calculating max connections

## Data Modeling
- [[2025-08-22-1100-soft-deletes-tradeoffs]] - When soft deletes cause more problems
- [[2025-07-10-0830-jsonb-vs-normalized-columns]] - When to use JSONB vs separate tables

## Unsorted
<!-- New database notes go here until I find the right section -->
```

### 4. Linking Conventions

Follow these linking rules for a well-connected knowledge graph:

| Link Type | Syntax | When to Use |
|-----------|--------|-------------|
| Direct reference | `[[note-id]]` | This note directly relates to that note |
| Context link | `Related: [[note-id]]` | Loosely related, useful for exploration |
| Source link | `Source: [title](url)` | External resource this note is based on |
| Contradiction | `Contradicts: [[note-id]]` | Notes that present opposing views |
| Supports | `Supports: [[note-id]]` | Evidence or example that reinforces another note |
| MOC backlink | `MOC: [[moc-name]]` | Which index this note belongs to |

The goal is **minimum 2 links per note** -- one to a related note and one to a MOC. Orphan notes (zero links) are nearly useless because you will never rediscover them.

### 5. Capture Workflow

The workflow has three stages: capture, process, and connect.

**Stage 1: Capture (inbox)** -- Quick and dirty, do not overthink it:

```markdown
---
title: "Redis memory spike in prod"
captured: 2026-02-08
source: "debugging session"
---

Redis memory jumped 3x. Turned out we were storing full user objects as
session data instead of just session IDs. `redis-cli --bigkeys` showed
the session keyspace was 80% of total memory. Fixed by storing only
session ID and looking up user data on each request. Latency impact
was negligible (~2ms) because user data was in PG with connection pooling.
```

**Stage 2: Process** -- Refine into an atomic note with a clear title:

```markdown
---
id: 2026-02-08-1430
title: "Store only session IDs in Redis, not full user objects"
tags: [redis, sessions, performance, debugging]
created: 2026-02-08
source: "production incident 2026-02-08"
---

# Store only session IDs in Redis, not full user objects

Storing full user objects in Redis sessions caused a 3x memory spike.
Session data grew to 80% of Redis memory (`redis-cli --bigkeys` to diagnose).

**Fix**: Store only the session ID. Look up user data from PostgreSQL on each
request. With connection pooling, the added latency was ~2ms -- negligible.

**Diagnostic command**: `redis-cli --bigkeys` scans all keys and reports the
largest ones per data type.

**Rule of thumb**: Redis session values should be < 1KB. If larger, you're
probably caching data that belongs in the database.

## Links

- Related: [[2026-02-07-0945-connection-pooling-tradeoffs]]
- Related: [[2025-11-03-1200-redis-memory-management]]
- MOC: [[debugging-techniques]]
```

**Stage 3: Connect** -- Add backlinks from related notes and update relevant MOCs.

### 6. Obsidian Configuration

If using Obsidian, create `.obsidian/templates/note.md` with the template from section 2, and configure settings:

```json
// .obsidian/app.json (relevant settings)
{
  "defaultViewMode": "source",
  "newFileLocation": "folder",
  "newFileFolderPath": "inbox",
  "attachmentFolderPath": "attachments"
}
```

Recommended plugins for developer Zettelkasten:
- **Templater**: For dynamic template variables (date, time)
- **Dataview**: Query notes like a database (`TABLE tags FROM "notes" WHERE contains(tags, "redis")`)
- **Graph View**: Built-in, visualize connections
- **Quick Switcher++**: Fast navigation by title or tag

Dataview query example for finding notes that need links:

```dataview
TABLE title, length(file.outlinks) as "Outlinks"
FROM "notes"
WHERE length(file.outlinks) < 2
SORT file.ctime DESC
```

### 7. Plain Markdown Alternative (No Obsidian)

For developers who prefer plain tools, use standard markdown links and a simple search:

```bash
# Find all notes linking to a specific note
grep -rl "connection-pooling-tradeoffs" zettelkasten/

# Find orphan notes (no outgoing links)
for f in zettelkasten/notes/*.md; do
  LINKS=$(grep -c '\[\[' "$f" || true)
  if [ "$LINKS" -eq 0 ]; then
    echo "Orphan: $f"
  fi
done

# Search notes by tag
grep -rl "tags:.*redis" zettelkasten/notes/
```

## Why It Works

The Zettelkasten method works for developers because it mirrors how technical knowledge actually connects -- a Redis memory issue links to connection pooling, which links to PostgreSQL configuration, which links to deployment architecture. Atomic notes force you to distill insights to their essence, making them reusable across contexts. The processing workflow (capture -> refine -> connect) separates the urgency of capturing information from the slower work of organizing it. MOCs provide just enough structure to navigate without imposing a rigid taxonomy that breaks down as knowledge grows.

## Context

This pattern is adapted from Niklas Luhmann's original Zettelkasten method for a software engineering context. It works with Obsidian (recommended for the graph view and Dataview plugin), Logseq, Notion, or plain markdown files in any editor. The key is consistency: the system only works if you process inbox notes regularly (weekly at minimum) and maintain links. A Zettelkasten with 50 well-linked notes is more valuable than 500 unlinked notes. Start small -- capture debugging insights and technical decisions from your daily work, and the graph will grow naturally.
