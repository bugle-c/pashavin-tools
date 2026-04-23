---
name: pashavin-portfolio
description: Use when working with pashavin.ru portfolio site - adding/updating projects, fixing screenshots, regenerating metadata, troubleshooting the generation pipeline, changing project pages design, or when user says "портфолио", "pashavin.ru", "проекты", "скриншоты". MUST be used for ANY work in the pashavin.ru project directory.
---

# pashavin.ru Portfolio Pipeline

## Overview

Portfolio site with dual-source project system: manual flagship projects (full case studies) + auto-generated projects (Claude API from GitHub repos). Automated screenshot capture with auth, AI illustrations for projects without URLs.

## Quick Reference

| Item | Value |
|------|-------|
| Project dir | `/home/deploy/projects/pashavin.ru` |
| Generate script | `scripts/generate-projects.ts` |
| Manual projects | `data/projects.ts` (Priority 1) |
| Auto-generated | `data/projects-generated.ts` (Priority 2) |
| Cache | `data/projects-meta-cache.json` |
| Screenshots | `public/screenshots/{slug}.jpg` |
| Design | Brutalist dark: bg `#0a0a0a`, accent `#FF4F00`, font Syne |
| Dokploy appId | `b5to9etG1A6H3yvlb2iVN` |
| Domain | `https://pashavin.ru` |

## Commands

```bash
# Full generation (cached, incremental)
npx tsx scripts/generate-projects.ts

# Force-refresh single project
npx tsx scripts/generate-projects.ts --refresh=<slug-or-repo-name>

# Build & verify
npm run build
```

## Data Priority System

```
Manual project (data/projects.ts) → ProjectDetailContent (full case study)
         ↓ not found
Generated project (data/projects-generated.ts) → GeneratedProjectDetail
         ↓ not found
404
```

**Manual projects** have `CaseStudy` with editorial control: brief, challenge, solution, techNodes, metrics, marqueeItems.

**Generated projects** have same structure but auto-created by Claude API. Same visual template but simplified component.

## Generation Pipeline

### Step-by-step flow

1. **Fetch repos** from GitHub org `bugle-c` via `gh` CLI
2. **Filter** through `SKIP_REPOS` (docs, infra, archived)
3. **For each repo**, check in order:
   - Priority 1: `MULTI_SERVICE_META[repoName]` (manual multi-service)
   - Priority 2: `PROJECT_META[repoName]` (manual single override)
   - Priority 3: Valid cache entry (< 7 days AND no new commits)
   - Priority 4: Generate via Claude Sonnet 4.6 API
   - Fallback: Minimal default if API fails
4. **Capture screenshots** for projects with `url`
5. **Generate AI illustrations** for projects without `url`
6. **Write** `data/projects-generated.ts`

### Cache rules

