---
title: "Work around Notion API not supporting full block export"
slug: workaround-notion-api-no-blocks-export
tags: [notion, api, export, migration]
problem: "Notion API does not export all block types making full content migration incomplete"
solution_type: workaround
created: "2026-02-08"
---

## Problem

When migrating content from Notion using the official API, several block types are not fully supported. Embedded databases, synced blocks, and some media types return incomplete data or are missing entirely. The Notion API's `blocks.children.list` endpoint returns `unsupported` for certain block types, making automated migration incomplete.

## Solution

Use a two-pass approach: API export for supported blocks, then Notion's built-in HTML/Markdown export for the rest.

Pass 1 — API export for structured content:

```typescript
import { Client } from '@notionhq/client';

const notion = new Client({ auth: process.env.NOTION_TOKEN });

async function exportPage(pageId: string): Promise<string> {
  const blocks = await notion.blocks.children.list({ block_id: pageId });
  let markdown = '';

  for (const block of blocks.results) {
    if (block.type === 'unsupported') {
      markdown += `<!-- UNSUPPORTED BLOCK: manual review needed -->\n\n`;
      continue;
    }
    markdown += blockToMarkdown(block);
  }

  return markdown;
}

function blockToMarkdown(block: any): string {
  switch (block.type) {
    case 'paragraph':
      return richTextToMd(block.paragraph.rich_text) + '\n\n';
    case 'heading_1':
      return '# ' + richTextToMd(block.heading_1.rich_text) + '\n\n';
    case 'heading_2':
      return '## ' + richTextToMd(block.heading_2.rich_text) + '\n\n';
    case 'heading_3':
      return '### ' + richTextToMd(block.heading_3.rich_text) + '\n\n';
    case 'code':
      return '```' + (block.code.language || '') + '\n' +
        richTextToMd(block.code.rich_text) + '\n```\n\n';
    case 'bulleted_list_item':
      return '- ' + richTextToMd(block.bulleted_list_item.rich_text) + '\n';
    case 'numbered_list_item':
      return '1. ' + richTextToMd(block.numbered_list_item.rich_text) + '\n';
    default:
      return `<!-- ${block.type}: not converted -->\n\n`;
  }
}

function richTextToMd(richText: any[]): string {
  return richText.map(t => {
    let text = t.plain_text;
    if (t.annotations.bold) text = `**${text}**`;
    if (t.annotations.italic) text = `*${text}*`;
    if (t.annotations.code) text = '`' + text + '`';
    if (t.href) text = `[${text}](${t.href})`;
    return text;
  }).join('');
}
```

Pass 2 — Fill gaps from Notion's HTML export:

```bash
# Export from Notion UI: Settings > Export > HTML
# Then convert unsupported blocks manually using the HTML as reference
```

## Why It Works

The Notion API is designed for integrations, not full-fidelity export. By using the API for standard blocks (text, headings, lists, code) and falling back to Notion's own export for complex blocks (databases, embeds, synced blocks), you get the best of both approaches. The HTML comments in the output mark where manual review is needed.

## Context

- API: Notion API v2022-06-28+
- Language: TypeScript
- Package: @notionhq/client
- Known unsupported types: child_database, synced_block, table_of_contents, breadcrumb
