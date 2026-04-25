---
name: gptweb-seo
description: Use when running SEO/blog work for gptweb.ru — the WebGPT landing + auto-generated blog. Covers the per-category round-robin article generator, canonical/301 consolidation for cannibalization, Yandex Metrika/Webmaster API access, IndexNow ping flow, and the admin UI for plans+blog. Triggers on "gptweb.ru", "WebGPT", "блог gptweb", "тарифы gptweb", "автогенератор блога", "каннибализация gptweb", or ANY work in `ai-aggregator-lobechat` / `webgpt-admin` / `webgpt-landing` directories.
---

# gptweb-seo — SEO & content pipeline for gptweb.ru

Operational knowledge for the WebGPT / gptweb.ru blog. Covers three
repos (`webgpt-landing`, `webgpt-admin`, `ai-aggregator-lobechat`) and
their shared `ai_aggregator` schema in Supabase.

For generic SEO audits use `seo` / `seo-audit` / `seo-schema`. For
pashavin.ru use `pashavin-seo` — it has a totally different pipeline.

## Infrastructure map

```
ask.gptweb.ru           = aggregator (LobeChat fork) + /admin → webgpt-admin
gptweb.ru               = webgpt-landing (Dokploy on VPS #2 5.35.80.222)
supabase.pashavin.ru    = Supabase, schema ai_aggregator
  tables: blog_posts / blog_categories / blog_authors / blog_keywords / plans
```

Single source of truth for **plans/tariffs**: `ai_aggregator.plans`.
Edited from `/admin/finance/plans`, read by both landing and aggregator
through `plans-source.ts` (REST + 60s cache). See memory file
`plans_source_of_truth.md` before touching pricing logic.

Single source of truth for **blog content**: `ai_aggregator.blog_posts`.
Landing renders from there with ISR=3600. Admin CRUD via Supabase
service role from `/admin/blog/*`.

## Blog auto-generator

Lives in `ai-aggregator-lobechat/scripts/blog/`:
- `generate-article.sh` — per-category idempotent round-robin (details below)
- `collect-keywords.sh` — pulls Yandex Webmaster popular queries to `blog_keywords`
- `sync-blog.sh` — legacy importer from blog.chadgpt.ru
- `notify.sh` — Brevo email helper
- `cluster-builder.sh` — Wordstat + LLM relevance + SERP competition → `blog_clusters`
- `wordstat.sh` — xmlriver helpers (`wordstat()` for suggestions, `search_serp()` for top-10 domains)
- `track-positions.sh` — per-URL Metrika traffic drop detector → `reoptimize_queue`
- `reoptimize-article.sh` — Claude CLI rewrites title+meta+first `<p>` for queued posts

