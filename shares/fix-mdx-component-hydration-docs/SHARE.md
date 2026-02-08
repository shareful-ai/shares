---
title: "Fix MDX component hydration errors in docs sites"
slug: fix-mdx-component-hydration-docs
tags: [mdx, hydration, react, docusaurus]
problem: "Interactive MDX components throw hydration mismatch errors in documentation sites"
solution_type: fix
created: "2026-02-08"
---

## Problem

Custom React components embedded in MDX documentation files throw hydration errors:

```
Warning: Expected server HTML to contain a matching <div> in <p>.
Error: Hydration failed because the initial UI does not match what was rendered on the server.
```

This happens when components render different HTML on the server vs the client, or when block-level elements are nested inside `<p>` tags.

## Solution

Two common causes and fixes:

**Cause 1: Block elements inside paragraphs**

MDX wraps inline content in `<p>` tags. If your component renders a `<div>`, it creates invalid HTML (`<p><div>...</div></p>`).

```mdx
{/* Bad - component is inline, gets wrapped in <p> */}
Here is an example:
<MyComponent />

{/* Good - blank lines make it a block element */}
Here is an example:

<MyComponent />
```

**Cause 2: Client-only content**

Components that use `window`, `localStorage`, or other browser APIs render differently on server vs client.

```tsx
import BrowserOnly from '@docusaurus/BrowserOnly';

export function InteractiveDemo() {
  return (
    <BrowserOnly fallback={<div>Loading...</div>}>
      {() => <ClientOnlyWidget />}
    </BrowserOnly>
  );
}
```

## Why It Works

React hydration expects server-rendered HTML to exactly match what the client renders. Block elements inside `<p>` tags cause browsers to break the DOM structure, creating a mismatch. `BrowserOnly` prevents server rendering of client-dependent components, avoiding the mismatch entirely.

## Context

- Framework: Docusaurus 3.x with MDX 3
- Language: TypeScript/JSX
- Also applies to Next.js MDX and other React-based doc sites
