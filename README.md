# fesalfayed.com

Personal site + blog. **v7** — Astro static build (hybrid: bespoke interactive
"systems board" homepage + markdown blog), deployed on Cloudflare Workers
static assets.

## Stack

- Astro 5 (content collections, MDX-ready) — zero JS by default
- Shiki dual-theme syntax highlighting, server-side KaTeX
- RSS (`/rss.xml`), sitemap, per-post OG meta
- Fonts: Newsreader / Inter / JetBrains Mono

## Develop

```bash
npm install
npm run dev      # localhost:4321
npm run build    # -> dist/
```

## Deploy

Push to `main` → Cloudflare Workers Builds runs `npm install && npm run build`
(from `wrangler.jsonc` `build.command`) then `npx wrangler versions upload`,
serving `dist/` as static assets on fesalfayed.com.

## Write a post

Drop a `.md`/`.mdx` file in `src/content/blog/` with frontmatter:

```yaml
---
title: "..."
description: "..."
date: 2026-07-06
tags: [a, b]
draft: false   # true hides it everywhere
---
```

TOC, reading time, heading anchors, RSS and sitemap entries are automatic.