Secrets at `/home/deploy/.config/blog-autogen/env` (chmod 600). Contains
`CRON_SECRET`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`,
`BREVO_API_KEY`, `NOTIFY_EMAIL`, `XMLRIVER_USER`, `XMLRIVER_KEY`. Do NOT put these in repo.
Yandex Metrika OAuth token lives separately at `/home/deploy/.config/yandex-metrika/token`.

Systemd units in `/etc/systemd/system/`:
- `blog-generate.timer` — 08,10,12,14,16,18,20,22 MSK (UTC 05/07/09/11/13/15/17/19), `RandomizedDelaySec=2700`, `Persistent=true`
- `blog-keywords.timer` — 03:00 MSK daily
- `blog-sync.timer` — 06:00 MSK daily
- `blog-positions.timer` — 04:00 MSK daily, `RandomizedDelaySec=600`
- `blog-reoptimize.timer` — every 15 min (`OnCalendar=*:0/15`)

Timers enabled; scripts live in the aggregator repo so they get
versioned through GitHub (`bugle-c/ai-aggregator-lobechat` branch `canary`).

### Round-robin algorithm (`generate-article.sh`)

1. Load Supabase creds + CRON_SECRET from env file.
2. Fetch active categories: `blog_categories?is_active=eq.true&order=sort_order`.
3. For each category in order: count posts with `created_at >= today UTC`. First one with 0 is the target. If all have ≥1 → exit clean.
4. Pick a keyword: first try `blog_keywords` filtered by `category_slug=target&status=pending` ordered by priority+impressions. If none → fall back to global `/admin/api/cron/blog-keywords/next`.
5. Dedup guard: pull last 60 titles **in the target category** only, compute Russian/English 4+ char word overlap against the keyword. If ≥80% overlap with any existing title → mark keyword `status=duplicate` directly via Supabase REST and exit.
6. **Cluster build or reuse**: call `cluster-builder.sh <keyword>`. Returns `{cluster_id, related_keywords[]}`. If existing cluster for keyword exists and has `related_keywords`, reuses it; else builds fresh via xmlriver Wordstat + LLM relevance filter + SERP competition scoring.
7. Build the prompt with:
   - `CRITICAL BRAND RULES` block (WebGPT always, never WeGPT/Web GPT/Wegpt)
   - `REQUIRED CATEGORY: <slug> — <name>` + category-specific shape guide
   - `EXISTING PUBLISHED ARTICLES IN THIS CATEGORY` list (to avoid angle duplication)
   - `RELATED LONG-TAILS` block from cluster's `related_keywords` (10-15 phrases)
   - Forces `"category": "<target>"` in the JSON output
8. Call `/home/deploy/.local/bin/claude --print -p "$PROMPT" --output-format json` with 480s timeout. `unset CLAUDECODE` before the call — Claude CLI refuses to run inside an active Claude Code session otherwise.
9. Parse `data.result` — strip markdown code fences, extract first `{...}`, parse JSON, override `category` to target (defense-in-depth against LLM ignoring instruction).
10. POST to `/admin/api/cron/blog-generate` with `auto_publish=true&keyword_id=...&cluster_id=...`. Endpoint inserts as `status=published`, `published_at=now()`, marks cluster `status=used`, pings IndexNow, sends Brevo "Auto-published" email. Returns `{id, slug, status, category}`.

### Cluster builder (`cluster-builder.sh`)

Builds `ai_aggregator.blog_clusters` rows with 10-15 long-tail keywords per seed.

1. Fetch Wordstat suggestions via xmlriver (`/wordstat/new/json?user=&key=&query=`).
2. **Response shape (actual, not what docs say):** top-level `associations[]` + `popular[]`, each `{text, value, isAssociations}`. `value` is a STRING — coerce to int. NOT `content.including/related[]` with `{phrase, number}` as older plans assumed.
3. **Short-seed retry:** if seed has ≥4 words and Wordstat returns <5 candidates, `shorten_seed()` drops stopwords (для/как/что/это/...) and retries once. Prevents the "`как лучший ai чат бот для бизнеса в россии`" → 0 suggestions death-spiral.
4. LLM relevance filter: Claude CLI rates each candidate 0-1 for topical fit, cuts <0.6.
5. SERP competition via `get_competition()`: xmlriver Search XML top-10, counts generic-portal domains (habr.com, vc.ru, medium.com, dzen.ru, pikabu.ru, ...) via `is_generic_domain()`, returns count/10 as `avg_competition` (0-1). Lower = more room for ranking.
6. Inserts `{seed, status='pending', related_keywords text[], avg_competition numeric, avg_impressions numeric}`.

### Position tracker (`track-positions.sh`)

Per-URL traffic drop detector via Yandex Metrika. **Not Webmaster** (which returns RESOURCE_NOT_FOUND on per-URL endpoints).

1. Token pre-flight: GET `/management/v1/counters` to fail fast if expired.
2. Two windows via `/stat/v1/data`:
   - Current: `date1=7daysAgo&date2=yesterday`
   - Baseline: `date1=30daysAgo&date2=8daysAgo`
3. Filter: `ym:s:startURLPath=~'^/blog/' AND ym:s:trafficSourceID==1` (organic only — excludes direct/referral/social).
4. Metric: `ym:s:pageviews` + `ym:s:visits`. **NOT `ym:pv:pageviews`** — returns `invalid_parameter (4011)` when combined with session-scope dimension `ym:s:startURLPath`.
5. `accuracy=low` — MANDATORY for 30-day windows, else "Query is too complicated".
6. For each URL: compute visits/day for both windows. Drop criterion: `cur_visits_per_day < baseline_visits_per_day × 0.5 AND baseline ≥ 1 visit/day`. (Excludes never-trafficked URLs from noise.)
7. Resolve URL → post_id via Supabase. Insert into `reoptimize_queue(post_id, reason, snapshot_date, status='pending')` — idempotent by `(post_id, status=pending)` unique filter.
8. Also writes raw snapshot to `blog_positions(post_id, snapshot_date, impressions=pageviews, clicks=visits, avg_position=NULL)` — column names repurposed from the original Webmaster-based design. Don't rename columns; downstream admin UI reads them as-is.

### Auto-rewrite (`reoptimize-article.sh`)

Processes one `reoptimize_queue` row per invocation (every 15 min via timer). Targets **title + meta_description + first `<p>` only** — body content is NEVER rewritten (preserves indexed depth).

1. `--next` flag: pick oldest `pending` row. Or pass specific `post_id` to rewrite on demand.
2. Flip queue row to `status=in_progress` (lock via Supabase REST atomic update).
3. Fetch post content + top 10 Webmaster queries for that URL (highest TOTAL_SHOWS).
4. Claude CLI rewrites title/meta/first paragraph with "target queries in H1 + lead paragraph, keep tone" instruction.
5. PATCH direct to Supabase REST `blog_posts?id=eq.<id>` with service role. **NOT the admin PUT endpoint** — admin PUT requires Better Auth cookie which shell scripts can't provide cleanly.
6. On success: flip queue row to `status=done` with `notes='auto-rewritten <timestamp>'`.
7. On ANY failure (Claude CLI timeout, JSON parse, Supabase 4xx): revert queue row to `status=pending` with notes. Never leaves in `in_progress`.
8. IndexNow ping NOT triggered from shell — next admin save or the next `/api/cron/blog-reindex` sweep handles it. (Avoids duplicating `lib/indexing.ts` logic.)

### Category-specific prompt hints
(in `generate-article.sh`, propagated via `REQUIRED CATEGORY` block)

- reviews: обзор возможностей, плюсы/минусы, кому подойдёт
- prompts: конкретные промпты и шаблоны с объяснением
- news: свежие события и их анализ
- cases: реальные кейсы использования с результатами
- guides: пошаговая инструкция "как сделать X"
- business: применение в продажах/маркетинге/HR с цифрами и ROI
- education: для школьников, студентов, абитуриентов, рефератов, дипломов

## Cannibalization consolidation (canonical + 301)

The blog had severe cannibalization before 2026-04-23 — multiple
articles ranking for the same intent, Yandex rotating them, none
getting to top-3. Fixed with two mechanisms:

**For archived dupes → 308 redirect.** `blog_posts.status='archived'` +
`canonical_url` (or fallback `index_status.canonical`) set. Landing's
`/blog/[cat]/[slug]/page.tsx` looks up `getArchivedRedirect(slug)` and
calls `permanentRedirect()`. URL stays out of 404 territory, Yandex
reassigns authority to the target over 2-4 weeks.

**For live dupes → rel=canonical.** `blog_posts.canonical_url` column
holds the `/blog/...` path to the cluster leader. Landing's
`generateMetadata()` reads the field and emits `<link rel="canonical">`.
Admin form has a "Canonical URL" input in the SEO collapsible. PUT
`/api/blog/posts` whitelists `canonical_url` and pings IndexNow on
every change (both the row's own URL and the new canonical target).

**DB columns added during cleanup:**
- `blog_posts.canonical_url text NULL` — writable per-article override
- `blog_posts.status` CHECK updated to include `archived`
- `blog_categories.is_active boolean DEFAULT true` — for category gating
- `blog_keywords.category_slug text NULL` — optional hint so manual
  seeds route to the right category

When setting a new canonical: verify the TARGET is the actual traffic
leader, not just the most recent article. Check Metrika pageviews per
URL before assigning — I wasted one round assigning canonical to a
newer article before realizing the older one outranked it.

## Yandex Metrika access

Counter for gptweb.ru: `106801684`. 31 other counters also accessible
on the same token (pashavin.ru, x10seo.ru, germanyun.ru, etc. — see
`management/v1/counters`).

OAuth creds in memory file `yandex_metrika_access.md`. Token at
`/home/deploy/.config/yandex-metrika/token` (60 days TTL by default,
refresh via device flow when expired).

**Device flow refresh:**
```bash
curl -X POST https://oauth.yandex.ru/device/code -d "client_id=<id>"
# → device_code, user_code, verification_url=https://ya.ru/device
# User visits URL, enters user_code
# Poll until we get access_token:
curl -X POST https://oauth.yandex.ru/token \
  -d "grant_type=device_code&code=<device_code>&client_id=...&client_secret=..."
