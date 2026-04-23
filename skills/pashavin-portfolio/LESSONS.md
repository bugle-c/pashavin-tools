# Portfolio Pipeline — Lessons Learned

Self-learning log. Read BEFORE any pipeline work. Update AFTER every issue or improvement.

## Rules (derived from mistakes)

### URLs & Buttons
- **NEVER show "Open project" button for projects behind auth** (CRM, admin panels, internal tools). Use `publicUrl: false`.
- **Only Landing pages and public SaaS get the button.** If `getAuthConfig(slug)` exists → `publicUrl: false` auto-applied.
- URL field is for screenshot capture, NOT for public display. These are separate concerns.

### Metrics (Business Impact)
- **Metrics must be REAL business results**, not technical facts.
- BAD: `{value: "REST", label: "эндпоинтов"}`, `{value: "AI", label: "провайдера"}`
- GOOD: `{value: "0", label: "ручной работы"}`, `{value: "10x", label: "быстрее"}`, `{value: "24/7", label: "автономная работа"}`
- If you can't measure a real result, use transformation metrics: "3ч→5мин", "0 ошибок", "100% coverage".

### Cache & Refresh
- **--refresh DELETES cache and re-generates.** Claude API doesn't know deploy URLs.
- Script now preserves URLs AND slugs on refresh, but ALWAYS verify after refresh.
- If URL disappeared → add manually to `projects-meta-cache.json`.
- **Slugs are PERMANENT for SEO** — script pins them automatically. If Claude API returns a different slug, the old one is forced back. Log: `⚠ Slug pinned: "new" → "old" (SEO protection)`.
- Exception: if multi-service count changes, slugs are NOT pinned — manual review needed.

### Screenshots & Auth
- **Always check screenshot result** — is it the real page or a login screen?
- Projects behind auth need `getAuthConfig()` entry. Without it, screenshot capture shows login page.
- Twenty CRM needs SPA wait (5s+) and multi-step login (Continue with Email → email → password).
- **PII protection**: script hides user data elements, but always visually verify.
- AI illustrations are fallback ONLY when no URL exists. **NEVER generate AI illustration if URL is available** — create a real screenshot-bot user instead.
- **Auth config key must match slug, NOT repo name.** Multi-service repos generate slugs different from repo name (e.g., repo `v0-german` → slugs `cjm-builder`, `kp-generator`). Wrong key = "no auth config" warning.
- **`__Secure-` cookie prefix** requires `secure: true` in Playwright `addCookies`. Script sets this automatically for HTTPS URLs.
- **Heavy SPAs** (LobeChat/WebGPT) need `waitMs: 10000` in auth config — default 3s timeout captures loading spinner instead of UI.
- **Better Auth user creation**: use `POST /api/auth/sign-up/email` with `{email, password, name}`. Sign in with `POST /api/auth/sign-in/email` to get session cookie. Cookie name: `__Secure-better-auth.session_token`.

### External Projects
- Non-GitHub projects (self-hosted services, forks) go in `EXTERNAL_PROJECTS` array.
- They need manual `url`, `publicUrl`, and auth config.
- They won't auto-update from GitHub — updates are manual.

### Metrics Labels
- Labels must NOT contain `\n` line breaks. Component strips them with `.replace(/\n/g, " ")`.
- Prompt updated to say "БЕЗ переносов строки".

## Success Patterns

- **Fresh browser context per project** in screenshot capture → isolated auth sessions, no cookie leaks
- **Category-aware AI illustrations** → Bot/SaaS/Tool/Landing produce relevant visuals
- **Preserve URLs on refresh** → no data loss when regenerating descriptions
- **Auto-hide URL for auth projects** → `getAuthConfig() + publicUrl: false` applied automatically
- **form-login auth type** → handles multi-step SPA login (Twenty CRM pattern)
- **Slug pinning** → preservedSlugs dict saves old slugs before cache reset, forces them back after Claude API regeneration. SEO-safe by default.
- **screenshot-bot user pattern** → dedicated `screenshot-bot@pashavin.ru` accounts per auth project, never admin creds
- **`waitMs` for heavy SPAs** → configurable per-project wait time in auth config, prevents capturing loading screens

## Checklist for new lessons

When something goes wrong or right:
1. Document the issue/success briefly
2. Add rule to appropriate section above
3. If it's a prompt issue → update prompt in `generate-projects.ts`
4. If it's a display issue → update component
5. If it's a pipeline issue → update script logic
6. Update the skill SKILL.md if the lesson changes the workflow
