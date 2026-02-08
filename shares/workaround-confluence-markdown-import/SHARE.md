---
title: "Work around Confluence lacking native markdown import"
slug: workaround-confluence-markdown-import
tags: [confluence, markdown, import, migration]
problem: "Confluence does not natively import markdown files requiring manual conversion"
solution_type: workaround
created: "2026-02-08"
---

## Problem

Confluence Cloud does not support importing markdown files directly. When migrating docs from a markdown-based system (MkDocs, Docusaurus, GitHub wiki) to Confluence, there's no built-in import tool. You need to convert markdown to Confluence's storage format (XHTML) before uploading via API.

## Solution

Use `markdown-to-confluence` or build a custom pipeline with `pandoc` and the Confluence REST API:

```bash
# Convert markdown to Confluence storage format using pandoc
pandoc input.md -f markdown -t html --wrap=none -o output.html
```

Then upload via the Confluence API:

```python
import requests
import subprocess
import os

CONFLUENCE_URL = os.environ['CONFLUENCE_URL']
CONFLUENCE_TOKEN = os.environ['CONFLUENCE_TOKEN']
SPACE_KEY = 'DOCS'

def md_to_confluence_html(md_path: str) -> str:
    result = subprocess.run(
        ['pandoc', md_path, '-f', 'markdown', '-t', 'html', '--wrap=none'],
        capture_output=True, text=True
    )
    return result.stdout

def create_page(title: str, body_html: str, parent_id: str = None):
    data = {
        'type': 'page',
        'title': title,
        'space': {'key': SPACE_KEY},
        'body': {
            'storage': {
                'value': body_html,
                'representation': 'storage'
            }
        }
    }
    if parent_id:
        data['ancestors'] = [{'id': parent_id}]

    resp = requests.post(
        f'{CONFLUENCE_URL}/rest/api/content',
        json=data,
        headers={
            'Authorization': f'Bearer {CONFLUENCE_TOKEN}',
            'Content-Type': 'application/json'
        }
    )
    resp.raise_for_status()
    return resp.json()

# Batch import
for md_file in sorted(os.listdir('docs/')):
    if md_file.endswith('.md'):
        title = md_file.replace('.md', '').replace('-', ' ').title()
        html = md_to_confluence_html(f'docs/{md_file}')
        page = create_page(title, html)
        print(f'Created: {page["title"]} ({page["_links"]["webui"]})')
```

## Why It Works

Pandoc converts markdown to clean HTML that Confluence's storage format accepts. The REST API then creates pages with the converted content. This two-step pipeline handles most markdown features including headings, lists, code blocks, tables, and links. Complex features like Mermaid diagrams or custom MDX components need manual conversion.

## Context

- Platform: Confluence Cloud
- Tool: pandoc (for conversion), Python requests (for API upload)
- Requires: Confluence API token with write permissions
- Limitation: Images must be uploaded separately via the attachment API