```

**Per-URL page-view query (hit-scope, works across any date window):**
```bash
TOKEN=$(cat /home/deploy/.config/yandex-metrika/token)
curl -s -G "https://api-metrika.yandex.net/stat/v1/data" \
  --data-urlencode "ids=106801684" \
  --data-urlencode "metrics=ym:pv:pageviews" \
  --data-urlencode "dimensions=ym:pv:URLPath" \
  --data-urlencode "filters=ym:pv:URLPath=~'^/blog/'" \
  --data-urlencode "date1=30daysAgo" --data-urlencode "date2=yesterday" \
  --data-urlencode "limit=300" --data-urlencode "sort=-ym:pv:pageviews" \
  --data-urlencode "accuracy=low" \
  -H "Authorization: OAuth ${TOKEN}"
```

**Per-URL organic-traffic query (session-scope, what `track-positions.sh` uses):**
```bash
curl -s -G "https://api-metrika.yandex.net/stat/v1/data" \
  --data-urlencode "ids=106801684" \
  --data-urlencode "metrics=ym:s:pageviews,ym:s:visits" \
  --data-urlencode "dimensions=ym:s:startURLPath" \
  --data-urlencode "filters=ym:s:startURLPath=~'^/blog/' AND ym:s:trafficSourceID==1" \
  --data-urlencode "date1=7daysAgo" --data-urlencode "date2=yesterday" \
  --data-urlencode "accuracy=low" \
  -H "Authorization: OAuth ${TOKEN}"
