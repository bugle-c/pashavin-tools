---
name: pashavin-blog
description: Use when generating, publishing, or managing blog articles on pashavin.ru — auto-generation from Claude Code logs, secret filtering, SEO audit, MDX management
---

# pashavin.ru Blog Pipeline

## Quick Reference

| Item | Value |
|------|-------|
| Blog dir | `content/blog/` |
| Articles format | MDX with YAML frontmatter |
| Generation script | `scripts/generate-blog.ts` |
| Secret scanner | `lib/blog-secrets-filter.ts` |
| Blog data lib | `lib/blog.ts` |
| Status flow | `draft` → SEO audit → `published` (or `blocked`) |
| Language | Russian (content), English (code/slugs) |
| Env required | `ANTHROPIC_API_KEY` in `.env.local` |

## Commands

```bash
# Load env (required for generation)
set -a && source .env.local && set +a

# List available sessions
npx tsx scripts/generate-blog.ts --list

# Generate from latest session
npx tsx scripts/generate-blog.ts

# Generate from specific session
npx tsx scripts/generate-blog.ts --session=<ID>

# Generate from project's sessions
npx tsx scripts/generate-blog.ts --project=<dir-name>

# Publish a draft
npx tsx scripts/generate-blog.ts --publish=<slug>

# Build & verify
npm run build
```

## Article Lifecycle (MANDATORY — follow every step)

### Step 1: Generate
```bash
set -a && source .env.local && set +a
npx tsx scripts/generate-blog.ts --session=<ID>
```
Script reads Claude logs + git → Claude API generates article → secret scan → saves as `draft`.

### Step 2: Review draft
Human checks `content/blog/<slug>.mdx` for accuracy, tone, completeness.

### Step 3: Publish draft
```bash
npx tsx scripts/generate-blog.ts --publish=<slug>
```

### Step 4: Build & verify
```bash
npm run build
```

### Step 5: SEO audit (MANDATORY before deploy)
Invoke SEO skills to audit the published article:
- `/seo-page <article-url>` — on-page SEO, meta tags, schema validation
- `/seo-content <article-url>` — E-E-A-T, content depth, AI citation readiness
- `/seo-schema <article-url>` — JSON-LD validation

Fix all HIGH and CRITICAL issues before deploying.

### Step 6: Deploy
Commit + push triggers Dokploy auto-deploy.

## Frontmatter Format

```yaml
---
title: "SEO-заголовок (50-60 символов)"
slug: "transliterated-slug"
date: "2026-03-15"
tags: ["тег1", "тег2"]
description: "Мета-описание 150-160 символов для SEO"
status: "draft"        # draft | published | blocked
relatedProject: "slug" # optional — links to portfolio project
---
```

## Secret Filtering

Two-layer protection:
1. **AI prompt** — explicit instructions to never include credentials
2. **Regex scanner** — `scanForSecrets()` checks 12 patterns (passwords, tokens, IPs, JWTs, etc.)

If secrets detected → article saved as `status: "blocked"`. Must be manually cleaned before publishing.

## SEO Checklist (built into blog pages)

- `generateMetadata` with title, description, OG + Twitter Card, canonical
- JSON-LD `BlogPosting` with image, dateModified, mainEntityOfPage, wordCount, inLanguage
- JSON-LD `BreadcrumbList` (Home → Blog → Article)
- Per-article OG image generation (`opengraph-image.tsx`)
- Sitemap includes all published blog posts
- Clean HTML layout (no heavy JS) for crawler indexability
- `robots: { index: true, follow: true }` on all blog pages
- Tag pages with proper canonicals

## Architecture

- Blog uses separate route group from portfolio (no Loader/GSAP)
- `app/blog/layout.tsx` + `app/blog/blog-shell.tsx` — client shell with ContactDialog
- `app/blog/page.tsx` — listing with tag filters
- `app/blog/[slug]/page.tsx` — article with MDX rendering via `next-mdx-remote`
- `app/blog/tag/[tag]/page.tsx` — tag filter page
- Typography via `@tailwindcss/typography` prose classes
- Header: logo, blog, projects, "Связаться" (ContactDialog popup)
- Footer: copyright, Telegram button, "Обсудить проект" (ContactDialog popup)

## Writing Articles Manually

Create `content/blog/<slug>.mdx` with frontmatter + Markdown content. Use `##` for headings, triple backticks for code blocks. Set `status: "published"` to make it visible. Still run SEO audit before deploy.
