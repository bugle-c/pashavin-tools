---
name: supabase
description: Use when managing self-hosted Supabase infrastructure — schemas, permissions, backups, Docker services, client setup, or troubleshooting database connectivity. Also use when user says "supabase", "schema", "PostgREST", "database", "бекап", "бэкап".
---

# Supabase Self-Hosted Management

> **Note:** Communicate with user in Russian. Code, commands, and comments in English.

## Quick Reference

| Item | Value |
|------|-------|
| URL | `https://supabase.pashavin.ru` |
| Server | `194.113.209.247` |
| Config dir | `/home/deploy/projects/supabase-selfhosted/` |
| Studio | `https://supabase.pashavin.ru` (basic auth) |
| REST API | `https://supabase.pashavin.ru/rest/v1/` (apikey header, no basic auth) |
| Auth API | `https://supabase.pashavin.ru/auth/v1/` (apikey header, no basic auth) |
| DB port | `127.0.0.1:5432` (local only) |
| Kong port | `127.0.0.1:8000` |
| Supavisor | `127.0.0.1:6543` (connection pooler) |

### Current Schemas (PGRST_DB_SCHEMAS)

```
public, storage, graphql_public, x10seo, ai_aggregator, seo_aggregator,
slides_generator, cjm, tg_army, crm_messenger, kp, arbscanner, uis_bridge, content_farm
```

## Docker Compose Operations

All commands from `/home/deploy/projects/supabase-selfhosted/` on `194.113.209.247`.

```bash
docker compose ps                          # Status
docker compose restart rest                # Restart single service
docker compose logs -f db --tail=100       # Logs
docker compose up -d                       # Start all (run twice on fresh deploy)
docker compose down                        # Stop all
```

## New Schema Setup (4 steps, all required)

### Step 1: Create schema + permissions

```bash
docker compose exec -T db psql -U supabase_admin -d postgres -c "
  CREATE SCHEMA IF NOT EXISTS <schema_name>;
  GRANT USAGE ON SCHEMA <schema_name> TO anon, authenticated, service_role;
  GRANT ALL ON ALL TABLES IN SCHEMA <schema_name> TO anon, authenticated, service_role;
  GRANT ALL ON ALL ROUTINES IN SCHEMA <schema_name> TO anon, authenticated, service_role;
  GRANT ALL ON ALL SEQUENCES IN SCHEMA <schema_name> TO anon, authenticated, service_role;
  ALTER DEFAULT PRIVILEGES FOR ROLE supabase_admin IN SCHEMA <schema_name>
    GRANT ALL ON TABLES TO anon, authenticated, service_role;
  ALTER DEFAULT PRIVILEGES FOR ROLE supabase_admin IN SCHEMA <schema_name>
    GRANT ALL ON ROUTINES TO anon, authenticated, service_role;
  ALTER DEFAULT PRIVILEGES FOR ROLE supabase_admin IN SCHEMA <schema_name>
    GRANT ALL ON SEQUENCES TO anon, authenticated, service_role;
"
```

### Step 2: Update PGRST_DB_SCHEMAS in .env

Append new schema to `PGRST_DB_SCHEMAS` in `.env`.

### Step 3: Restart PostgREST

```bash
# CRITICAL: use `up -d`, NOT `restart` — restart doesn't pick up new env vars!
docker compose up -d rest
```

### Step 4: Verify

```bash
curl -s https://supabase.pashavin.ru/rest/v1/ \
  -H "apikey: $ANON_KEY" \
  -H "Accept-Profile: <schema_name>"
# Should return [], not 404
```

## Client Integration Patterns

### Lazy Init via Proxy (prevents build crash without env vars)

```typescript
import { createClient, SupabaseClient } from "@supabase/supabase-js";

const supabaseProxy = new Proxy({} as SupabaseClient, {
  get(_, prop) {
    const client = createClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_ROLE_KEY!,
      { db: { schema: "my_schema" } }
    );
    return (client as any)[prop];
  },
});
export default supabaseProxy;
```

### Service Role vs Anon Key

| Key | Use case | RLS |
|-----|----------|-----|
| `SUPABASE_SERVICE_ROLE_KEY` | Backend, cron, admin | Bypasses RLS |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Client-side | Respects RLS |

### Row Locking (no FOR UPDATE SKIP LOCKED)

Optimistic locking: select + update with status check.

```typescript
const { data: task } = await supabase
  .from("tasks").select("*").eq("status", "pending").limit(1).single();
if (task) {
  const { data: updated } = await supabase
    .from("tasks").update({ status: "processing" })
    .eq("id", task.id).eq("status", "pending").select().single();
  // updated is null if another worker grabbed it
}
```

## Backup & Restore

```bash
# Manual backup
docker compose exec -T db pg_dumpall -U supabase_admin | gzip > backups/supabase-manual-$(date +%Y%m%d-%H%M%S).sql.gz

# Single schema backup
docker compose exec -T db pg_dump -U supabase_admin -d postgres -n <schema> | gzip > backups/<schema>-$(date +%Y%m%d).sql.gz

# Restore (DESTRUCTIVE)
gunzip -c backups/file.sql.gz | docker compose exec -T db psql -U supabase_admin -d postgres
```

Automated: cron `0 3 * * *`, 14-day retention, log `/var/log/supabase-backup.log`.

## Direct Database Access

```bash
docker compose exec db psql -U supabase_admin -d postgres           # Interactive
docker compose exec -T db psql -U supabase_admin -d postgres -c "\dn"  # List schemas
docker compose exec -T db psql -U supabase_admin -d postgres -c "\dt x10seo.*"  # Tables in schema
```

## Caddy (Reverse Proxy)

**CRITICAL:** API endpoints (`/rest/*`, `/auth/*`, `/realtime/*`, `/storage/*`) → Kong, NO basic_auth. Studio → basic_auth.

```bash
sudo cp Caddyfile /etc/caddy/Caddyfile && sudo systemctl reload caddy
```

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| API 401 after Caddy change | `basic_auth` at site level blocks API | Put basic_auth in non-API handle only |
| Schema not visible via REST | PGRST_DB_SCHEMAS not updated or `restart` used | Update .env + `docker compose up -d rest` |
| New tables not accessible | Missing GRANT | Run GRANT for anon/authenticated/service_role |
| Future tables not accessible | Missing ALTER DEFAULT PRIVILEGES | Add default privileges |
| Services fail on first deploy | DB healthcheck race | Run `docker compose up -d` twice |
| Storage healthcheck fails | IPv6 issue | Use `127.0.0.1` not `localhost` |
| Supavisor won't start | `_supabase` DB missing | Create via deploy script |

## Troubleshooting

```
Can't connect to API
├── 401 → basic_auth blocking? apikey correct?
├── 404 → schema in PGRST_DB_SCHEMAS? `up -d rest`
├── Connection refused → docker compose ps / systemctl status caddy
└── 500 → docker compose logs kong rest auth

Schema query returns no data
├── Table exists? → \dt <schema>.*
├── GRANT on schema? → \dn+ <schema>
├── RLS blocking? → Use service_role key
└── Accept-Profile header set?
```

## Monitoring

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}" | grep supabase
docker compose exec -T db psql -U supabase_admin -d postgres -c "SELECT pg_size_pretty(pg_database_size('postgres'));"
ls -lht backups/ | head -5
```
