# pashavin.ru — SEO Standards (project-specific)

Project-specific SEO requirements distilled from the IndexLift audit run on
2026-04-26 / 2026-04-27. Apply these rules whenever generating, editing, or
auditing content for **pashavin.ru** (project `/home/deploy/projects/pashavin.ru`).

When writing, reviewing, or auditing pashavin.ru content, **always check
against this document first**, then fall back to general SEO best practices.

## Title & Description

| Page type | Title length | Notes |
|---|---|---|
| Homepage | 38-65 chars | "Паша Вин — AI-инженер и предприниматель" template |
| `/blog` | 50-65 chars | Include "статьи про AI/автоматизацию/разработку" |
| `/blog/[slug]` | 38-60 chars | Adaptive suffix: long titles drop suffix entirely |
| `/blog/tag/[tag]` | 32-60 chars | Pad short tags ("AI") with "статьи и кейсы" |
| `/blog/author/...` | 45-60 chars | Avoid "автор блога" alone — too short |
| `/projects/[slug]` (full) | 45-60 chars | "{name} — кейс из портфолио Паши Вина" |
| `/projects/[slug]` (gen) | 45-60 chars | "{seoTitle} — {category} Паши Вина" |

**Title↔Description alignment:** must share 2-3 keywords + same intent.
**Title↔H1 alignment:** if H1 is brand-only (visual styling), add `<span class="sr-only">` inside `<h1>` carrying the keyword-rich subtitle.

## Description

- 120-160 chars (Russian Cyrillic)
- Contains primary keyword exactly once
- Contains 1-2 LSI / cluster keywords
- First-person voice ("я", "собрал", "запустил") — at least once
- Concrete number / tool / outcome from body
- No phrase repeated 3+ times

## Heading hierarchy

Strict H1 → H2 → H3 nesting. Never skip a level.

- Section labels (Featured, Tags, Archive on blog index) MUST be `<h2>` not span.
- Each post card title can be `<h3>` (siblings of section h2).
- Inside MDX article: only one H1 (the page title); start sections at H2.

**No repeated headings.** "0", "Auth", "Dashboard" appearing 2+ times = audit flag.

## Question-led structure (audit: question-led-structure, GEO)

For long-form pages and project case studies — at least **60% of H2** in question form:

- "Что такое X?"
- "Как работает Y?"
- "Чем X отличается от Z?"
- "Сколько стоит / экономит?"
- "Почему это важно для бизнеса?"
- "С чего начать?"

Renders as natural FAQ for AI search and Yandex Алиса.

## Direct-answer intro (audit: direct-answer-intro, GEO)

First paragraph (above any H2) must be **50-150 words, definition-style**:

```
[Topic] — это [определение в одной фразе]. [Кому полезно]. [Что даёт].
[Конкретный результат / число].
```

NOT a story-hook, NOT a question, NOT abstract framing. AI-search engines lift
this paragraph as the citation, so it must read as a clean self-contained
definition.

## Schema.org / JSON-LD

Required on every indexable page:

- `WebPage` or `Article` (depending on type)
- `BreadcrumbList` (Главная → Section → Page)

Specific pages:
- `/` — `WebPage` + `FAQPage` + `BreadcrumbList`
- `/blog` — `CollectionPage` + `ItemList` + `BreadcrumbList`
- `/blog/[slug]` — `Article` + `BreadcrumbList` + `Person` author
- `/blog/author/[slug]` — `Person` (with `address`, `knowsAbout`) + `BreadcrumbList`
- `/projects/[slug]` — `WebPage` + `SoftwareApplication` + `BreadcrumbList`
- `/contacts` — `Person` + `BreadcrumbList`
- `/privacy` — `BreadcrumbList`

## Yandex trust signals (audit: yandex-legal-transparency, FAIL)

Pages that look commercial MUST link to:

- `/privacy` — privacy policy
- `/contacts` — contacts page with phone/email/Telegram
- Footer with **legal name** ("Винецкий Павел"), city ("Москва"), email, Telegram

Without these, Yandex caps commercial query rankings even at score 95.

## Freshness signals (audit: freshness-signals)

On long-form / collection pages — show **visible "Обновлено YYYY-MM-DD"** date
within the first viewport. JSON-LD `datePublished` + `dateModified` alone is
not enough.

## Image requirements

Every `<img>`:
- explicit `width` and `height` attributes (CLS guard)
- `loading="lazy"` for non-hero images
- `decoding="async"`
- meaningful `alt` (describes content, not "image")
- `title` attribute can carry SEO value

Default for placeholder: 800×450 (16:9).

## Private / non-indexable pages

Pages NOT meant for SEO must have:

```ts
robots: { index: false, follow: false }
```

Currently: `/strat2026`, `/mm2026` (private TZ + apply landings).

## Automation hooks

The SEO pipeline (`lib/seo/`) automatically applies these standards on:

- New blog posts via `seo-generate-articles.ts` → `writeArticle()` prompt + `styliseArticle()` second pass
- Project regenerations via `lib/seo/content-rewriter.ts` (BLOG_PROMPT, PROJECT_PROMPT)
- Manual SEO runs via `scripts/seo-optimize.ts --target=blog:slug --deep`

Both prompts now reference **"IndexLift аудит pashavin.ru"** explicitly so the
LLM knows to apply these specific rules.

## Verification

After any meaningful change to a template, re-run:

```bash
node /home/deploy/.claude/skills/indexlift-seo-auditor/scripts/run-audit.js \
  --url "https://pashavin.ru/<path>" \
  --tier standard --engines google,yandex \
  --output /home/deploy/projects/pashavin.ru/tasks/seo-spotcheck/
```

Target scores after fixes:
- Homepage / blog index / blog post: **≥95** (A)
- Project pages: **≥94** (A)
- Tag / author / mastermind: **≥93** (A)

Anything below these without a justifying reason = regression.
