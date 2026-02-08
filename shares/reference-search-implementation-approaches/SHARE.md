---
title: "Compare search implementation approaches for docs sites"
slug: reference-search-implementation-approaches
tags: [search, algolia, typesense, comparison]
problem: "Teams do not know which search solution fits their docs site size and budget"
solution_type: reference
created: "2026-02-08"
---

## Problem

Documentation sites need search, but the landscape ranges from free client-side libraries to expensive hosted services. Teams either pick the first option they find (often Algolia because it is popular) without understanding the trade-offs, or they build a fragile custom solution. The wrong choice surfaces as slow search on large sites, unexpected invoices when traffic spikes, or missing features like typo tolerance that users expect.

## Solution

Use the comparison below to pick the search approach that matches your document count, budget, and infrastructure preferences.

### Feature comparison matrix

| Feature | Algolia DocSearch | Typesense | Lunr.js | Pagefind | FlexSearch | Orama |
|---|---|---|---|---|---|---|
| Architecture | Hosted (server-side) | Self-hosted or Typesense Cloud | Client-side | Client-side (WASM) | Client-side | Client-side or server-side |
| Max practical doc count | Unlimited (plan-based) | Unlimited (resource-based) | ~5,000 pages | ~50,000 pages | ~10,000 pages | ~50,000 pages |
| Index size concern | None (server-side) | None (server-side) | Full index in browser | Chunked binary index | Full index in browser | Chunked or in-memory |
| Typo tolerance | Yes | Yes | No | Yes | No | Yes |
| Faceted search | Yes | Yes | No | No | No | Yes |
| Latency (typical) | <50ms | <50ms | <10ms (local) | <20ms (local) | <5ms (local) | <10ms (local) |
| Highlighting | Yes | Yes | Manual | Yes | Manual | Yes |
| Pricing | Free for open source docs; paid plans from $1/1000 requests | Free (self-hosted); Cloud from $0.035/hr | Free (MIT) | Free (MIT) | Free (Apache 2.0) | Free (Apache 2.0) |
| Analytics | Yes | Yes (Cloud) | No | No | No | Plugin-based |

### Integration examples

#### Algolia DocSearch

Apply at [docsearch.algolia.com](https://docsearch.algolia.com) for free open source docs hosting, or use the paid API.

```js title="docusaurus.config.js"
module.exports = {
  themeConfig: {
    algolia: {
      appId: 'YOUR_APP_ID',
      apiKey: 'YOUR_SEARCH_ONLY_API_KEY',
      indexName: 'YOUR_INDEX_NAME',
      contextualSearch: true,
    },
  },
};
```

For non-Docusaurus sites, install the frontend library:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@docsearch/css@3" />
<script src="https://cdn.jsdelivr.net/npm/@docsearch/js@3"></script>
<script>
  docsearch({
    appId: 'YOUR_APP_ID',
    apiKey: 'YOUR_SEARCH_ONLY_API_KEY',
    indexName: 'YOUR_INDEX_NAME',
    container: '#search',
  });
</script>
```

#### Typesense

Run a Typesense server and use the InstantSearch adapter.

```sh
# Start Typesense with Docker
docker run -p 8108:8108 \
  -v /tmp/typesense-data:/data \
  typesense/typesense:27.1 \
  --data-dir /data \
  --api-key=xyz123 \
  --enable-cors
```

```js title="search.js"
import TypesenseInstantSearchAdapter from 'typesense-instantsearch-adapter';

const typesenseAdapter = new TypesenseInstantSearchAdapter({
  server: {
    apiKey: 'xyz123',
    nodes: [{ host: 'localhost', port: 8108, protocol: 'http' }],
  },
  additionalSearchParameters: {
    query_by: 'title,content',
    typo_tokens_threshold: 1,
  },
});

const searchClient = typesenseAdapter.searchClient;
```

#### Lunr.js

Best for small sites under 5,000 pages where you want zero external dependencies.

```js title="build-index.js"
const lunr = require('lunr');
const fs = require('fs');

const documents = JSON.parse(fs.readFileSync('docs.json', 'utf8'));

const index = lunr(function () {
  this.ref('id');
  this.field('title', { boost: 10 });
  this.field('body');

  documents.forEach((doc) => {
    this.add(doc);
  });
});

fs.writeFileSync('search-index.json', JSON.stringify(index));
```

```js title="search-client.js"
const index = lunr.Index.load(JSON.parse(searchIndexJson));
const results = index.search('authentication');
// results: [{ ref: 'doc-42', score: 1.23, matchData: {...} }]
```

#### Pagefind

Pagefind indexes your static site at build time and serves a chunked WASM-powered index. No server required.

```sh
# Run after your static site build
npx pagefind --site public --output-subdir _pagefind
```

```html title="search.html"
<link href="/_pagefind/pagefind-ui.css" rel="stylesheet" />
<script src="/_pagefind/pagefind-ui.js"></script>
<div id="search"></div>
<script>
  new PagefindUI({ element: '#search', showSubResults: true });
</script>
```

#### FlexSearch

Extremely fast in-memory search with a small footprint. No typo tolerance.

```js title="search.js"
import FlexSearch from 'flexsearch';

const index = new FlexSearch.Document({
  document: {
    id: 'id',
    index: ['title', 'content'],
    store: ['title', 'url'],
  },
  tokenize: 'forward',
});

documents.forEach((doc) => index.add(doc));

const results = index.search('authentication', { limit: 10, enrich: true });
```

#### Orama

Full-text search with typo tolerance, facets, and optional server-side deployment.

```js title="search.js"
import { create, insert, search } from '@orama/orama';

const db = await create({
  schema: {
    title: 'string',
    content: 'string',
    category: 'enum',
  },
});

await insert(db, {
  title: 'Authentication guide',
  content: 'Learn how to set up OAuth2...',
  category: 'guides',
});

const results = await search(db, {
  term: 'authentcation', // typo tolerance catches this
  tolerance: 1,
  properties: ['title', 'content'],
  facets: { category: {} },
});
```

### Decision guide

- **Under 1,000 pages, zero budget** -- Use Pagefind or Lunr.js. Pagefind is preferred because it handles larger sites and includes typo tolerance.
- **1,000 to 10,000 pages, zero budget** -- Use Pagefind or Orama. Both handle the scale client-side without a server.
- **Any size, need facets and analytics** -- Use Algolia (if budget allows) or Typesense (if you can run infrastructure).
- **Open source project** -- Apply for Algolia DocSearch (free) or use Pagefind (self-contained, no account needed).
- **Enterprise with compliance requirements** -- Use Typesense self-hosted to keep data on your infrastructure.

## Why It Works

Search is the primary navigation method for documentation sites -- studies show that over 50 percent of docs visitors go straight to the search bar. Matching the search solution to your site's scale and budget ensures fast results without overengineering. Client-side solutions eliminate server costs and latency for small sites. Server-side solutions handle scale and advanced features like faceted search when you need them.

## Context

This comparison covers search-as-a-feature for documentation sites specifically. If you are building search for an e-commerce product catalog or a general-purpose application, the requirements differ significantly (geosearch, personalization, merchandising). The code snippets show minimal integrations; consult each library's documentation for production configuration including index refresh strategies, relevance tuning, and analytics setup.
