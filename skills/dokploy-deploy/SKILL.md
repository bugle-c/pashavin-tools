---
name: dokploy-deploy
description: Use when deploying a new app, creating Dokploy infrastructure, setting up GitHub Actions CI/CD, debugging deploy failures, or when user says "deploy", "dokploy", "задеплоить". Also use when creating Dockerfiles for Next.js standalone builds.
---

# Dokploy Deploy (GitHub Actions + GHCR + SSH Pipeline)

## Overview

Deploy Next.js apps via GitHub Actions → GHCR → SSH pull + service update on VPS. VPS never builds — only pulls and runs pre-built images. Deploy time: **1-3 min** (was 20-30 min).

## Quick Reference

| Item | Value |
|------|-------|
| Dokploy URL | `https://deploy.pashavin.ru` |
| API auth | `x-api-key: AUEXKnibHiiBpfBjZUcFnESzelQAKllLgCQRnflgxzbOKoYhIxdpxVucpbubEkHU` |
| VPS IP | `5.35.80.222` |
| GHCR Registry ID | `mX2azxgDrTTaNHp5yuShv` (name: `ghcr`) |
| GitHub org | `bugle-c` |
| SSH key (VPS) | `/home/deploy/.ssh/id_ed25519` → `root@135.181.115.234` |
| SSH key (GitHub) | `/home/deploy/.ssh/github_deploy` |
| DNS | Wildcard `*.pashavin.ru` → `5.35.80.222` |
| git-auto-sync | `~/.claude/scripts/git-auto-sync.sh` |

## Pipeline Flow

```
git push → GitHub Actions builds Docker image → pushes to GHCR
         → SSH into VPS → docker pull :latest → docker service update --force
```

- **Build:** GitHub Actions runner (7 GB RAM, free tier: 2000 min/month)
- **Cache:** GHA Docker layer cache (`type=gha`) — repeat builds 1-2 min
- **Deploy:** SSH → `docker pull` (only changed layers) + `service update --force` — 30-90 sec
- **Concurrency:** `cancel-in-progress: true` — rapid pushes cancel stale builds

## CRITICAL: Why SSH instead of Dokploy API

Dokploy uses Docker Swarm. `application.deploy` calls `docker service update` but Swarm does NOT re-pull `:latest` — it reuses the cached image digest. SHA tags force pull but break Docker layer caching (every deploy downloads ALL layers).

**Solution:** SSH into VPS, run `docker pull :latest` (pulls only changed layers), then `docker service update --force` (restarts with fresh image). Dokploy stays for infrastructure (Traefik, SSL, env vars, dashboard) but is NOT used for the actual deploy trigger.

## Deploy Workflow (New App)

### 1. Prepare project

```bash
# next.config.ts MUST have standalone output
output: "standalone"
```

Create Dockerfile (see template below), `.dockerignore`, init git, push:
```bash
git init && git add -A && git commit -m "Initial commit"
gh repo create bugle-c/<name> --private --source=. --push
```

### 2. Create Dokploy project + app

```bash
API_KEY="AUEXKnibHiiBpfBjZUcFnESzelQAKllLgCQRnflgxzbOKoYhIxdpxVucpbubEkHU"

# Create project
PROJECT=$(curl -s -X POST "https://deploy.pashavin.ru/api/project.create" \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"<name>","description":"<desc>"}')
PROJECT_ID=$(echo $PROJECT | python3 -c "import sys,json; print(json.load(sys.stdin)['project']['projectId'])")
ENV_ID=$(echo $PROJECT | python3 -c "import sys,json; print(json.load(sys.stdin)['environment']['environmentId'])")

# Create app
APP=$(curl -s -X POST "https://deploy.pashavin.ru/api/application.create" \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d "{\"name\":\"<name>\",\"projectId\":\"$PROJECT_ID\",\"environmentId\":\"$ENV_ID\"}")
APP_ID=$(echo $APP | python3 -c "import sys,json; print(json.load(sys.stdin)['applicationId'])")
```

### 3. Configure app for Docker image source

```bash
curl -s -X POST "https://deploy.pashavin.ru/api/application.update" \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d "{
    \"applicationId\":\"$APP_ID\",
    \"sourceType\":\"docker\",
    \"dockerImage\":\"ghcr.io/bugle-c/<name>:latest\",
    \"registryId `me8i2YOWSfydCd3TWvsAk` (was KZ: mX2az...) \"
  }"
```

### 4. Find Docker service name

After first deploy via Dokploy UI, find the generated service name:
```bash
ssh root@135.181.115.234 'docker service ls --format "{{.Name}} {{.Image}}" | grep <name>'
```

### 5. Create GitHub Actions workflow

```bash
mkdir -p .github/workflows
```

Copy `.github/workflows/deploy.yml` from any existing project (e.g. x10seo).
**Check branch:** use `main` or `master` depending on project.

