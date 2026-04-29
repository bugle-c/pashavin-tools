---
name: seo
description: >
  Comprehensive SEO analysis for any website or business type. Performs full site
  audits, single-page deep analysis, technical SEO checks (crawlability, indexability,
  Core Web Vitals with INP), schema markup detection/validation/generation, content
  quality assessment (E-E-A-T framework per Dec 2025 update extending to all
  competitive queries), image optimization, sitemap analysis, and Generative Engine
  Optimization (GEO) for AI Overviews, ChatGPT, and Perplexity citations. Analyzes
  AI crawler accessibility (GPTBot, ClaudeBot, PerplexityBot), llms.txt compliance,
  brand mention signals, and passage-level citability. Industry detection for SaaS,
  e-commerce, local business, publishers, agencies. Triggers on: "SEO", "audit",
  "schema", "Core Web Vitals", "sitemap", "E-E-A-T", "AI Overviews", "GEO",
  "technical SEO", "content quality", "page speed", "structured data".
---

# SEO — Universal SEO Analysis Skill

Comprehensive SEO analysis across all industries (SaaS, local services,
e-commerce, publishers, agencies). Orchestrates 12 specialized sub-skills
and 6 subagents (+ optional extension sub-skills).

## Project-specific standards (READ FIRST)

When working on **pashavin.ru** (`/home/deploy/projects/pashavin.ru`),
ALWAYS read `references/pashavin-ru-standards.md` **before** writing
prompts, generating content, or making metadata changes. That file encodes
the audit-derived rules for title length, title↔H1 alignment, question-led
H2 structure, direct-answer intros, freshness signals, image dimensions,
and Yandex trust paths (`/privacy`, `/contacts`, footer requisites).

## Russian-market projects (Yandex + RU AI engines) — READ FIRST

For ANY Russian-language commercial site (pashavin.ru, gptweb.ru,
likedog.ru, x10seo.ru, germanyun.ru, kp.pashavin.ru, etc.), read
`references/yandex-geo-ru-rules.md` **before** writing prompts or
auditing. It overrides generic English-market guidance with:

