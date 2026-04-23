---
name: twenty-crm
description: Use when working with Twenty CRM - managing contacts, companies, deals, pipeline stages, CRM infrastructure, Docker containers, backups, schema configuration, or when code imports crm_sync or references TWENTY_CRM
---

# Twenty CRM

## Overview

Skill for managing Twenty CRM at crm.pashavin.ru — both data operations (via MCP tools or REST/GraphQL API) and infrastructure administration (via SSH to VPS #2).

## When to Use

- User mentions CRM, Twenty, deals, contacts, leads, pipeline, funnel, opportunities
- Code imports `crm_sync`, `CRMSync`, or references `TWENTY_CRM`
- Tasks involve Docker containers `twenty-*`
- User asks about backups, CRM schema, custom fields, or pipeline stages

## Infrastructure

**Server:** VPS #2 (5.35.80.222), SSH: `ssh root@5.35.80.222`
**Path:** `/opt/twenty/`
**Containers:** twenty-app, twenty-worker, twenty-db (postgres:16), twenty-redis (redis:7-alpine)
**Domain:** crm.pashavin.ru
**Health:** `curl https://crm.pashavin.ru/healthz`

### Common Operations

```bash
# Status
ssh root@5.35.80.222 "cd /opt/twenty && docker compose ps"

# Logs (server)
ssh root@5.35.80.222 "cd /opt/twenty && docker compose logs --tail=100 server"

# Logs (worker)
ssh root@5.35.80.222 "cd /opt/twenty && docker compose logs --tail=100 worker"

# Restart
ssh root@5.35.80.222 "cd /opt/twenty && docker compose restart server worker"

# Update Twenty version
ssh root@5.35.80.222 "cd /opt/twenty && docker compose pull && docker compose up -d"

# DB shell
ssh root@5.35.80.222 "docker exec -it twenty-db psql -U postgres default"
```

## Backups

```bash
# Backup
ssh root@5.35.80.222 "docker exec twenty-db pg_dump -U postgres default" > /home/deploy/backups/twenty/backup_$(date +%Y%m%d).sql

# Restore (CAREFUL — overwrites all data)
cat backup.sql | ssh root@5.35.80.222 "docker exec -i twenty-db psql -U postgres default"
```

**Rule:** ALWAYS backup before updating Twenty version or modifying schema.

## Data Model

```
Company (1) ──→ People (many) ──→ Opportunities/Deals (many)
                                   ├── Tasks (many)
                                   └── Notes (many)
```

### Lead Sources
- `TELEGRAM` — auto-import from tg-army
- `WEBSITE` — website forms
- `MANUAL` — manual entry
- `ADS` — advertising campaigns

### Custom Fields on Person
- `niche` — business niche
- `direction` — service direction (MARKETING, INFOBIZ, AI_INTEGRATION)
- `leadSource` — lead source
- `city` — city

### Pipeline Stages
Configurable via Settings or Metadata API.

## API Reference

**REST base:** `https://crm.pashavin.ru/rest/`
**GraphQL:** `https://crm.pashavin.ru/graphql`
**Auth:** `Authorization: Bearer <TWENTY_CRM_API_KEY>`
**API key location:** `/home/deploy/.env.twenty`
**Rate limit:** 100 req/min, batch up to 60 records

### REST Endpoints (main pattern: /rest/<entity>)

```bash
# List people
curl -s "https://crm.pashavin.ru/rest/people?limit=10" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"

# List companies
curl -s "https://crm.pashavin.ru/rest/companies?limit=10" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"

# List opportunities
curl -s "https://crm.pashavin.ru/rest/opportunities?limit=10" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"

# Create person
curl -s "https://crm.pashavin.ru/rest/people" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": {"firstName": "Ivan", "lastName": "Petrov"}, "leadSource": "MANUAL"}'

# Create company
curl -s "https://crm.pashavin.ru/rest/companies" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Company"}'

# Update person
curl -s "https://crm.pashavin.ru/rest/people/<id>" \
  -X PATCH \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jobTitle": "CEO"}'

# Delete person
curl -s "https://crm.pashavin.ru/rest/people/<id>" \
  -X DELETE \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"
```

### Filtering

```bash
# Filter by lead source
curl -s 'https://crm.pashavin.ru/rest/people?filter=leadSource[eq]:"TELEGRAM"' \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"

# Filter by date
curl -s 'https://crm.pashavin.ru/rest/opportunities?filter=createdAt[gte]:"2026-01-01"' \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"
```

### Pagination

```bash
# First page
curl -s "https://crm.pashavin.ru/rest/companies?limit=60" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"

# Next page (use endCursor from previous response)
curl -s "https://crm.pashavin.ru/rest/companies?limit=60&starting_after=<endCursor>" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"
```

### GraphQL Examples

```bash
# List companies
curl -s https://crm.pashavin.ru/graphql \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ companies { edges { node { id name } } } }"}'

# List opportunities
curl -s https://crm.pashavin.ru/graphql \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ opportunities { edges { node { id name stage } } } }"}'
```

**Note:** GraphQL introspection is disabled. Use REST OpenAPI spec for schema discovery:
`https://crm.pashavin.ru/open-api/core`

## MCP Tools (29 tools, SDK 1.27)

When MCP server `twenty-crm` is active, use MCP tools instead of curl:
- **People:** `create_person`, `get_person`, `update_person`, `list_people`, `delete_person`
- **Companies:** `create_company`, `get_company`, `update_company`, `list_companies`, `delete_company`
- **Opportunities:** `create_opportunity`, `get_opportunity`, `update_opportunity`, `list_opportunities`, `delete_opportunity`, `batch_update_opportunities`
- **Tasks:** `create_task_crm`, `update_task_crm`, `list_tasks_crm`, `delete_task_crm`
- **Notes:** `create_note`, `update_note`, `list_notes`, `delete_note`
- **Metadata:** `get_metadata_objects`, `get_object_metadata`, `list_fields`
- **Other:** `search_records`, `list_workspace_members`

Tools are named `mcp__twenty-crm__<tool_name>` in Claude Code.

**Prefer MCP tools over curl when available.**

## Integrations

### tg-army (auto-sync qualified leads)
- **File:** `/home/deploy/projects/tg-army/src/crm/crm_sync.py`
- **Trigger:** lead reaches `qualified` state in dialog_manager
- **Creates:** Person + Opportunity via GraphQL
- **Env:** `TWENTY_CRM_URL`, `TWENTY_CRM_API_KEY` in tg-army `.env`

## Metadata API

### Discover objects

```bash
curl -s "https://crm.pashavin.ru/rest/metadata/objects" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"
```

### Discover fields for an object

```bash
curl -s "https://crm.pashavin.ru/rest/metadata/fields?filter=objectMetadataId[eq]:<object-uuid>" \
  -H "Authorization: Bearer $TWENTY_CRM_API_KEY"
```

## Common Mistakes

**REST path is /rest/<entity>, NOT /rest/core/<entity>**
- `/rest/people` works
- `/rest/core/people` does NOT work

**GraphQL introspection is disabled**
- Cannot use `__schema` queries
- Use REST OpenAPI spec instead: `/open-api/core`

**API key in env file needs sourcing**
- `source /home/deploy/.env.twenty` before using `$TWENTY_CRM_API_KEY`

**Not backing up before schema changes**
- Always `pg_dump` before modifying fields/objects via Metadata API

**Rate limiting**
- 100 req/min limit; use batch operations (up to 60 records per request) for bulk work
