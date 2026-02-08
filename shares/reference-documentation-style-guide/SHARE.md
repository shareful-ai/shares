---
title: "Developer documentation style guide reference"
slug: reference-documentation-style-guide
tags: [style-guide, writing, conventions, tone]
problem: "Docs lack consistency in tone formatting and terminology across multiple contributors"
solution_type: reference
created: "2026-02-08"
---

## Problem

When more than one person contributes to a documentation site, inconsistencies creep in quickly: some pages use "you" while others use "we," headings alternate between title case and sentence case, code blocks sometimes have language annotations and sometimes do not, and the same concept is referred to by different names on different pages. Readers lose trust when the docs feel like a patchwork of competing voices.

## Solution

Adopt the following style guide conventions and enforce them with a prose linter (see the Vale config share for automation).

### Voice and tone

- Use **second person** ("you") when addressing the reader. Avoid "we" except when referring to your organization by name.
- Use **active voice**. Write "The server returns a 404 error" not "A 404 error is returned by the server."
- Be direct and concise. Prefer imperative sentences in procedural steps: "Run the migration" not "You should run the migration."
- Maintain a professional but approachable tone. Avoid humor that does not translate across cultures.

### Tense

- Use **present tense** for descriptions of how things work: "The function accepts a string and returns a boolean."
- Use present tense for results of actions: "After you save the file, the server restarts."
- Reserve past tense for changelogs and migration guides that describe previous behavior.

### Sentence and paragraph length

- Aim for sentences under 25 words. If a sentence exceeds 30 words, split it.
- Keep paragraphs to three to five sentences. Long paragraphs discourage scanning.
- Use bulleted lists when presenting three or more parallel items.

### Heading conventions

- Use **sentence case** for all headings: "Configure the database connection" not "Configure the Database Connection."
- Start headings with a verb for task-based pages: "Install dependencies," "Create a configuration file."
- Do not skip heading levels. An `h2` must be followed by an `h3`, never an `h4`.
- Do not end headings with punctuation.

### Code block formatting

- Always specify the language identifier on fenced code blocks for syntax highlighting.
- Use `js` not `javascript`, `ts` not `typescript`, `py` not `python`, `sh` not `bash` for brevity unless your site tooling requires the full name.
- Include a title or filename comment when the code block represents a specific file.
- Keep code blocks under 40 lines. If longer, extract into a linked example file.
- Use `// highlight-next-line` or equivalent for the doc platform to draw attention to the important line.

````markdown
```js title="docusaurus.config.js"
module.exports = {
  title: 'My Site',
  // highlight-next-line
  url: 'https://example.com',
};
```
````

### Admonitions

Use admonitions sparingly. When you use them, follow these conventions:

| Type | When to use |
|---|---|
| `note` | Supplementary information the reader may find useful but can skip. |
| `tip` | A shortcut or best practice that saves time. |
| `info` | Background context that helps the reader understand why something works a certain way. |
| `warning` | Something that could cause unexpected behavior if ignored. |
| `danger` | Something that could cause data loss, security vulnerabilities, or downtime. |

Do not use more than two admonitions per page. If you need more, the page is trying to cover too many topics.

### Word list

Use these preferred terms consistently across all documentation.

| Preferred | Avoid |
|---|---|
| open source | open-source (as a noun or adjective before a noun) |
| command line (noun), command-line (adjective) | commandline, command line (as adjective) |
| email | e-mail |
| set up (verb), setup (noun/adjective) | set-up |
| log in (verb), login (noun/adjective) | log-in |
| API key | api key, API-key, apikey |
| repository | repo (in formal docs; "repo" is acceptable in tutorials) |
| select | click on |
| enter | type in |
| ID | id, Id |
| URL | url, Url |
| okay | OK, ok, O.K. |
| for example | e.g. (spell it out in running text) |
| that is | i.e. (spell it out in running text) |

### Link text conventions

- Use descriptive link text that tells the reader where the link leads. Write "see the [installation guide](/docs/install)" not "click [here](/docs/install)."
- Do not use "click here," "this page," or "this link" as link text.
- Use relative links for internal pages and absolute links for external resources.
- Check links regularly with a link checker in CI.

### Lists

- Use numbered lists only when order matters (steps in a procedure).
- Use bulleted lists for unordered items.
- Start each list item with a capital letter.
- Do not end list items with periods unless they are complete sentences.
- Keep list items parallel in structure. If one item starts with a verb, all items should start with a verb.

## Why It Works

Consistency reduces cognitive load for readers. When formatting, tone, and terminology are predictable, readers focus on the content rather than the presentation. A published style guide also speeds up reviews because reviewers can point to a specific rule instead of debating subjective preferences in every pull request.

## Context

This style guide is intentionally opinionated. Adapt the specific choices (sentence case vs title case, "you" vs "we") to your organization's preferences, but make a decision and stick with it. The important thing is not which convention you choose but that you choose one and enforce it consistently. Pair this guide with a Vale configuration to automate enforcement in CI.
