---
title: "Fix MkDocs Material search not finding CJK characters"
slug: fix-mkdocs-material-search-cjk
tags: [mkdocs, search, cjk, i18n]
problem: "Built-in search in MkDocs Material fails to match Chinese, Japanese, or Korean text"
solution_type: fix
created: "2026-02-08"
---

## Problem

MkDocs Material's built-in search plugin cannot find Chinese, Japanese, or Korean (CJK) text. Searching for CJK terms returns zero results even though the content exists. The default search tokenizer splits on whitespace and word boundaries, which doesn't work for CJK languages where words are not separated by spaces.

## Solution

Install and configure the `jieba` tokenizer for Chinese, or use the built-in separator configuration for CJK support:

```yaml
# mkdocs.yml
plugins:
  - search:
      separator: '[\s\-\.]+'
      lang:
        - en
        - ja
        - zh
```

For better Chinese segmentation, install `jieba`:

```bash
pip install jieba
```

Then configure:

```yaml
plugins:
  - search:
      jieba_dict: dict.txt  # optional custom dictionary
      separator: '[\s\-\.]+'
      lang:
        - en
        - zh
```

## Why It Works

CJK languages don't use spaces between words, so the default whitespace-based tokenizer treats entire sentences as single tokens. Adding CJK language support enables character-level indexing, and `jieba` provides proper word segmentation for Chinese text. The `separator` regex also helps by splitting on additional boundary characters.

## Context

- Framework: MkDocs Material 9.x
- Language: Python
- Requires: `jieba` package for Chinese word segmentation
- Japanese support uses `tinysegmenter` (bundled with MkDocs)
