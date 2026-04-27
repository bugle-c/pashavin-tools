# Local Patches Applied

Forked from `google-yandex-seo-skills` (slug from `_meta.json`) on 2026-04-27 for
gptweb.ru. The patches below were applied to the upstream skill — re-apply
manually if you ever pull a newer upstream version.

## 1. Russian definition-lead heuristic (parsers/html-parser.js)

Extended `DEFINITION_LEAD_PATTERN` with Russian definitional verbs so the
"answer-first intro" heuristic recognizes copy like "WebGPT помогает...",
"платформа для...", "это сервис..." instead of only matching "это",
"представляет собой", "означает".

Added: `помогает|позволяет|решает|обеспечивает|включает|содержит|объединяет|единый сервис|это сервис|сервис для|платформа для|инструмент для|combines|provides|includes|enables|service for|platform for`.

## 2. JSON-LD regional signal detection (parsers/html-parser.js)

New function `detectJsonLdRegionSignals(jsonLd)` walks every JSON-LD node
and harvests:
- `address.addressCountry` (string or `{ name }`)
- `address.addressLocality` / `addressRegion`
- `areaServed` (string | object | array)
- `contactPoint.areaServed`

Returns `{ countries, areas, hasJsonLdCountry, hasJsonLdAreaServed }`.

Wired into `parseHtmlPage` → `jsonLdRegionSignals` field on the parsed page.

## 3. Snapshot mapping (lib/index.js)

`region_signals` snapshot now exposes:
- `has_jsonld_country: boolean`
- `has_jsonld_area_served: boolean`
- `jsonld_countries: string[]`
- `jsonld_areas: string[]`

## 4. Yandex regional check (checks/yandex.js)

`yandex-regional-signals` finding now PASSes if EITHER visible signals
(2+ region mentions, LocalBusiness, address mentions) OR JSON-LD signals
(addressCountry, areaServed) are present. Evidence list cites both.

Why: Schema.org regional fields are a strong signal for Yandex even when
the visible page doesn't repeat city/region words.

## How to deploy patches to the active skill location

The skill ALSO lives at `/home/deploy/.claude/skills/indexlift-seo-auditor/`
(the runtime location). After editing here, sync to runtime:

```bash
rsync -a --exclude=node_modules --exclude=deliverables \
  /home/deploy/projects/pashavin-tools/skills/indexlift-seo-auditor/ \
  /home/deploy/.claude/skills/indexlift-seo-auditor/
```

(Or symlink — but rsync is safer if the runtime location is recreated by a
skill reinstall.)

## Cron-side companion

The blog auto-generator (`/home/deploy/projects/ai-aggregator-lobechat/
scripts/blog/seo-audit-post.sh`) calls this auditor after every published
article and alerts via Brevo when score < 80 or FAIL findings > 1. See
`MEMORY.md → seo_automation_pipeline.md` for the full pipeline.
