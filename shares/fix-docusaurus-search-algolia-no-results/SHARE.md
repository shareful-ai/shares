---
title: "Fix Docusaurus Algolia search returning zero results"
slug: fix-docusaurus-search-algolia-no-results
tags: [docusaurus, algolia, search, indexing]
problem: "Algolia DocSearch returns no results because the crawler config does not match the site HTML structure"
solution_type: fix
created: "2026-02-08"
---

## Problem

After setting up Algolia DocSearch with Docusaurus, search returns zero results even though pages are indexed. The crawler runs successfully but the search bar shows "No results" for every query. This happens because the default Algolia crawler selectors target HTML elements that don't exist in your Docusaurus theme.

## Solution

Update your Algolia crawler configuration to match the actual HTML structure of your Docusaurus site. In your `docusaurus.config.js`:

```js
module.exports = {
  themeConfig: {
    algolia: {
      appId: 'YOUR_APP_ID',
      apiKey: 'YOUR_SEARCH_API_KEY',
      indexName: 'YOUR_INDEX_NAME',
      contextualSearch: true,
      searchParameters: {},
    },
  },
};
```

Then update the crawler config in the Algolia dashboard. The key fix is ensuring `recordExtractor` targets the correct selectors:

```json
{
  "index_name": "YOUR_INDEX_NAME",
  "start_urls": ["https://your-site.com/"],
  "selectors": {
    "lvl0": {
      "selector": ".menu__link--sublist.menu__link--active",
      "global": true,
      "default_value": "Documentation"
    },
    "lvl1": "article h1",
    "lvl2": "article h2",
    "lvl3": "article h3",
    "content": "article p, article li, article td"
  }
}
```

## Why It Works

Docusaurus wraps documentation content in an `<article>` element. The default DocSearch crawler config often targets generic selectors like `h1`, `h2`, `p` which match navigation elements, footers, and sidebars — polluting the index with irrelevant content. By scoping selectors to `article`, the crawler only indexes actual page content, making search results accurate.

## Context

- Framework: Docusaurus 3.x
- Search: Algolia DocSearch v3
- Also applies to custom Docusaurus themes that wrap content in `<article>`
