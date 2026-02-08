---
title: "Fix GitHub Pages 404 on SPA docs site client-side routes"
slug: fix-github-pages-404-spa-routing
tags: [github-pages, spa, routing, 404]
problem: "Refreshing or deep-linking a client-side route on GitHub Pages returns a 404"
solution_type: fix
created: "2026-02-08"
---

## Problem

A single-page application (SPA) docs site hosted on GitHub Pages returns a 404 error when users refresh the page or navigate directly to a URL like `/docs/guide/setup`. GitHub Pages only serves static files, so it looks for a literal file at that path and fails.

## Solution

Create a `404.html` that redirects to `index.html` with the original path preserved as a query parameter:

```html
<!-- public/404.html -->
<!DOCTYPE html>
<html>
<head>
  <script>
    // Store the path and redirect to index
    const path = window.location.pathname + window.location.search + window.location.hash;
    window.location.replace(
      window.location.origin + '/?redirect=' + encodeURIComponent(path)
    );
  </script>
</head>
<body></body>
</html>
```

Then in your app's entry point, handle the redirect:

```js
// In your app's initialization
const params = new URLSearchParams(window.location.search);
const redirect = params.get('redirect');
if (redirect) {
  window.history.replaceState(null, '', redirect);
}
```

For Docusaurus or VitePress sites, the simpler approach is to use `trailingSlash: true` in the config, which generates `path/index.html` files for every route:

```js
// docusaurus.config.js
module.exports = {
  trailingSlash: true,
};
```

## Why It Works

GitHub Pages serves `404.html` for any route that doesn't match a file. The redirect trick uses this to bounce users to the SPA entry point while preserving the intended URL. The `trailingSlash` approach avoids the problem entirely by generating a physical `index.html` file for every route.

## Context

- Hosting: GitHub Pages
- Framework: Any SPA (React, Vue, Docusaurus, VitePress)
- The `trailingSlash` solution is preferred when using a static site generator