- **TTL**: 7 days, then check GitHub for new commits
- **Staleness**: `commitDate > generatedAt` = stale, regenerate
- **On refresh**: preserves `url` AND `slug` from old cache (Claude API doesn't know deploy URLs!)
- **Slug pinning**: Once a slug is established in cache, it NEVER changes. On regeneration, if Claude API returns a different slug, the old one is forced back. This protects SEO (backlinks, search rankings, social shares).
- **Format**: `{ _generatedAt: ISO, data: ProjectMeta | ProjectMeta[] }`

### SEO slug protection

**CRITICAL**: Slugs are permanent. They form URLs like `/projects/{slug}` that get indexed by search engines, shared on social media, and linked from external sites. Changing a slug = broken links = lost SEO.

The script enforces this automatically via `preservedSlugs`:
- On `--refresh` or stale cache: old slugs saved before deletion
- After Claude API regeneration: new slugs compared to old, forced back if different
- Warning logged: `⚠ Slug pinned: "new-slug" → "old-slug" (SEO protection)`
- Exception: if service count changes (multi-service split/merge), slugs are NOT pinned and manual review is needed

## Screenshot Capture

### Decision tree

```
Has url? ─── No ──→ AI illustration (WaveSpeed)
   │
  Yes
   │
Screenshot exists & fresh? ─── Yes ──→ Skip
   │
   No
   │
Has auth config? ─── Yes ──→ performAuth() → capture
   │
   No
   │
Navigate → login page detected? ─── Yes ──→ Skip with warning
   │
   No
   │
Capture screenshot (1280x800, JPEG q85, 5s wait)
```

### Auth types

| Type | How it works | Projects |
|------|-------------|----------|
| `cookie-login` | POST credentials to login endpoint | crm-messenger, content-factory-crm |
| `password-gate` | Find password input, submit | cjm-builder, kp-generator |
| `jwt-generate` | Sign JWT, set Authorization header | conference-leads-collector |
| `cookie-preset` | Set cookie before navigation | tg-army, webgpt, webgpt-admin |
| `form-login` | Fill email/password in DOM form | twenty-crm |

### Auth credentials location

All credentials are in `getAuthConfig()` inside `scripts/generate-projects.ts`. When adding new auth:
1. Find credentials in project's `.env`, database, or Docker containers
2. Create a dedicated `screenshot-bot` user (NEVER use production admin accounts)
3. Add to `getAuthConfig()` with appropriate type
4. Test single project: `--refresh=<slug>`

### Auth config key = project slug (NOT repo name)

**CRITICAL**: The key in `getAuthConfig()` must match the project's `slug` in cache, NOT the GitHub repo name. Multi-service repos generate slugs that differ from the repo name (e.g., repo `v0-german` → slugs `cjm-builder`, `kp-generator`, `sales-analyzer`, `20-files`).

### Creating screenshot-bot users for auth projects

When a project is behind login, create a REAL user for screenshots — NEVER generate AI illustrations for projects with URLs.

**Process by auth system:**

| Auth system | How to create user |
|-------------|-------------------|
| Better Auth | `curl -X POST https://{domain}/api/auth/sign-up/email -H "Content-Type: application/json" -d '{"email":"screenshot-bot@pashavin.ru","password":"screenshot-bot-2026","name":"Screenshot Bot"}'` |
| Supabase Auth | Create user via Supabase dashboard or `supabase.auth.admin.createUser()` |
| Custom (password-gate) | Use existing shared password from project env |
| Custom (form-login) | Create user in project's DB directly |

**After creating user:**
1. Sign in to get session token/cookie
2. Add to `getAuthConfig()` with correct slug key
3. For `__Secure-` cookies: set `secure: true` and `sameSite: "Lax"` (script does this automatically)
4. For heavy SPAs: add `waitMs: 10000` to give the app time to load after auth
5. Test with `--refresh=<slug>`, verify screenshot is real UI

### PII protection

Before screenshot capture, script hides elements matching:
`[class*="user"], [class*="email"], .user-data, .profile` etc.

**CRITICAL**: Never expose real user data, passwords, or tokens in screenshots.

## AI Illustrations

### When generated

- Project has NO `url` (no live deployment)
- Uses WaveSpeed Nano Banana 2 API (`WAVESPEED_API_KEY` in `.env.local`)
- Prompt in Russian, category-aware (Bot/SaaS/Tool/Landing)
- Saved to same `public/screenshots/{slug}.jpg` path

### Category visuals

| Category | Visual |
|----------|--------|
| Bot | Telegram bot interface with chat bubbles |
| SaaS | Modern dashboard with data visualizations |
| Tool | Developer tool with code editor |
| Landing | Modern landing with hero section |

## Adding a New Project

### Automatic (most common)

1. Push repo to `bugle-c` GitHub org
2. Run `npx tsx scripts/generate-projects.ts`
3. Script auto-discovers, generates metadata via Claude API
4. If project has a live URL, add it manually to cache:
   ```bash
   # Edit data/projects-meta-cache.json, add "url": "https://..." to the project's data
   ```
5. Re-run script to capture screenshot
6. Build, commit, deploy

### Manual flagship project

1. Add `CaseStudy` entry to `data/projects.ts`
2. Ensure screenshot exists in `public/screenshots/{slug}.jpg`
3. Manual projects override generated ones (same slug = manual wins)

## Updating an Existing Project

### Refresh metadata (project code changed significantly)

```bash
npx tsx scripts/generate-projects.ts --refresh=<slug>
```

This preserves the URL from cache. Always verify after refresh:
- Check `data/projects-generated.ts` for correct description
- Check screenshot was re-captured (not replaced with AI illustration)
- Run `npm run build` to verify

### Fix broken screenshot

1. Delete: `rm public/screenshots/<slug>.jpg`
2. Check auth config exists in `getAuthConfig()` if behind login
3. Re-run: `npx tsx scripts/generate-projects.ts`
4. Verify screenshot is real page, not login screen

### Add URL to project without one

1. Edit `data/projects-meta-cache.json`
2. Add `"url": "https://..."` inside the project's `data` object
3. Delete old illustration: `rm public/screenshots/<slug>.jpg`
4. Re-run script to capture real screenshot

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| AI illustration instead of screenshot | URL missing from cache | Add `url` to `projects-meta-cache.json`, delete old image, re-run |
| URL lost after `--refresh` | Claude API doesn't know deploy URLs | Fixed: script now preserves URLs. If still lost, add manually to cache |
| Screenshot shows login page | No auth config for project | Add auth to `getAuthConfig()`, find credentials in project's env/DB |
| Metrics text broken (line breaks) | `\n` in metric labels | Labels cleaned with `.replace(/\n/g, " ")` in components |
| Wrong screenshot (stale) | Cache considers it fresh | Use `--refresh=<slug>` to force re-capture |
| Build fails after generation | TypeScript errors in generated data | Check `data/projects-generated.ts` for malformed entries |
| Duplicate project (manual + generated) | Same slug in both sources | Manual wins. Remove from `data/projects.ts` if generated version is better |
| conference-leads-collector auth fails via HTTPS | Traefik strips Authorization headers | Uses `localUrl: "http://127.0.0.1:8080"` |

## Design System

### Brutalist dark theme

- Background: `#0a0a0a`
- Accent: `#FF4F00` (orange)
- Text: white (`text-white`) + gray (`text-gray-400`, `text-gray-600`)
- Font display: Syne (font-display)
- Font body: system sans-serif

### Project page sections (GeneratedProjectDetail)

1. **Hero** — category label, project name (split words), metadata grid
2. **Tech marquee** — scrolling orange banner with stack (repeated 4x)
3. **Brief + BrowserMockup** — narrative hook + 3D screenshot frame
4. **Challenge vs Solution** — two-column with bullet points
5. **Architecture mindmap** — 4 techNodes in 3x3 grid with SVG connections
6. **Metrics** — 3-column with large values + labels
7. **CTA** — "Open project" button (if URL exists)
8. **Next project** — orange footer with arrow animation

### BrowserMockup component

- macOS-style browser chrome (orange dots)
- 3D tilt on mousemove: `perspective(1000px) rotateX/Y(max 4deg)`
- Screenshot via Next.js `Image` (lazy loading)

## URL Button Rules

**"Open project" button** (`publicUrl`) visibility:
- **Show**: Landing pages, public SaaS products (anyone can visit)
- **Hide**: CRM, admin panels, internal tools, anything behind auth
- Script auto-sets `publicUrl: false` for projects with `getAuthConfig()`
- Override with explicit `publicUrl: true/false` in project data

## Metrics Quality Rules

Metrics must show **REAL business results**, NOT technical facts:
- GOOD: `"0" + "ручной работы"`, `"10x" + "быстрее"`, `"3ч→5мин" + "генерация"`
- BAD: `"REST" + "эндпоинтов"`, `"AI" + "провайдера"`, `"4" + "модуля"`

If the prompt generates bad metrics, update the prompt in `scripts/generate-projects.ts`.

## Self-Learning System

**REQUIRED**: Read `LESSONS.md` in this skill directory BEFORE any pipeline work.

After every issue or improvement:
1. Read current `LESSONS.md`
2. Add new rule/pattern to appropriate section
3. If it's a recurring issue → update the prompt, component, or script to prevent it
4. If it changes workflow → update this SKILL.md too

The goal: the same mistake never happens twice. Every fix should be both:
- **Immediate** (fix the code/data now)
- **Preventive** (update LESSONS.md + prompt/logic so it can't recur)

## Verification Checklist

Before deploying portfolio changes:

- [ ] `npm run build` passes
- [ ] Screenshots are real pages, not login screens
- [ ] No PII visible in screenshots
- [ ] Metrics are real business results, not technical facts
- [ ] Metrics labels readable (no broken words)
- [ ] URLs in cache match actual deployments
- [ ] Slugs unchanged after regeneration (check log for "Slug pinned" warnings)
- [ ] `publicUrl: false` for projects behind auth (no "Open project" button)
- [ ] New projects have correct category (SaaS/Tool/Bot/Landing)
- [ ] Auth config keys match slugs (NOT repo names)
- [ ] Brief text is compelling, not technical
- [ ] Read and update `LESSONS.md` if anything new learned