```

**Pitfall:** `accuracy=low` is MANDATORY for multi-dimensional queries
on 30+ day windows — otherwise Metrika returns `{"error_type":
"query_error", "message": "Query is too complicated"}`.

**Pitfall:** mixing scopes returns `invalid_parameter (4011)`. Metric
`ym:pv:pageviews` (hit-scope) CANNOT pair with dimension
`ym:s:startURLPath` (session-scope) — use `ym:s:pageviews` when
filtering by session attributes like `trafficSourceID`.

**Pitfall:** `ym:s:searchPhrase` returns `null` for ~90% of Yandex
traffic (not-provided). Per-URL query attribution is effectively
impossible from Metrika alone. Use Webmaster API for query data, and
assume heuristic mapping.

**Pitfall:** `trafficSourceID` values — `1=organic`, `2=direct`,
`3=referral`, `4=internal`, `5=ad`, `6=email`, `7=social`, `8=saved`,
`9=recommendation`. Filtering `==1` gives only search traffic (what we
want for ranking-drop detection).

## Yandex Webmaster access

Token in `/opt/lobechat/.env` as `YANDEX_WEBMASTER_TOKEN`. User id
`187169407`, host id `https:gptweb.ru:443`.

Top query clicks (what gptweb.ru actually ranks for):
```bash
TOKEN="<YANDEX_WEBMASTER_TOKEN>"; USER="187169407"; HOST="https:gptweb.ru:443"
curl -s "https://api.webmaster.yandex.net/v4/user/${USER}/hosts/${HOST}/search-queries/popular/?query_indicator=TOTAL_CLICKS&query_indicator=TOTAL_SHOWS&order_by=TOTAL_CLICKS&limit=500" \
  -H "Authorization: OAuth ${TOKEN}"
```

SERP-sample URLs (what Yandex actually shows in search):
```bash
curl -s "https://api.webmaster.yandex.net/v4/user/${USER}/hosts/${HOST}/search-urls/in-search/samples/" \
  -H "Authorization: OAuth ${TOKEN}"
# Note: DO NOT add ?limit=— it returns 0 samples. Works only without it.
```

**Pitfall:** Webmaster's per-URL stats endpoint `/search-urls/popular/`
and per-query URL endpoint `/search-queries/{id}/` both return
`RESOURCE_NOT_FOUND`. We cannot get per-URL clicks directly. Workaround
= use Metrika `ym:pv:URLPath` views + Webmaster query list and map by
semantic overlap.

## IndexNow ping flow

`webgpt-admin/lib/indexing.ts` exports `submitToIndex(url)` which hits
both IndexNow (`yandex.com/indexnow`) and Yandex Webmaster recrawl API
in parallel. Requires `INDEXNOW_KEY` (key file at
`gptweb.ru/<key>.txt`, hosted in `webgpt-landing/public/`) and
`YANDEX_WEBMASTER_TOKEN/USER_ID/HOST_ID`.

**Automatic ping triggers:**
- POST `/api/blog/posts` with status=published → ping
- PUT `/api/blog/posts` with status→published or status→archived → ping
- PUT with canonical_url changed → ping BOTH own URL and canonical target
- POST `/api/cron/blog-generate` with auto_publish=true → ping

Results saved to `blog_posts.index_status.pings` as an array.

## Admin UI entry points

All at `https://ask.gptweb.ru/admin` (Better Auth session, allowed
emails from `ADMIN_EMAILS` env).

