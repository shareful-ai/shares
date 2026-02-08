---
title: "Improving Knowledge Base Search Relevance with Weighted Fields"
slug: "pattern-knowledge-base-search-relevance"
solution_type: pattern
tags:
  - search
  - relevance
  - ranking
  - knowledge-base
created: "2026-02-08"
---

## Problem

Knowledge base search returns irrelevant results because all text fields are weighted equally, burying exact title matches below pages that mention the term many times in body text.

## Solution

Implement a field-weighted scoring function that prioritizes matches in titles over headings and headings over body content, combine it with BM25 for term frequency normalization, and boost by document freshness to surface current information.

### 1. Field-Weighted Scoring Function

The core idea is to assign different importance multipliers to each document field:

```
score(query, doc) = w_title  * BM25(query, doc.title)
                  + w_heading * BM25(query, doc.headings)
                  + w_body    * BM25(query, doc.body)
                  + freshness_boost(doc.updated_at)
```

Practical weights that work well for developer documentation:

| Field | Weight | Rationale |
|-------|--------|-----------|
| `title` | 10.0 | Titles are the strongest signal of page topic |
| `headings` (h2, h3) | 5.0 | Section headings indicate sub-topics |
| `body` | 1.0 | Body text provides supporting context |
| `tags` / `keywords` | 8.0 | Explicit metadata from the author |
| `code_blocks` | 2.0 | Useful for API/function name searches |

### 2. BM25 Basics

BM25 (Best Matching 25) is the standard ranking function used by Elasticsearch, Meilisearch, and most search engines. It improves on raw term frequency by accounting for document length and diminishing returns:

```
BM25(query, field) = IDF(term) * (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * (fieldLen / avgFieldLen)))
```

Where:
- `tf` = term frequency in the field
- `IDF` = inverse document frequency (rarer terms score higher)
- `k1` = term saturation parameter (typically 1.2 -- limits the effect of repeated terms)
- `b` = length normalization (typically 0.75 -- penalizes very long documents)

You rarely need to tune `k1` and `b` directly. The field weights above have far more impact.

### 3. Elasticsearch Implementation

```json
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "title": {
              "query": "authentication setup",
              "boost": 10
            }
          }
        },
        {
          "match": {
            "headings": {
              "query": "authentication setup",
              "boost": 5
            }
          }
        },
        {
          "match": {
            "tags": {
              "query": "authentication setup",
              "boost": 8
            }
          }
        },
        {
          "match": {
            "body": {
              "query": "authentication setup",
              "boost": 1
            }
          }
        }
      ],
      "functions": [
        {
          "gauss": {
            "updated_at": {
              "origin": "now",
              "scale": "90d",
              "decay": 0.5
            }
          },
          "weight": 1.5
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

### 4. Meilisearch Implementation

For teams using Meilisearch (simpler setup, good for smaller knowledge bases):

```javascript
import { MeiliSearch } from 'meilisearch';

const client = new MeiliSearch({ host: 'http://localhost:7700' });

// Configure the index with field ranking
await client.index('docs').updateSettings({
  // Fields searched in priority order (Meilisearch uses ordered ranking)
  searchableAttributes: [
    'title',        // Searched first, highest priority
    'tags',         // Second priority
    'headings',     // Third priority
    'code_blocks',  // Fourth priority
    'body',         // Lowest priority
  ],

  // Ranking rules (order matters)
  rankingRules: [
    'words',        // Number of matched query terms
    'typo',         // Fewer typos rank higher
    'proximity',    // Closer query terms rank higher
    'attribute',    // Matches in higher-priority fields rank higher
    'sort',
    'exactness',    // Exact matches rank higher
    'updated_at:desc',  // Freshness tiebreaker
  ],

  filterableAttributes: ['type', 'tags', 'updated_at'],
  sortableAttributes: ['updated_at'],
});

// Index a document
await client.index('docs').addDocuments([
  {
    id: 'configure-sso',
    title: 'Configure SSO Authentication',
    headings: ['Prerequisites', 'SAML Setup', 'OIDC Setup', 'Testing'],
    tags: ['authentication', 'sso', 'saml', 'oidc'],
    body: 'This guide walks you through setting up Single Sign-On...',
    code_blocks: 'AUTH_SSO_ENABLED=true\nAUTH_SSO_PROVIDER=okta',
    type: 'how-to',
    updated_at: '2026-01-20T00:00:00Z',
  },
]);

// Search with filtering
const results = await client.index('docs').search('sso authentication', {
  filter: 'type = "how-to"',
  limit: 10,
  attributesToHighlight: ['title', 'body'],
});
```

### 5. Freshness Boost Function

Apply a time-decay function so recently updated documents score slightly higher when relevance is otherwise equal:

```typescript
function freshnessBoost(updatedAt: Date, halfLife: number = 90): number {
  const daysOld = (Date.now() - updatedAt.getTime()) / (1000 * 60 * 60 * 24);
  // Exponential decay: score halves every `halfLife` days
  // Returns 1.0 for today, 0.5 for 90 days ago, 0.25 for 180 days ago
  return Math.pow(0.5, daysOld / halfLife);
}

function computeScore(query: string, doc: Document): number {
  const titleScore  = 10.0 * bm25(query, doc.title);
  const headScore   =  5.0 * bm25(query, doc.headings);
  const tagScore    =  8.0 * bm25(query, doc.tags);
  const bodyScore   =  1.0 * bm25(query, doc.body);

  const relevance = titleScore + headScore + tagScore + bodyScore;
  const freshness = freshnessBoost(doc.updatedAt);

  // Freshness is a mild multiplier (0.8-1.2 range), not a dominant factor
  return relevance * (0.8 + 0.4 * freshness);
}
```

### 6. Measuring Relevance Quality

Track these metrics to know if your ranking is actually improving:

- **Mean Reciprocal Rank (MRR)**: Average of 1/position-of-first-relevant-result. Target > 0.7.
- **Click-through rate on rank 1**: If users consistently click the first result, ranking is working. Target > 60%.
- **Search refinement rate**: How often users modify their query. Lower is better.
- **Zero-results rate**: Percentage of queries with no results. Target < 5%.

## Why It Works

Field weighting aligns with how documentation is structured. Authors put the most descriptive, specific language in titles and headings. A document titled "Configure SSO" is almost certainly more relevant to the query "configure SSO" than a troubleshooting page that mentions SSO once in its body. BM25's length normalization prevents long pages from dominating results simply because they contain more text. The freshness boost acts as a tiebreaker, ensuring that when two pages are equally relevant, the more recently maintained one surfaces first.

## Context

This pattern applies to any full-text search engine (Elasticsearch, Meilisearch, Typesense, Algolia, PostgreSQL full-text search). The specific weight values (10/5/1) are a strong starting point but should be tuned based on your search analytics. Start with these defaults, measure MRR and click-through rates, and adjust. Most teams find that increasing the title weight has the single biggest impact on perceived search quality.