- Behavioural signals (dwell-time, scroll-depth) as Yandex ranking #1
- 1200-1500 word floor for RU commercial pages (not Google's "topical floor")
- Yandex trust paths: privacy + cookies + cookies-banner + custom 404 +
  legal requisites in footer + Я.Метрика (mandatory) + GA4 РКН-уведомление
- JSON-LD set per page-type (Course, Service, LocalBusiness, etc.)
- Long-tail-only keyword strategy (no fight on "bitcoin", "trading")
- Image copyright (only own / AI-generated; stock = 10k-250k ₽ fines in RF)
- GEO/AI optimization: definition-first intro, question-led H2,
  134-167 word answer blocks, /llms.txt, AI-bot allowlist in robots.txt
- Cursor vibe-coding workflow: block-by-block, version every theme zip,
  Gemini for visual, GPT-5.4 for code

For non-RU sites — fall back to general best practices below.

## Audit tooling

This skill provides strategy and standards, but it does NOT crawl pages
itself. For actual audits (Google + Yandex findings, JSON + Markdown
reports, per-template scoring), invoke the **indexlift-seo-auditor**
skill:

```bash
node /home/deploy/.claude/skills/indexlift-seo-auditor/scripts/run-audit.js \
  --url "<URL>" --tier standard --engines google,yandex \
  --output /tmp/seo-audit/
```

Use this whenever the user asks for an "audit" — `claude-seo` covers
strategy/standards, `indexlift-seo-auditor` runs the actual checks.

## Quick Reference

| Command | What it does |
|---------|-------------|
| `/seo audit <url>` | Full website audit with parallel subagent delegation |
| `/seo page <url>` | Deep single-page analysis |
| `/seo sitemap <url or generate>` | Analyze or generate XML sitemaps |
| `/seo schema <url>` | Detect, validate, and generate Schema.org markup |
| `/seo images <url>` | Image optimization analysis |
| `/seo technical <url>` | Technical SEO audit (9 categories) |
| `/seo content <url>` | E-E-A-T and content quality analysis |
| `/seo geo <url>` | AI Overviews / Generative Engine Optimization |
| `/seo plan <business-type>` | Strategic SEO planning |
| `/seo programmatic [url\|plan]` | Programmatic SEO analysis and planning |
| `/seo competitor-pages [url\|generate]` | Competitor comparison page generation |
| `/seo hreflang [url]` | Hreflang/i18n SEO audit and generation |
| `/seo dataforseo [command]` | Live SEO data via DataForSEO (extension) |

## Orchestration Logic

When the user invokes `/seo audit`, delegate to subagents in parallel:
1. Detect business type (SaaS, local, ecommerce, publisher, agency, other)
2. Spawn subagents: seo-technical, seo-content, seo-schema, seo-sitemap, seo-performance, seo-visual, seo-geo
3. Collect results and generate unified report with SEO Health Score (0-100)
4. Create prioritized action plan (Critical → High → Medium → Low)

For individual commands, load the relevant sub-skill directly.

## Industry Detection

Detect business type from homepage signals:
- **SaaS**: pricing page, /features, /integrations, /docs, "free trial", "sign up"
- **Local Service**: phone number, address, service area, "serving [city]", Google Maps embed
- **E-commerce**: /products, /collections, /cart, "add to cart", product schema
- **Publisher**: /blog, /articles, /topics, article schema, author pages, publication dates
- **Agency**: /case-studies, /portfolio, /industries, "our work", client logos

## Quality Gates

Read `references/quality-gates.md` for thin content thresholds per page type.
Hard rules:
- ⚠️ WARNING at 30+ location pages (enforce 60%+ unique content)
- 🛑 HARD STOP at 50+ location pages (require user justification)
- Never recommend HowTo schema (deprecated Sept 2023)
- FAQ schema for Google rich results: only government and healthcare sites (Aug 2023 restriction); existing FAQPage on commercial sites → flag Info priority (not Critical), noting AI/LLM citation benefit; adding new FAQPage → not recommended for Google benefit
- All Core Web Vitals references use INP, never FID

## Reference Files

Load these on-demand as needed — do NOT load all at startup:
- `references/cwv-thresholds.md` — Current Core Web Vitals thresholds and measurement details
- `references/schema-types.md` — All supported schema types with deprecation status
- `references/eeat-framework.md` — E-E-A-T evaluation criteria (Sept 2025 QRG update)
- `references/quality-gates.md` — Content length minimums, uniqueness thresholds

## Scoring Methodology

### SEO Health Score (0-100)
Weighted aggregate of all categories:

| Category | Weight |
|----------|--------|
| Technical SEO | 22% |
| Content Quality | 23% |
| On-Page SEO | 20% |
| Schema / Structured Data | 10% |
| Performance (CWV) | 10% |
| AI Search Readiness | 10% |
| Images | 5% |

### Priority Levels
- **Critical**: Blocks indexing or causes penalties (immediate fix required)
- **High**: Significantly impacts rankings (fix within 1 week)
- **Medium**: Optimization opportunity (fix within 1 month)
- **Low**: Nice to have (backlog)

## Sub-Skills

This skill orchestrates 12 specialized sub-skills (+ 1 extension):

1. **seo-audit** — Full website audit with parallel delegation
2. **seo-page** — Deep single-page analysis
3. **seo-technical** — Technical SEO (8 categories)
4. **seo-content** — E-E-A-T and content quality
5. **seo-schema** — Schema markup detection and generation
6. **seo-images** — Image optimization
7. **seo-sitemap** — Sitemap analysis and generation
8. **seo-geo** — AI Overviews / GEO optimization
9. **seo-plan** — Strategic planning with templates
10. **seo-programmatic** — Programmatic SEO analysis and planning
11. **seo-competitor-pages** — Competitor comparison page generation
12. **seo-hreflang** — Hreflang/i18n SEO audit and generation
13. **seo-dataforseo** — Live SEO data via DataForSEO MCP (extension)

## Subagents

For parallel analysis during audits:
- `seo-technical` — Crawlability, indexability, security, CWV
- `seo-content` — E-E-A-T, readability, thin content
- `seo-schema` — Detection, validation, generation
- `seo-sitemap` — Structure, coverage, quality gates
- `seo-performance` — Core Web Vitals measurement
- `seo-visual` — Screenshots, mobile testing, above-fold
- `seo-geo` — AI crawler access, llms.txt, citability, brand mention signals
- `seo-dataforseo` — Live SERP, keyword, backlink, local SEO data (extension, optional)