- `/admin/finance/plans` — tariff CRUD, single source of truth. Toggle
  is_active, reorder via landing_sort_order, edit features array as
  textarea (one feature per line).
- `/admin/finance/models` — model_rates CRUD: edit markup, pricing_unit
  (tokens/image/second), input/output per 1M, per_unit. Cost preview
  widget. Switching pricing_unit nulls stale fields (DB CHECK enforces
  mutual exclusion).
- `/admin/finance/api-costs` — invoice-vs-booked audit for API
  providers. Month×provider table: `invoiced_usd` (from
  `manual_expenses` `category='api'`) vs `booked_usd` (from
  `usage_logs.cost_usd` mapped via `lib/provider-mapping.ts`:
  anthropic/openai direct, everything else → openrouter). Δ% red at
  >20% — signal that `writeUsageLog` is dropping rows. Summary shows
  gross margin %. Chart: stacked bar invoice vs revenue + margin line.
- `/admin/blog/[id]` — article editor with Canonical URL input in the
  SEO collapsible. Blank = self-canonical; fill `/blog/...` to point to
  a dup leader.
- `/admin/blog` — list + batch filters.
- `/admin/blog/reoptimize` — `reoptimize_queue` browser. Shows pending
  rows with drop reason, baseline vs current visits, status buttons
  (dismiss / re-queue / open article).
- `/admin/blog/clusters` — read-only `blog_clusters` browser with
  related_keywords, avg_competition, status.

## Light/Pro UI modes (end-user side)

Aggregator at `ask.gptweb.ru/chat` has a per-user runtime UI toggle in
top-bar: `[ ✨ Light │ ⚙️ Pro ]`. Persisted in
`user_onboarding.ui_mode ∈ {'light','pro'}` (default `'light'` for all
users). See memory file `light_pro_modes.md` for full architecture.

**SEO-relevant:** the upstream LobeChat onboarding flow at `/onboarding`
is fully BYPASSED — `useWebUserStateRedirect` is no-op; `/onboarding`
hard-redirects to `/chat`. Welcome modal in chat handles new-user UX
instead. Don't link to `/onboarding` in SEO copy or marketing.

