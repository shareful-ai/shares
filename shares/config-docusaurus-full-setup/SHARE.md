---
title: "Configure Docusaurus with search versioning and i18n"
slug: config-docusaurus-full-setup
tags: [docusaurus, setup, versioning, i18n]
problem: "Setting up Docusaurus with all enterprise features requires configuring many separate plugins"
solution_type: config
created: "2026-02-08"
---

## Problem

Docusaurus supports versioning, internationalization, search, diagrams, and progressive web app features, but each capability lives in a separate plugin or theme with its own configuration surface. Teams either miss important settings during initial setup or end up with a tangled config file that is hard to maintain. Getting all of these features working together from scratch typically takes one to two days of reading scattered docs pages.

## Solution

Use the following complete configuration files as a starting point. Every section is annotated so you can remove features you do not need.

### package.json

```json title="package.json"
{
  "name": "my-docs",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "docusaurus": "docusaurus",
    "start": "docusaurus start",
    "build": "docusaurus build",
    "swizzle": "docusaurus swizzle",
    "deploy": "docusaurus deploy",
    "clear": "docusaurus clear",
    "serve": "docusaurus serve",
    "write-translations": "docusaurus write-translations",
    "write-heading-ids": "docusaurus write-heading-ids",
    "version": "docusaurus docs:version",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@docusaurus/core": "^3.7.0",
    "@docusaurus/plugin-content-blog": "^3.7.0",
    "@docusaurus/plugin-content-docs": "^3.7.0",
    "@docusaurus/plugin-sitemap": "^3.7.0",
    "@docusaurus/preset-classic": "^3.7.0",
    "@docusaurus/theme-mermaid": "^3.7.0",
    "@docusaurus/plugin-pwa": "^3.7.0",
    "clsx": "^2.1.0",
    "prism-react-renderer": "^2.4.1",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@docusaurus/module-type-aliases": "^3.7.0",
    "@docusaurus/types": "^3.7.0",
    "typescript": "^5.7.0"
  },
  "browserslist": {
    "production": [">0.5%", "not dead", "not op_mini all"],
    "development": ["last 1 chrome version", "last 1 firefox version", "last 1 safari version"]
  },
  "engines": {
    "node": ">=18.0"
  }
}
```

### docusaurus.config.js