### 6. Set GitHub Actions secrets

```bash
gh secret set DOKPLOY_URL -b "https://deploy.pashavin.ru" --repo bugle-c/<name>
gh secret set DOKPLOY_API_KEY -b "$API_KEY" --repo bugle-c/<name>
gh secret set DOKPLOY_APP_ID -b "$APP_ID" --repo bugle-c/<name>
gh secret set VPS_SSH_KEY -b "$(cat /home/deploy/.ssh/id_ed25519)" --repo bugle-c/<name>
gh secret set VPS_HOST -b "5.35.80.222" --repo bugle-c/<name>
gh secret set DOKPLOY_SERVICE_NAME -b "<service-name-from-step-4>" --repo bugle-c/<name>
```

If project has `NEXT_PUBLIC_*` build args, set them as GitHub Variables (not secrets):
```bash
gh variable set NEXT_PUBLIC_API_URL -b "https://..." --repo bugle-c/<name>
```

### 7. Add to git-auto-sync.sh

Edit `~/.claude/scripts/git-auto-sync.sh`:
```bash
GHA_APPS[<dir-name>]=1
```

### 8. Push and verify

```bash
git add -A && git commit -m "ci: add GitHub Actions build + GHCR deploy pipeline" && git push
```

Monitor: `gh run watch --repo bugle-c/<name>`

## GitHub Actions Workflow Template

See `/home/deploy/projects/x10seo/.github/workflows/deploy.yml` — copy and adjust `branches:` (`main` or `master`).

**Key deploy step (SSH pull + force update):**
```yaml
      - name: Deploy to VPS
        env:
          SSH_KEY: ${{ secrets.VPS_SSH_KEY }}
          VPS_HOST: ${{ secrets.VPS_HOST }}
          SERVICE_NAME: ${{ secrets.DOKPLOY_SERVICE_NAME }}
        run: |
          mkdir -p ~/.ssh
          echo "$SSH_KEY" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H $VPS_HOST >> ~/.ssh/known_hosts 2>/dev/null
          SSH_OPTS="-i ~/.ssh/deploy_key -o ServerAliveInterval=15 -o ServerAliveCountMax=20 -o ConnectTimeout=10"
          IMAGE="ghcr.io/${{ github.repository }}:latest"
          ssh $SSH_OPTS root@$VPS_HOST "docker pull $IMAGE"
          ssh $SSH_OPTS root@$VPS_HOST "docker service update --force --image $IMAGE $SERVICE_NAME"
```

**For projects with NEXT_PUBLIC build args, add to build step:**
```yaml
          build-args: |
            NEXT_PUBLIC_API_URL=${{ vars.NEXT_PUBLIC_API_URL }}
```

## Dockerfile Template

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS base
# ALL system deps HERE (apk add) — BEFORE any COPY for cache

FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN --mount=type=cache,target=/app/.next/cache npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"
CMD ["node", "server.js"]
```

## .dockerignore Template

```
node_modules
.next
.git
*.md
docs/
.vscode/
.env*
tasks/
.github/
```

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| GHA workflow not triggered | Wrong branch in `deploy.yml` | Check `branches: [main]` vs `[master]` |
| **Deploy succeeds but old code runs** | **Dokploy API doesn't force pull** | **Use SSH: `docker pull` + `service update --force`** |
| **SSH broken pipe** | **Long pull, SSH timeout** | **Add `-o ServerAliveInterval=15`** |
| **NEXT_PUBLIC vars = localhost** | **Build args not passed in GHA** | **Add `build-args` + set `gh variable`** |
| GHCR push 403 | Missing `packages: write` permission | Check `permissions:` block in workflow |
| Dokploy can't pull image | Registry not linked | Set `registryId `me8i2YOWSfydCd3TWvsAk` (was KZ: mX2az...) "` |
| **Slow pull (5+ min)** | **Large image (>400 MB)** | **Optimize Dockerfile, reduce standalone bundle** |
| Build takes 20+ min on GHA | No layer cache | Verify `cache-from: type=gha` in workflow |

## API Endpoints (for infrastructure only, NOT for deploy trigger)

| Endpoint | Method | Key params |
|----------|--------|------------|
| `project.all` | GET | — |
| `project.create` | POST | `name`, `description` |
| `application.create` | POST | `name`, `projectId`, `environmentId` |
| `application.update` | POST | `applicationId` + fields |
| `application.one` | GET | `?applicationId=` |
| `registry.all` | GET | — |

All endpoints: `https://deploy.pashavin.ru/api/`.

## Rollback

If SSH deploy fails, Dokploy API still works as fallback:
```bash
curl -s -X POST "https://deploy.pashavin.ru/api/application.deploy" \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"applicationId":"<APP_ID>"}'
```
Note: this restarts with cached image, not fresh pull.