Pricing/upgrade buttons should always link to **`/settings/plans`**
(NOT `/settings/subscription/plans` — that's a 404 in our build).

## Common tasks

### Add a new category
```sql
INSERT INTO ai_aggregator.blog_categories (slug, name, description, sort_order, is_active)
VALUES ('new-slug', 'Человеческое имя', 'описание', <next_sort_order>, true);
```
Then add a matching category-specific prompt hint to the `REQUIRED CATEGORY`
block in `generate-article.sh` and push to `canary`. No deploy needed
(scripts run from filesystem).

### Disable a category temporarily
```sql
UPDATE ai_aggregator.blog_categories SET is_active=false WHERE slug='<slug>';
```
Script skips it automatically next fire.

### Consolidate a newly-found duplicate cluster
1. Query Metrika for 30/90d pageviews per URL in the cluster. Identify
   the leader by TRAFFIC, not recency.
2. For secondary URLs with <20% of leader's traffic: set
   `canonical_url = '/blog/.../leader-slug'`. They keep serving, rel=canonical
   tells Yandex to consolidate.
3. For URLs with 0 traffic in 90d: set `status='archived'` +
   `index_status.canonical` to the leader. Landing returns 308.
4. PUT triggers IndexNow automatically. Expect effect in 2-4 weeks.

### Add manual high-priority keywords
```sql
INSERT INTO ai_aggregator.blog_keywords (keyword, source, priority, status, category_slug)
VALUES
  ('пример ключевика', 'manual', 'high', 'pending', 'guides'),
  ('другой ключевик', 'manual', 'high', 'pending', 'business');
```
Source must be one of `yandex_api | manual | ai_generated` (CHECK
constraint). `category_slug` is optional; if set, generator will prefer
this keyword when filling that category.

`/admin/api/cron/blog-keywords/next` now returns one candidate per
priority bucket in order `high > medium > low`, so manual seeds
always fire before the yandex_api queue.

### Manually trigger reoptimization for a specific post
```bash
cd /home/deploy/projects/ai-aggregator-lobechat
./scripts/blog/reoptimize-article.sh <post_uuid>
# Or pick next pending from queue:
./scripts/blog/reoptimize-article.sh --next
```
Rewrites title + meta_description + first `<p>` only. Body untouched.

### Run position-drop check on demand
```bash
cd /home/deploy/projects/ai-aggregator-lobechat
./scripts/blog/track-positions.sh
# Populates blog_positions (raw) + reoptimize_queue (drops ≥50%)
```

### Build cluster for a specific keyword
```bash
cd /home/deploy/projects/ai-aggregator-lobechat
./scripts/blog/cluster-builder.sh "chatgpt для школьников"
# Prints cluster_id + related_keywords JSON
# Inserts into ai_aggregator.blog_clusters if not already present
```

## Pitfalls (learned the hard way)

- **NEVER create `/home/deploy/projects/ai-aggregator/`** — automated
  cleanup once classified it as a duplicate of `ai-aggregator-lobechat`
  and moved it to `_archive/`, breaking the blog pipeline for hours.
  Scripts live inside `ai-aggregator-lobechat/scripts/blog/` exactly
  for this reason.
- **Landing has had transient `EAI_AGAIN supabase.pashavin.ru`** from
  Hetzner internal DNS. Retry wrapper is in `lib/supabase.ts` — 3
  attempts on EAI_AGAIN/ENOTFOUND/ECONNRESET/ETIMEDOUT/fetch-failed.
  If you add new Supabase calls to landing, use the existing `supabase`
  proxy; don't create a new client instance without the retry wrapper.
- **Never 404 an archived post** — always 308 to canonical. Yandex is
  far slower at de-indexing a 404 than at following a redirect; 404
  bleeds rankings without consolidating them.
- **Brand is WebGPT (capital W, lowercase eb, capital GPT).** Yandex
  Wordstat has "wegpt ru" and "вебгпт" queries — when one lands in
  the keyword queue, the LLM used to invent a product called WeGPT.
  The `CRITICAL BRAND RULES` block in the prompt now pre-empts this.
- **`category_slug` on `blog_keywords` is independent of Yandex-origin
  keywords.** They come with `category_slug=null`. If you want Yandex
  keywords to route to specific categories, you'd need a classifier —
  currently generator just forces category via prompt + overrides LLM
  output, which works fine.
- **xmlriver Wordstat response shape** is top-level `associations[]` +
  `popular[]` with `{text, value, isAssociations}` (value is STRING,
  coerce to int). NOT `content.including/related[]` with
  `{phrase, number}` as some older plan drafts assumed. Check the actual
  response before writing a parser.
- **`ym:pv:pageviews` won't work with session-scope dimensions.**
  Metrika's 4011 error is uninformative — if you filter by
  `ym:s:trafficSourceID` or group by `ym:s:startURLPath`, use
  `ym:s:pageviews` (session-scope) not `ym:pv:pageviews` (hit-scope).
- **Shell scripts can't call admin PUT endpoints** — those require
  Better Auth cookie. For automation (reoptimize, bulk ops) write
  directly to Supabase REST with service role + `Accept-Profile:
  ai_aggregator`. Don't try to fake a session cookie.
- **`reoptimize_queue` idempotency** — `track-positions.sh` must check
  for existing `pending` rows before inserting or the queue fills with
  duplicates on every daily run. Use `WHERE post_id=X AND status IN
  ('pending','in_progress')` filter before insert.
- **Body is SACRED.** `reoptimize-article.sh` only rewrites title +
  meta_description + first `<p>`. Never rewrite body on automated
  re-optimization — that's how you lose indexed anchor text and
  internal link structure. If body needs full rewrite, that's a manual
  `/admin/blog/[id]` edit.

## Git / deploy

- `webgpt-landing`: `bugle-c/webgpt-landing` branch `main` → GHA → GHCR → Dokploy. Push to main = live in ~3 min.
- `webgpt-admin`: `bugle-c/webgpt-admin` branch `master` → manual build on VPS #1 via `docker build -t webgpt-admin:latest . && docker compose up -d --force-recreate webgpt-admin`. No GHA pipeline yet.
- `ai-aggregator-lobechat`: `bugle-c/ai-aggregator-lobechat` branch `canary`. Scripts execute directly from checkout (systemd points at filesystem path) — push effects next timer fire, no build/deploy for script changes. For aggregator code changes, rebuild `lobechat-custom:latest` and `docker compose up -d --force-recreate lobe` on VPS #1.

Author identity for commits: `pasha <2396741@gmail.com>` (use inline
`-c user.name=pasha -c user.email=2396741@gmail.com` per CLAUDE.md rule
about not updating git config).