```js title="docusaurus.config.js"
// @ts-check
const { themes: prismThemes } = require('prism-react-renderer');

/** @type {import('@docusaurus/types').Config} */
const config = {
  title: 'My Docs',
  tagline: 'Documentation for My Product',
  favicon: 'img/favicon.ico',

  url: 'https://docs.example.com',
  baseUrl: '/',

  organizationName: 'my-org',
  projectName: 'my-docs',

  onBrokenLinks: 'throw',
  onBrokenMarkdownLinks: 'throw',
  onDuplicateRoutes: 'throw',

  // ---------------------------------------------------------------------------
  // i18n: English (default) + Japanese
  // ---------------------------------------------------------------------------
  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'ja'],
    localeConfigs: {
      en: { label: 'English', direction: 'ltr', htmlLang: 'en-US' },
      ja: { label: '日本語', direction: 'ltr', htmlLang: 'ja' },
    },
  },

  // ---------------------------------------------------------------------------
  // Mermaid diagram support
  // ---------------------------------------------------------------------------
  markdown: {
    mermaid: true,
  },
  themes: ['@docusaurus/theme-mermaid'],

  // ---------------------------------------------------------------------------
  // Presets
  // ---------------------------------------------------------------------------
  presets: [
    [
      'classic',
      /** @type {import('@docusaurus/preset-classic').Options} */
      ({
        docs: {
          sidebarPath: require.resolve('./sidebars.js'),
          editUrl: 'https://github.com/my-org/my-docs/edit/main/',
          showLastUpdateTime: true,
          showLastUpdateAuthor: true,
          // Versioning: after running `npm run version 1.0`, snapshot lives in
          // versioned_docs/ and versioned_sidebars/.
          lastVersion: 'current',
          versions: {
            current: {
              label: 'Next',
              path: 'next',
            },
          },
        },
        blog: {
          showReadingTime: true,
          editUrl: 'https://github.com/my-org/my-docs/edit/main/',
          blogSidebarCount: 10,
          postsPerPage: 10,
        },
        theme: {
          customCss: require.resolve('./src/css/custom.css'),
        },
        sitemap: {
          changefreq: 'weekly',
          priority: 0.5,
          filename: 'sitemap.xml',
        },
      }),
    ],
  ],

  // ---------------------------------------------------------------------------
  // Plugins
  // ---------------------------------------------------------------------------
  plugins: [
    [
      '@docusaurus/plugin-pwa',
      {
        debug: process.env.NODE_ENV === 'development',
        offlineModeActivationStrategies: ['appInstalled', 'standalone', 'queryString'],
        pwaHead: [
          { tagName: 'link', rel: 'icon', href: '/img/logo.png' },
          { tagName: 'link', rel: 'manifest', href: '/manifest.json' },
          { tagName: 'meta', name: 'theme-color', content: '#4f46e5' },
          { tagName: 'meta', name: 'apple-mobile-web-app-capable', content: 'yes' },
          { tagName: 'meta', name: 'apple-mobile-web-app-status-bar-style', content: 'default' },
          { tagName: 'link', rel: 'apple-touch-icon', href: '/img/logo.png' },
        ],
      },
    ],
  ],

  // ---------------------------------------------------------------------------
  // Theme configuration
  // ---------------------------------------------------------------------------
  themeConfig:
    /** @type {import('@docusaurus/preset-classic').ThemeConfig} */
    ({
      image: 'img/social-card.png',

      // Algolia search
      algolia: {
        appId: 'YOUR_APP_ID',
        apiKey: 'YOUR_SEARCH_ONLY_API_KEY',
        indexName: 'YOUR_INDEX_NAME',
        contextualSearch: true, // scoped to current version + locale
      },

      navbar: {
        title: 'My Docs',
        logo: { alt: 'My Logo', src: 'img/logo.svg' },
        items: [
          { type: 'docSidebar', sidebarId: 'docs', position: 'left', label: 'Docs' },
          { to: '/blog', label: 'Blog', position: 'left' },
          { type: 'docsVersionDropdown', position: 'right' },
          { type: 'localeDropdown', position: 'right' },
          { href: 'https://github.com/my-org/my-docs', label: 'GitHub', position: 'right' },
        ],
      },

      footer: {
        style: 'dark',
        links: [
          {
            title: 'Docs',
            items: [{ label: 'Getting started', to: '/docs/next/intro' }],
          },
          {
            title: 'Community',
            items: [
              { label: 'Discord', href: 'https://discord.gg/example' },
              { label: 'GitHub', href: 'https://github.com/my-org/my-docs' },
            ],
          },
        ],
        copyright: `Copyright ${new Date().getFullYear()} My Org. Built with Docusaurus.`,
      },

      prism: {
        theme: prismThemes.github,
        darkTheme: prismThemes.dracula,
        additionalLanguages: ['bash', 'json', 'yaml', 'diff', 'docker'],
      },

      mermaid: {
        theme: { light: 'neutral', dark: 'dark' },
      },

      colorMode: {
        defaultMode: 'light',
        disableSwitch: false,
        respectPrefersColorScheme: true,
      },

      tableOfContents: {
        minHeadingLevel: 2,
        maxHeadingLevel: 4,
      },
    }),
};

module.exports = config;
```

### Creating a version

```sh
# Snapshot the current docs as version 1.0
npm run version 1.0

# This creates:
# versioned_docs/version-1.0/
# versioned_sidebars/version-1.0-sidebars.json
```

### Generating i18n translation files

```sh
# Write default English translation JSON files
npm run write-translations -- --locale en

# Write Japanese placeholder files (translate the JSON values)
npm run write-translations -- --locale ja

# Start the dev server in Japanese
npm run start -- --locale ja
```

## Why It Works

A single annotated config file is easier to reason about than a dozen separate documentation pages. Every option is visible in one place, making it clear which plugins interact (for example, `contextualSearch` in Algolia scopes results to the active version and locale automatically). Teams can copy this file, replace placeholder values, delete sections they do not need, and have a production-ready docs site in under an hour.

## Context

This config targets Docusaurus 3.7+. The PWA plugin requires a valid `manifest.json` in your `static/` folder. Algolia DocSearch is free for open source projects; apply at docsearch.algolia.com. For private docs, use Algolia's paid plan or swap in Typesense or Pagefind. The versioning model creates a full snapshot of your docs directory for each version, which increases build time proportionally; only create versions at major releases.
