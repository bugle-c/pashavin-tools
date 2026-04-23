# pashavin-tools

Личный плагин Claude Code для Паши Вин. Один источник правды для skills, subagents и MCP-конфигов на всех устройствах (Hetzner, Mac, новый ноут).

## Что внутри

**Skills (20)** — в `skills/`:

| Категория | Skills |
|---|---|
| Проектные | `pashavin-blog`, `pashavin-portfolio`, `pashavin-seo` |
| Инфра | `dokploy-deploy`, `supabase`, `twenty-crm`, `pipeline` |
| SEO фреймворк | `seo`, `seo-audit`, `seo-page`, `seo-plan`, `seo-programmatic`, `seo-content`, `seo-technical`, `seo-schema`, `seo-sitemap`, `seo-hreflang`, `seo-images`, `seo-competitor-pages`, `seo-geo` |

**Subagents (8)** — в `agents/`:

`dokploy-deployer`, `seo-content`, `seo-geo`, `seo-performance`, `seo-schema`, `seo-sitemap`, `seo-technical`, `seo-visual`

**MCP servers (5)** — в `.mcp.json`:

- `exa` — web search (требует `EXA_API_KEY`)
- `chrome-devtools` — управление Chrome
- `gemini-design` — Gemini для дизайна (требует `GEMINI_API_KEY` + `GEMINI_DESIGN_MCP_PATH`)
- `gemini` — Gemini для кода (требует `GEMINI_API_KEY`)
- `twenty-crm` — Twenty CRM MCP (требует `TWENTY_CRM_MCP_PATH`)

## Установка

**Добавить маркетплейс:**

```
/plugin marketplace add bugle-c/pashavin-tools
```

**Установить плагин:**

```
/plugin install pashavin-tools@bugle-c
```

## Env переменные (настроить на каждой машине)

Добавь в `~/.bashrc` / `~/.zshrc`:

```bash
export EXA_API_KEY="ed1f9a61-..."
export GEMINI_API_KEY="AIzaSy..."
export GEMINI_DESIGN_MCP_PATH="/path/to/gemini-design-mcp"
export TWENTY_CRM_MCP_PATH="/path/to/twenty-crm-mcp-server"
```

Проектам `gemini-design-mcp` и `twenty-crm-mcp-server` установи деп-ы отдельно — они лежат в `bugle-c/gemini-design-mcp` и `bugle-c/twenty-crm-mcp-server` (или аналог).

## Обновление

```
/plugin update pashavin-tools
```

## История

Создан 2026-04-23 при переезде Latvia → Hetzner для мульти-девайс синка (ранее skills жили только в `~/.claude/skills/` на одном сервере).
