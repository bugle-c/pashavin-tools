---
name: pashavin-seo
description: Use when running, tuning, or extending the SEO optimisation pipeline on pashavin.ru — applying /api/seo/* endpoints, rewriting title/description/body for blogs and projects, managing keyword clusters, dashboards at /admin/seo, cron-reoptimize and positions tracking. Triggers on "seo-optimize", "переоптимизация", "кластер ключевых", "business takeaway", or ANY work inside lib/seo/* on pashavin.ru.
---

# pashavin-seo — SEO-first pipeline

Everything you need to operate and extend the custom keyword-research + content-rewrite pipeline built into pashavin.ru. This is NOT a generic SEO skill — for general audits/schema/sitemaps use `seo` / `seo-audit` / `seo-schema`.

## Where it lives

```
lib/seo/
  llm.ts                    # Anthropic SDK wrapper + llmJson (tolerates trailing text, extracts first balanced JSON)
  wordstat-client.ts        # xmlriver client: wordstat() + getPosition() + getCompetition(keyword, frequency)
  yandex-webmaster-client.ts# impressions/queries + pingIndexNow()
  seed-extractor.ts         # LLM distils 3-5 market-shaped seeds from a draft
  cluster-builder.ts        # 2-pass Wordstat expansion + LLM relevance filter + body-first scoring
  content-adapter.ts        # reads MDX frontmatter + projects-meta-cache.json; writes title/description (+ body in deep-mode)
  content-rewriter.ts       # BLOG_PROMPT, PROJECT_PROMPT, rewriteFirstParagraph, generateBusinessTakeaway
  pasha-voice.ts            # PASHA_VOICE_FOR_SHORT_TEXT, PASHA_VOICE_FOR_FIRST_PARA, BUSINESS_TAKEAWAY_PROMPT
  guardrails/
    markdown.ts   # code-fenced blocks stripped before HTML-tag counting
    seo.ts        # Russian-aware stemmed containsNormalized for title/desc/first-para checks
    similarity.ts # LLM-based, resilient (fallback 0.9 on parse fail)
    factual.ts    # catches invented numbers/products; metadata-only by default
    index.ts      # runGuardrails({ bodyChanged }) composes all checks
  snapshot.ts               # takeSnapshot / restoreFromSnapshot — rollback point before every apply
  orchestrator.ts           # runSeoOptimize({ type, slug, dryRun, deep, reason })
  position-tracker.ts       # cron positions tracker, drop >10 → needs_reoptimize
  reoptimizer.ts            # processReoptimizeQueue — picks 5 clusters, runs optimize, freezes on failure
  telegram-seo.ts           # alertApiDown, alertGuardrailsFailed, alertClusterFrozen, sendDigest
  admin-auth.ts             # basic auth for /admin/seo

middleware.ts               # 401 + WWW-Authenticate for /admin/seo and /api/admin/seo
app/admin/seo/              # dashboard + per-cluster detail (card layout, live title, public_url link)
app/api/admin/seo/          # clusters / cluster/[id] / config (kill-switch) / health
app/api/seo/                # cron-positions, cron-reoptimize, send-digest (all Bearer-auth)
scripts/seo-optimize.ts     # CLI: --target=blog:<slug>|project:<slug> [--dry-run] [--deep] [--rollback=<target>]
scripts/seo-regen-takeaway.ts # regenerate ONLY businessTakeaway for a blog (no full pipeline)
sql/pashavin_seo/*.sql      # schema + grants in Supabase schema `pashavin_seo`
```

## Daily commands

```bash
# Dry-run on a blog (preview diff, no writes)
npm run seo:optimize -- --target=blog:<slug> --dry-run

# Apply — writes frontmatter + body (deep), inserts revision row, pings IndexNow
npm run seo:optimize -- --target=blog:<slug> --deep

# Projects — rewrites only name + description (body stays derived from cache)
npm run seo:optimize -- --target=project:<slug>

# Rollback to the latest snapshot
npm run seo:optimize -- --rollback=blog:<slug>

# Regenerate only the businessTakeaway without re-running cluster/rewrite
npx tsx scripts/seo-regen-takeaway.ts --slug=<blog-slug>

# Manual cron triggers (SEO_CRON_SECRET in env)
curl -X POST -H "Authorization: Bearer $SEO_CRON_SECRET" https://pashavin.ru/api/seo/cron-positions
curl -X POST -H "Authorization: Bearer $SEO_CRON_SECRET" https://pashavin.ru/api/seo/cron-reoptimize
curl -X POST -H "Authorization: Bearer $SEO_CRON_SECRET" https://pashavin.ru/api/seo/send-digest
```

Dashboard: https://pashavin.ru/admin/seo — Basic auth, password in `SEO_ADMIN_PASSWORD`.

## Pipeline contract (per run)

1. **Snapshot** original content (blog MDX / project cache entry) → `pashavin_seo.snapshots`
2. **Extract seeds** — 3-5 market-shaped phrases from body via LLM
3. **Build cluster**:
   - 2-pass Wordstat: seeds → top frequencies from pass 1 → more related
   - Webmaster augments with real query impressions
   - LLM relevance filter; fallback to top-by-frequency if filter empties set
   - Competition: freq <100 auto 0.1; otherwise SERP analysis excluding generic portals (Wiki/Dzen/VK/Avito/aggregator patterns)
   - Score = `log(freq) × (1 - competition) × (1 + 0.3 if in body + 0.3 if in first paragraph)` — **body-first selection**
4. **Rewrite** with 2-iteration guardrails loop, feedback fed back on retry:
   - Blog: title + description (`--deep` also rewrites first paragraph)
   - Project: keeps original `name` as brand, adds ` - <seoSuffix>` (1-3 words)
5. **Business takeaway** (blog only): 900-1400 chars, 3 paragraphs — problem + modelled metrics + next step
6. **Guardrails** must all pass: markdown, seo (primary in title/desc/first-para/h1), similarity≥0.85, factual (no invented numbers/products), length-delta (body only when `bodyChanged`)
7. **Apply** (non-dry-run): `writeContent` + `revisions` row + `pingIndexNow` → Яндекс
8. On 2nd failure: `cluster.status = frozen` + Telegram alert

## Rules — do NOT break these

### Keyword selection
- **Body-first**: never pick a primary that has zero stem-match in the blog body. Our scoring rewards it, but if you're manually picking, follow the same principle.
- **Long-tail is fine**: 100-200 imp/mo with low competition is the sweet spot. Don't chase 1000+ keywords with 0.9 competition.
- **Generic portals don't count as competitors**: Wikipedia/Dzen/VK/Avito/Habr/aggregator patterns (otzyv/reiting/sravn/katalog) are excluded in `getCompetition()`.

### Voice (Pasha Vin)
- First person ALWAYS: "я", "мне", "собрал", "попробовал". If not in description → guardrail will still pass but voice is wrong.
- Stop-words are banned: "стоит отметить", "в заключение", "на самом деле", "безусловно", "подводя итоги", "позвольте объяснить", "давайте разберёмся", "революционный", "в мире технологий", "друзья мои".
- Use `-` (hyphen), never em-dash `—`. The prompts enforce this; guardrails don't — if you see em-dash in output, prompt regression.
- Full voice spec in `lib/seo/pasha-voice.ts` — distilled from the author's personal Google Sheet style guide (role / voice / structure / lexicon / formatting / avoid / rules / authenticity / post_types / numbers_and_proof / hooks_bank / rewrite_meta). When updating voice, align with that source.

### Facts — no hallucination
- Rewriter cannot invent numbers/products/names not in body. Enforced in prompts AND by factual guardrail (checks title+desc, and first-paragraph in deep-mode).
- Exception: **business takeaway** explicitly allows modelled metrics ("по опыту похожих внедрений", "на сравнимых кейсах", "типичный эффект"). The marker phrase is what separates modelling from lying.
- Never replace a real number with a different one (487 → 400 is a factual error).

### Blog vs Project
- **Blog**: body is content-first. Default mode only touches title+description. `--deep` rewrites ONLY the first paragraph. Everything else (MDX body) is authorial.
- **Project**: body in `data/projects-meta-cache.json` is structured (description/result/stack/challenge/solution). Pipeline is **metadata-only**.
- **Project: name vs seoTitle** — two SEPARATE fields, do NOT merge:
  - `name` = brand only (e.g. "Arb Scanner"), used by the UI card / h1.
  - `seoTitle` = full SEO string (e.g. "Arb Scanner - арбитраж в ставках"), used ONLY in `generateMetadata` → `<title>` tag.
  - Why split: `<brand> - <suffix> — Паша Вин` template overflows project cards in the UI design.
  - PROJECT_PROMPT asks LLM for `seoSuffix`; `writeContent` splits `"brand - suffix"` back into the two fields and updates both `projects-meta-cache.json` AND `projects-generated.ts`.

### Description rules
- 120-165 chars, enforced by `checkSeo`.
- Primary keyword exactly once (stemmed match), 1-2 LSI from cluster, first-person verb, one concrete number/tool from the article.
- 3+ occurrences of the same primary = keyword stuffing → guardrail fails.

### Business takeaway (BLOG)
- Always 3 paragraphs (`\n\n` separated):
  1. Pain + illustrative kasus (can use article numbers: 487 диалогов, 10 минут)
  2. Modelled metrics with marker ("по опыту похожих внедрений" / "на сравнимых кейсах" / "типичный эффект")
  3. Who + first step
- Minimum 900 chars. `maxTokens: 1500` in the call.
- Rendered via `<BusinessTakeaway text={frontmatter.businessTakeaway} />` — collapsible, with "Обсудить проект" CTA (opens ContactDialog).

### Business impact (PROJECT)
- Always 3 `{value, label}` pairs modelled by "100 entrepreneurs" for the project page "Бизнес Импакт" section.
- `value` — 1-8 chars: number/range/multiplier ("-35%", "+25%", "2×", "1-2 мес", "до 90%").
- `label` — 2-5 words: business KPI ("времени менеджеров", "окупаемость проекта").
- Generated by `generateBusinessImpact()` using `BUSINESS_IMPACT_PROMPT`, stored in `projects-meta-cache.json` AND `projects-generated.ts` under `businessImpact` field.
- UI in `generated-project-detail.tsx`: prefers `businessImpact` over legacy `metrics`, overline reads "Смоделировано 100 предпринимателями". Value font sized `text-5xl md:text-7xl whitespace-nowrap` to keep ranges like "до 90%" on one line.
- Rules in prompt: no tech jargon (TTL/SSE/API/webhook), no em-dash, don't reuse case numbers from description as business KPIs, diverse metrics (not three ways to say "faster").

### Two content patterns — at a glance

| | Blog | Project |
|---|---|---|
| Rewrite scope | title + description (+ first para with `--deep`) | `seoTitle` + `description` + `businessImpact` metrics |
| UI heading source | `title` in frontmatter | `name` (brand only, unchanged by pipeline) |
| `<title>` meta source | same `title` | `seoTitle` field (brand + SEO suffix) |
| Body handling | untouched unless `--deep` (then first paragraph only) | never touched (derived from structured fields) |
| Extra LLM artefact | `businessTakeaway` (3 paragraphs, 900-1400 chars) | `businessImpact` (3 value/label pairs) |
| CTA | "Обсудить проект" inside expanded takeaway | card-wide metric grid + project URL/contact elsewhere |
| Persisted to | `content/blog/<slug>.mdx` frontmatter | `projects-meta-cache.json` **and** `projects-generated.ts` (both via `syncGeneratedProject`) |

## Models and tuning knobs

LLM models and auth:
- `lib/seo/llm.ts` is **hybrid**: Anthropic SDK by default, `claude -p` CLI when `CLAUDE_USE_CLI=1`.
- **Local batch on host**: set `CLAUDE_USE_CLI=1` in `.env.local` → `spawn("claude", ...)` uses OAuth (Max/Pro subscription quota). Good when API balance is empty.
- **Prod Docker container**: CLI is NOT installed. Keep `CLAUDE_USE_CLI` unset → SDK path via `ANTHROPIC_API_KEY` from Dokploy env. Cron endpoints only work when API has credit.
- Strip `ANTHROPIC_API_KEY` from CLI child env (we do this in `runClaudeCli`) so CLI doesn't route through API.
- Default model `claude-sonnet-4-6`; health ping `claude-haiku-4-5` (`app/api/admin/seo/health/route.ts`). Pass `model:` in `LlmCallOptions` per-call to override.

Magic numbers (adjust with care, each has a reason):

| Knob | Value | Where | Why |
|---|---|---|---|
| min frequency filter | `>= 5` | cluster-builder | drops single-impression noise; go to `>= 1` only if Wordstat is starving the cluster |
| 2nd-pass seed threshold | `>= 20` | cluster-builder | which pass-1 keywords seed pass-2 Wordstat calls |
| 2nd-pass seed count | `top 6` | cluster-builder | caps xmlriver quota — 6×20 = 120 extra candidates per run |
| related per seed | `top 20-30` | cluster-builder | |
| scored candidates | `top 30 by freq` | cluster-builder | only these get SERP-competition analysis (cost) |
| final cluster size | `top 15 by score` | cluster-builder | |
| body-alignment bonus | `+30%` body, `+30%` first-para | cluster-builder | pushes primary toward keywords already in content |
| competition auto-threshold | `freq < 100 → 0.1` | wordstat-client getCompetition | long-tail is inherently low-comp |
| competition tier factor | 0.3–1.0 by freq | wordstat-client | scales SERP density impact by query size |
| reoptimize batch | `BATCH_LIMIT = 5` | reoptimizer.ts | LLM + xmlriver cost per run |
| position drop trigger | `>= 10` positions | position-tracker.ts `DROP_THRESHOLD` | queues cluster → `needs_reoptimize` |
| zero-traffic window | `30 days` | position-tracker.ts `ZERO_TRAFFIC_DAYS` | |
| description length | `120-165 chars` | guardrails/seo.ts | Google/Yandex snippet cutoff |
| business takeaway length | `900-1400 chars` | pasha-voice.ts prompt | 3 paragraphs × 300-500 chars |
| takeaway maxTokens | `1500` | content-rewriter.ts | fits 1400 chars + safety margin |
| guardrail iterations | `max 2` | orchestrator.ts loop | then freeze + Telegram alert |
| ToC min headings | `>= 3` | components/blog/table-of-contents.tsx | don't render ToC for tiny posts |

## Env vars (all required, stored in Dokploy)

```
XMLRIVER_USER, XMLRIVER_API_KEY              # wordstat + serp
YANDEX_WEBMASTER_TOKEN, YANDEX_WEBMASTER_USER_ID, YANDEX_WEBMASTER_HOST_ID  # host id format: https:pashavin.ru:443
ANTHROPIC_API_KEY                            # all LLM calls (replaced Voyage — no extra billing)
NEXT_PUBLIC_SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY  # schema: pashavin_seo
SEO_ADMIN_PASSWORD                           # basic auth for /admin/seo (via middleware.ts)
SEO_CRON_SECRET                              # Bearer for /api/seo/cron-*
TELEGRAM_BOT_TOKEN, TELEGRAM_ADMIN_CHAT_ID   # alerts + weekly digest
SEO_ENABLED=true                             # gates generators/integrations — must be set
```

## System cron on VPS (135.181.115.234 / 5.35.80.222)

```cron
0 3 * * 0 curl -sfX POST -H 'Authorization: Bearer <SEO_CRON_SECRET>' https://pashavin.ru/api/seo/cron-positions >> /var/log/seo-cron.log 2>&1
0 2 * * 1 curl -sfX POST -H 'Authorization: Bearer <SEO_CRON_SECRET>' https://pashavin.ru/api/seo/cron-reoptimize >> /var/log/seo-cron.log 2>&1
0 9 * * 0 curl -sfX POST -H 'Authorization: Bearer <SEO_CRON_SECRET>' https://pashavin.ru/api/seo/send-digest >> /var/log/seo-cron.log 2>&1
```

## Schema (Supabase `pashavin_seo`)

- `clusters(content_type, content_slug, seeds, primary_keyword, status)` — status: active | needs_reoptimize | zero_traffic | frozen; unique (content_type, content_slug)
- `keywords(cluster_id, keyword, frequency, competition, source, is_primary)` — source: wordstat | webmaster | llm_seed
- `positions(keyword_id, position, url, checked_at)`
- `revisions(cluster_id, diff, guardrails, published, iteration, created_at)` — written even on guardrails_failed (published=false)
- `snapshots(content_type, content_slug, fields, reason, cluster_id)` — reason: initial | reoptimize | batch_migration | manual
- `config(key, value)` — `frozen` key for kill-switch
- `run_logs(run_id, step, status, ...)` — step: snapshot | seed_extract | keyword_research | rewrite | guardrails | apply | index

PostgREST schema must include `pashavin_seo` in `PGRST_DB_SCHEMAS` (self-hosted Supabase at supabase.pashavin.ru).

## Troubleshooting playbook

| Symptom | Cause | Fix |
|---|---|---|
| "no JSON in: <prose>" | LLM refused (empty/insufficient content) | Check source data; rewriter surfaces as `guardrails_failed`, cluster freezes cleanly. |
| "primary keyword missing in description" | Case/inflection mismatch | `containsNormalized` uses token stemming; if it still fails, primary was genuinely dropped — retry. |
| "length delta 0.00 < 0.8" | LLM returned `"..."` for body | Check bodyChanged flag; for blog default mode body must not be in prompt output. |
| "markdown: unbalanced html tags" | TS generics in code blocks | Already handled — code blocks stripped before counting. If regression, check `guardrails/markdown.ts`. |
| "seo: primary keyword missing in first paragraph" (blog, non-deep) | bodyChanged mis-set | Guardrail skips this check when `bodyChanged=false`. Only apply this error with `--deep`. |
| factual catches invented number | Hallucination during rewrite | Tighten prompt with explicit "NEVER invent numbers" — already in rewriteFirstParagraph. |
| Admin 401 with no auth prompt | WWW-Authenticate header missing | Ensure middleware.ts is in repo root (not app/) and matcher covers `/admin/seo/:path*` + `/api/admin/seo/:path*`. |
| Admin pages show stale data | Client cache | Fetch with `cache: "no-store"` — already set. Hard reload. |
| Deploy successful but env not applied | Dokploy stores env but GHA uses SSH `docker service update --force` | Call Dokploy `application.deploy` API OR `docker service update --env-add ...` manually. |
| Wordstat returns `code: 101, "Сбор старого вордстата больше не доступен"` | Hit deprecated old endpoint | We use `https://xmlriver.com/wordstat/new/json` (constant `BASE_WORDSTAT` in wordstat-client.ts). If xmlriver changes format again: inspect raw JSON (has `popular: [{isAssociations: false, value, text}]` + `associations: [...]`), adjust parser in `wordstat()`. |
| Stem check false-negative (primary present but guardrail fails) | Kept `\b` regex in old stemmer — doesn't match Cyrillic | Current impl in `guardrails/seo.ts` tokenises and stems per-word; don't revert to single-regex stem. |
| LLM returns prose instead of JSON in similarity/factual | Model ignored JSON instruction | Both wrapped in try/catch with safe defaults (sim=0.9, factual=ok). Don't block pipeline on parse fails; log and move on. |
| Project changes in cache but `/projects/<slug>` still shows old title | **Two-file gotcha**: site renders from `data/projects-generated.ts`, NOT from `data/projects-meta-cache.json`. | `writeContent` for projects calls `syncGeneratedProject()` which bracket-matches the `= [` after `generatedProjects` marker, JSON-parses the array, patches the slug, writes back. If you regress this, projects-generated.ts will drift from the cache and live pages won't update. |

## Overlap with other pashavin.ru skills

- `pashavin-portfolio` — project generation from GitHub (`scripts/generate-projects.ts`, screenshots, meta cache).
  **This skill takes over** once the `runSeoOptimize` hook fires inside the generator — the hook is gated by `SEO_ENABLED=true` and uses `await import("../lib/seo/orchestrator")` (lazy, won't break the generator if SEO fails).
- `pashavin-blog` — blog draft generation from Claude Code session logs, secret filtering.
  **This skill takes over** once a blog is published (`--publish` path in `generate-blog.ts`) and again triggers `runSeoOptimize({ type: "blog" })` behind `SEO_ENABLED`.
- `dokploy-deploy` — generic deploy to Dokploy/GHCR/GHA.
  **Use it** for infra changes (Dockerfile, workflow, new app). **This skill owns** the env-var delta inside the existing pashavin.ru app (XMLRIVER_*, YANDEX_WEBMASTER_*, SEO_* etc).
- `supabase` — general Supabase admin.
  **Use it** for schema grants, PGRST_DB_SCHEMAS edits. **This skill owns** the `pashavin_seo` schema tables and their lifecycle.
- `seo` / `seo-audit` / `seo-schema` etc — generic SEO analysis.
  Use those **before** deciding to optimise (audit to identify candidates). This skill runs **after** — the actual optimisation step.

## When to extend vs when not to

**Extend this pipeline** when:
- Adding a new content type (e.g. landing pages) — add to `ContentType` union, `content-adapter` read/write branches, orchestrator.
- Adding a new guardrail — add a file in `lib/seo/guardrails/`, wire in `index.ts`, respect `bodyChanged` flag.
- Tuning voice — edit `lib/seo/pasha-voice.ts`. That's the single source of truth for prompts.

**Don't extend** — use a general skill instead:
- Schema validation → `seo-schema`
- Core Web Vitals → `seo-technical`
- Full site audit → `seo-audit`
- New industry templates → `seo-plan`

## Infrastructure notes

- GHA workflow uses `concurrency: cancel-in-progress: true` — rapid pushes cancel prior builds. Expected behaviour.
- `middleware.ts` MUST be in repo root (not `app/`) — Next.js 16 requirement.
- Body for blog written ONLY when `fields.body !== parsed.content` in content-adapter — protects non-deep runs from accidental body mutation.
- Generators (generate-blog.ts / generate-projects.ts) call `runSeoOptimize` behind `SEO_ENABLED=true` flag — never breaks generator if SEO step fails.
