---
title: "Architecture Decision Records (ADR) Template and Workflow"
slug: "pattern-adr-architecture-decision-records"
solution_type: pattern
tags:
  - adr
  - architecture
  - decisions
  - templates
created: "2026-02-08"
---

## Problem

Teams repeatedly revisit past architectural decisions because the reasoning behind them is lost, undocumented, or scattered across Slack threads and meeting notes.

## Solution

Adopt Architecture Decision Records (ADRs) -- short, structured documents that capture the context, decision, and consequences of each significant technical choice. Store them in the repository alongside the code they govern.

### 1. ADR Template

Create `docs/adr/template.md`:

```markdown
# ADR-NNNN: [Short Title of Decision]

## Status

[Proposed | Accepted | Deprecated | Superseded by ADR-XXXX]

## Date

YYYY-MM-DD

## Context

Describe the forces at play: technical constraints, business requirements,
team capabilities, timeline pressure. What problem are you solving? What
options did you consider?

## Decision

State the decision clearly and concisely. Use active voice:
"We will use PostgreSQL as the primary data store."

## Consequences

### Positive
- What becomes easier or better as a result of this decision.

### Negative
- What becomes harder, what trade-offs are accepted.

### Neutral
- Side effects that are neither clearly positive nor negative.

## References

- Links to relevant RFCs, documentation, benchmarks, or discussions.
```

### 2. File Naming Convention

Store ADRs in `docs/adr/` using a zero-padded sequential numbering scheme:

```
docs/adr/
  0001-use-react-for-frontend.md
  0002-adopt-postgresql-over-mongodb.md
  0003-event-driven-architecture-for-notifications.md
  0004-monorepo-with-turborepo.md
  0005-rest-over-graphql-for-public-api.md
  template.md
  README.md
```

The sequential prefix ensures ADRs sort chronologically in file explorers and makes cross-referencing unambiguous ("as decided in ADR-0002").

### 3. Decision Log Index

Create `docs/adr/README.md` as a living index:

```markdown
# Architecture Decision Log

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](0001-use-react-for-frontend.md) | Use React for frontend | Accepted | 2025-03-15 |
| [0002](0002-adopt-postgresql-over-mongodb.md) | Adopt PostgreSQL over MongoDB | Accepted | 2025-04-02 |
| [0003](0003-event-driven-architecture-for-notifications.md) | Event-driven architecture for notifications | Accepted | 2025-05-20 |
| [0004](0004-monorepo-with-turborepo.md) | Monorepo with Turborepo | Accepted | 2025-06-10 |
| [0005](0005-rest-over-graphql-for-public-api.md) | REST over GraphQL for public API | Proposed | 2026-01-28 |
```

### 4. Concrete Example ADR

`docs/adr/0002-adopt-postgresql-over-mongodb.md`:

```markdown
# ADR-0002: Adopt PostgreSQL over MongoDB

## Status

Accepted

## Date

2025-04-02

## Context

Our application needs a primary data store. We have relational data with
strong consistency requirements (user accounts, financial transactions,
access control). The team evaluated:

1. **MongoDB** -- flexible schema, good for rapid prototyping, but requires
   application-level joins and careful denormalization for our relational data.
2. **PostgreSQL** -- ACID transactions, mature ecosystem, strong typing via
   schemas, native JSON support via `jsonb` for semi-structured data.
3. **MySQL** -- viable, but PostgreSQL offers better JSON support, richer
   indexing (GIN, GiST), and extensions like PostGIS if geo features are needed.

Our data is inherently relational (users -> organizations -> projects -> tasks),
and we need transactional integrity for billing operations.

## Decision

We will use PostgreSQL 16 as the primary data store, accessed through Prisma ORM.

## Consequences

### Positive
- ACID transactions simplify billing and access control logic.
- Schema enforcement catches data integrity issues at the database level.
- The `jsonb` column type handles our semi-structured metadata without
  needing a separate document store.
- Rich ecosystem of tooling, monitoring, and managed hosting options.

### Negative
- Schema migrations require more upfront planning than schema-less approaches.
- Horizontal scaling is more complex than MongoDB's built-in sharding (mitigated
  by read replicas and connection pooling for our expected scale).

### Neutral
- Team has experience with both PostgreSQL and MongoDB, so no significant
  ramp-up time either way.

## References

- [PostgreSQL vs MongoDB comparison](https://www.prisma.io/dataguide/postgresql/postgresql-vs-mongodb)
- Internal spike: `docs/spikes/2025-03-datastore-evaluation.md`
- Team discussion: Slack thread #eng-architecture, 2025-03-28
```

### 5. CLI Helper Script

Create a script to scaffold new ADRs at `scripts/new-adr.sh`:

```bash
#!/bin/bash
set -euo pipefail

ADR_DIR="docs/adr"
TEMPLATE="$ADR_DIR/template.md"

# Find the next ADR number
LAST=$(ls "$ADR_DIR" | grep -E '^[0-9]{4}-' | sort -r | head -1 | cut -d'-' -f1)
NEXT=$(printf "%04d" $((10#${LAST:-0} + 1)))

# Slugify the title
TITLE="${*:?Usage: new-adr.sh <title of decision>}"
SLUG=$(echo "$TITLE" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9-')

FILENAME="$ADR_DIR/${NEXT}-${SLUG}.md"

# Create from template
sed "s/ADR-NNNN/ADR-${NEXT}/; s/\[Short Title of Decision\]/${TITLE}/" "$TEMPLATE" > "$FILENAME"

echo "Created: $FILENAME"
echo "Next step: edit the file and submit a PR for review."
```

Usage:

```bash
./scripts/new-adr.sh "Use Redis for session caching"
# Created: docs/adr/0006-use-redis-for-session-caching.md
```

## Why It Works

ADRs succeed because they are lightweight (one file per decision), discoverable (stored in the repo, linked in PRs), and immutable (superseded, never deleted). New team members can read the decision log to understand why the system is shaped the way it is, rather than guessing or re-debating settled choices. The sequential numbering creates a timeline of the architecture's evolution, and the structured format ensures that context and trade-offs are always captured, not just the final choice.

## Context

This pattern follows the format originally proposed by Michael Nygard. It works best when ADRs are reviewed through the same PR process as code changes, ideally with the ADR submitted alongside the code that implements the decision. Teams should establish a lightweight threshold for when an ADR is required -- a good rule of thumb is "any decision that would be hard to reverse or that future developers will ask about."
