# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This is a multi-project workspace containing:

- **x10seo** - SEO click management service with BAS integration
- **v0-subscription-tracker** - Subscription tracker with Telegram + AI
- **germanyun-ru** - Public SEO site for German Yun (programs, blog, cases, glossary) with admin panel
- **kz.pashavin.ru**, **v0-german** - Various v0.app applications

## Infrastructure

### Supabase (self-hosted)
- **URL:** `https://supabase.pashavin.ru` (server: 135.181.115.234)
- **Config:** `/home/deploy/projects/supabase-selfhosted/`
- **Schemas:** `public` (subscription-tracker), `x10seo` (x10seo)
- **PostgREST schemas:** `PGRST_DB_SCHEMAS=public,storage,graphql_public,x10seo`
- **Adding new schema:** update `PGRST_DB_SCHEMAS` in `.env`, then `docker compose up -d rest` (restart won't pick up new env vars)
- **Permissions:** new schemas need `GRANT USAGE/ALL` to `anon`, `authenticated`, `service_role`

### Dokploy (deployment) — мигрирован KZ → Hetzner 2026-04-24
- **URL:** `https://deploy.pashavin.ru` (VPS: **`135.181.115.234`** — Hetzner, Helsinki)
- **API key:** `HXFWBzzjMucGmaquIEwgvfVcpLmbtSySlUQagfONmrclqtsKvKxpbhAmNBOmVnjS` (saved in `/home/deploy/.env.dokploy-hz`)
- **Purpose:** self-hosted PaaS replacing Vercel (blocked in Russia since June 2025)
- **Architecture:** Caddy (host:80/443, SSL для всех доменов) → Dokploy Traefik (`127.0.0.1:9080`, плейн HTTP) → Docker Swarm services
- **Deploy pattern:** `output: "standalone"` в next.config + multi-stage Dockerfile (node:20-alpine)
- **Dockerfile cache rule:** ALL system-level deps (`apk add`, `apt-get`) MUST go в `base` stage BEFORE `COPY`. Never install in `runner` or after `COPY . .` — это убивает Docker layer caching.
- **Cron:** no built-in; use system cron with `curl -H "Authorization: Bearer $CRON_SECRET"`

### Hetzner — production server (Helsinki)
- **IP:** `135.181.115.234`
- **Hostname:** `pashavin-main`
- **OS:** Ubuntu 24.04 · 12 cores Ryzen 9 3900 · 128 GB ECC · 2×1.92 TB NVMe RAID-1
- **Caddy frontend:** `/etc/caddy/Caddyfile` — explicit blocks для инфры (supabase, beszel, gatus, claw...) + один консолидированный block для 22 Dokploy-доменов → `127.0.0.1:9080`
- **Dokploy Traefik:** `dokploy-traefik` контейнер на `127.0.0.1:9080` (НЕ публичный 80/443) — Caddy проксирует туда по Host header
- **Apps лежат:** все Dokploy-managed apps в Docker Swarm; legacy stuff в `/opt/<name>/docker-compose.yml` (supabase, lobechat, parser-hub-deps, twenty)

### KZ (5.35.80.222) — DEPRECATED
- Dokploy stack ещё крутится но без apps (scaled 0/0). Готов к выключению.
- НЕ деплоить ничего туда. Все секреты теперь Hetzner-Dokploy.

### Deploy Pipeline (GitHub Actions + GHCR + SSH → Hetzner)
- **Flow:** push → GHA builds → pushes to GHCR → SSH `root@135.181.115.234` → `docker pull :latest` → `docker service update --force`
- **Time:** ~2 min build + ~1 min deploy = **<3 min**
- **Why SSH not Dokploy API:** Dokploy's `application.deploy` не force-pull'ит `:latest` — Swarm reuses cached image
- **GHCR Registry в Dokploy:** name `ghcr`, registryId `me8i2YOWSfydCd3TWvsAk`
- **GHA SSH key:** `/home/deploy/.ssh/gha_deploy` (public добавлен в `/root/.ssh/authorized_keys` на Hetzner)
- **Secrets per repo (GHA):**
  - `VPS_HOST=135.181.115.234`
  - `VPS_SSH_KEY` = содержимое `gha_deploy` (приватный ключ)
  - `DOKPLOY_URL=https://deploy.pashavin.ru`
  - `DOKPLOY_API_KEY=HXFWBzzj...`
  - `DOKPLOY_APP_ID` = unique per app (Dokploy app id)
  - `DOKPLOY_SERVICE_NAME` = unique per app (Docker Swarm service name `app-<random>-<hash>`)
- **Workflow:** `.github/workflows/deploy.yml` — uses `GITHUB_TOKEN` для GHCR push, SSH для deploy
- **GHA cache:** Docker layer cache via `cache-from: type=gha`
- **NEXT_PUBLIC build args:** projects with Dockerfile ARGs need `gh variable set` + `build-args` in workflow

### New app checklist
1. Create app в Dokploy: `sourceType: "docker"`, `dockerImage: "ghcr.io/bugle-c/<repo>:latest"`, `registryId: "me8i2YOWSfydCd3TWvsAk"`
2. Deploy впервые через Dokploy UI, затем find service name: `docker service ls | grep <repo>`
3. Update domain в Dokploy с `https=false, certificateType=none` (Caddy handles SSL upstream)
4. Add `.github/workflows/deploy.yml` (copy from existing project, e.g. x10seo)
5. Set GHA secrets (см. выше)
6. Add domain в Caddyfile внутрь общего Dokploy block ИЛИ создать explicit block если нужны кастомные headers/middleware

### Files & data backups
- Hetzner backups: `/opt/backups/` (daily 3:00 MSK, retention 7)
- KZ Dokploy DB backup: `/opt/backups/kz-dokploy-20260424_1542.sql.gz`
- parser-hub last DB dump: `/opt/backups/parser-hub-20260424_1636.sql.gz`

### Brevo (transactional email)
- **Plan:** Free (300 emails/day)
- **Domain:** `pashavin.ru` (verified, SPF/DKIM configured)
- **Sender:** `noreply@pashavin.ru`
- **SDK:** `@getbrevo/brevo` — installed in x10seo, v0-subscription-tracker, pasha-vin-post-bot
- **Env var:** `BREVO_API_KEY` (set in `.env.local` / `.env` of each project)
- **Email utilities:**
  - Next.js projects: `lib/email.ts` — `sendEmail({ to, subject, html })`
  - pasha-vin-post-bot: `src/notifications/email.ts` — same API, lazy init via `getConfig()`
- **Adding to new project:** `npm install @getbrevo/brevo`, copy `lib/email.ts`, add `BREVO_API_KEY` to env
- **Note:** v0-subscription-tracker needs `--legacy-peer-deps` for install

## Tech Stack Summary

| Project | Framework | Package Manager | Database |
|---------|-----------|-----------------|----------|
| x10seo | Next.js 16, React 19 | npm | Self-hosted Supabase (schema `x10seo`) |
| v0-subscription-tracker | Next.js 16, React 19 | npm | Self-hosted Supabase (schema `public`) |
| germanyun-ru | Next.js 16, React 19 | npm | Self-hosted Supabase (schema `germanyun`) |
| Other v0 projects | Next.js 16, React 19 | npm | Self-hosted Supabase |

Common stack: TypeScript, Tailwind CSS, shadcn/ui

## Commands by Project

### v0 Projects (x10seo, v0-subscription-tracker, etc.)

```bash
npm install
npm run dev      # Development server
npm run build    # Production build
npm run lint     # ESLint
```

## Architecture Patterns

### x10seo

- **Standard Next.js** App Router with ~40 API routes
- **Database**: Supabase JS SDK with `{ db: { schema: "x10seo" } }`, service role key (bypasses RLS)
- **Auth**: Cookie-based sessions (`session_user_id`), legacy md5 passwords + bcrypt
- **BAS API**: `/api/bas/next` and `/api/tasks/next` return `text/plain`, not JSON
- **Cron**: 3 endpoints (process-tasks, deliver-tasks, run-intervals) behind Bearer token auth

### v0 Projects (general)

- **Standard Next.js** App Router structure
- **Auth**: NextAuth with OAuth (Google, Yandex, Telegram) or Supabase Auth
- **Database**: Supabase JS SDK via HTTPS REST API

## Key Guidelines

### Supabase JS SDK Patterns

When working with Supabase SDK (used in x10seo, v0-subscription-tracker):
- **Lazy init via Proxy** in `lib/supabase.ts` — prevents `createClient` crash during build without env vars
- **Custom schema**: `createClient(url, key, { db: { schema: "name" } })`
- **Complex SQL → JS**: `GROUP BY` / aggregations fetched as raw rows, aggregated in JavaScript
- **Row locking**: `FOR UPDATE SKIP LOCKED` not available — use optimistic locking (select + update with status check)
- **Advisory locks**: Use a `process_orchestration` table instead of `pg_try_advisory_lock`

## Global Rules

### Dokploy — только через скилл или агента
Любая работа с Dokploy (деплой, создание приложений, настройка, отладка) — ТОЛЬКО через скилл `dokploy-deploy` или агент `dokploy-deployer`. Никаких ad-hoc curl-команд к API. Скилл и агент содержат актуальный pipeline (GitHub Actions + GHCR), правильные ID, секреты и чеклисты.

### Язык общения
Всегда общайся с пользователем на русском языке. Код, комментарии в коде и commit messages — на английском.

### Knowledge Files
Each project that undergoes active development MUST have a `KNOWLEDGE.md` in its root. This file captures:
- **Architecture decisions** — what was chosen and why (X replaced with Y, got Z)
- **Known pitfalls** — things that broke and how they were fixed
- **Key constants and limits** — with rationale
- **Infrastructure** — how services are run, restarted, connected

When starting work on a project, read its `KNOWLEDGE.md` first. When finishing work, update it with new decisions and lessons learned. Keep it concise — only essential facts, no prose.

**Size limit: 150 lines.** If `KNOWLEDGE.md` exceeds 150 lines — refactor into index + files:
- Keep `KNOWLEDGE.md` as a short index (links + one-line descriptions), like `MEMORY.md`
- Move sections into `docs/knowledge/` as separate files (e.g., `architecture.md`, `pitfalls.md`, `infrastructure.md`)
- Refactoring happens organically during project work, not as a separate task

## Workflow Orchestration

### Task Complexity → Action

| Задача | Действие |
|--------|----------|
| Тривиальная (1-2 шага, очевидно) | Делай сразу |
| Нетривиальная (3+ шагов, архитектурные решения) | `brainstorming` → `writing-plans` → реализация |
| Баг-репорт | Диагностируй → исправляй → проверяй. Не проси вести за ручку |
| Что-то идёт не так при реализации | СТОП → перепланируй, не продолжай давить |

### Subagent Strategy
- Используй субагентов свободно для: исследования кодовой базы, параллельного анализа, тяжёлых поисков
- Одна задача на субагента — сфокусированное выполнение
- Основное контекстное окно — для принятия решений, не для сбора данных

### Task Management & Lessons
- **Per-project:** `<project>/tasks/todo.md` — чеклист текущих задач проекта
- **Глобально:** `tasks/lessons.md` — накопленные уроки из всех проектов
- Перед началом работы с проектом — проверь `tasks/todo.md` если существует
- После замечания от пользователя или исправления ошибки — обнови `tasks/lessons.md` с паттерном и превентивным правилом
- В начале сессии — просмотри `tasks/lessons.md` для контекста

### Verification Before Completion
- Никогда не отмечай задачу завершённой без доказательства работоспособности
- Запускай `npm run build`, тесты, проверяй логи — демонстрируй корректность
- Используй скилл `verification-before-completion` для полного чеклиста

### Engineering Principles
- **Простота прежде всего**: минимальное изменение для достижения цели. Не переусложняй.
- **Корневые причины**: ищи root cause, никаких временных заплаток. Стандарты senior developer.
- **Элегантность для нетривиального**: перед подачей нетривиального изменения — спроси себя «есть ли более чистый способ?». Для простых фиксов — пропускай.
- **Автономность**: получив баг-репорт — диагностируй и чини. Не проси пользователя вести за ручку. Указывай на логи, ошибки, падающие тесты — затем устраняй.
- **Минимальное воздействие**: изменения затрагивают только то, что необходимо. Не вноси новых багов.
