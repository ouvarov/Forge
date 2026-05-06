# Forge MCP Server

**Re-Engagement Reasoning Engine** — per-user push reasoning with closed outcome feedback loop.

Built for the Promova Product Engineer Challenge, May 5–6, 2026.

> **Status (2026-05-04):** scaffolding complete, MCP surface live (1 working tool + 3 stubs + 5 prompts). Core service bodies pending. See [PLAN.md §11](./PLAN.md#11-поточний-статус-live) for the live file-by-file checklist.

---

## What this is

A NestJS MCP server that acts as **long-term memory + brand-voice keeper** for Claude Desktop (or any MCP client) when generating personalized re-engagement messages (push + email).

forge does NOT:
- Pull user data from external systems (Walhalla, Amplitude) — Claude Desktop does that
- Generate message text itself — Claude Desktop's model does that
- Send messages — out of scope for MVP, would integrate with Customer.io transactional API in production (push + email)

forge is designed to (MVP target):
- Persist every generated attempt (push or email) with full reasoning, token usage, predicted confidence
- Group attempts deterministically by `group_key` (e.g. `promova:es:B1:churned-7-14d:lesson-food`); channel is a variant inside the bucket, not part of the key
- Aggregate win-rate per group AND per format (push vs email) and return top winners + losers as in-context examples for the next composition
- Measure outcomes after the domain window elapses, recompute reward scores, close the loop
- Expose brand-voice prompts (`promova_voice` for push, `promova_email_voice` for email) and recipe prompts (`forge_promova_collect`) via MCP

**Demo path:** Claude Desktop is the MCP client. You ask Claude in plain language to find candidates and generate messages — Claude orchestrates Walhalla / Amplitude MCPs to gather data, then calls forge tools to learn from past outcomes and persist new attempts (push and/or email). Reasoning, tokens, and cost are visible in chat.

**Production path (post-MVP):** add `orchestration/ingestion.service.ts` (Walhalla/Amplitude REST clients), `orchestration/composer.service.ts` (Anthropic SDK), `cron/engagement.cron.ts` — same core services and DB schema, no rewrite. See [PLAN.md](./PLAN.md) §10 for migration delta.

---

## Quick start

### 1. Install
```bash
yarn install
```

The repo includes `.env` with `DATABASE_URL=postgresql://forge:forge@localhost:5434/forge`. Override only if needed.

### 2. Postgres + pgvector
```bash
yarn docker:up
docker compose logs postgres   # verify
```
Postgres listens on `localhost:5434` (5432/5433 are typically occupied by other local DBs).

### 3. Database migrations
```bash
yarn db:migrate   # apply existing migration (0000_exotic_tarantula.sql)
yarn db:studio    # optional: visual table inspector
```
When the schema evolves (e.g. enriched ContextSnapshot fields land), regenerate via `yarn db:generate` then re-run `yarn db:migrate`.

### 4. Build
```bash
yarn build
```
Output: `dist/src/main.js`. Note the `/src/` segment — `drizzle.config.ts` at repo root forces tsc rootDir to be the repo root.

### 5. Run (dev mode)
```bash
yarn start:dev
```
The server speaks MCP over stdio. To verify without Claude Desktop:
```bash
yarn mcp:inspect
```

---

## Connect to Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```jsonc
{
  "mcpServers": {
    "forge": {
      "command": "/Users/<you>/.nvm/versions/node/v22.22.0/bin/node",
      "args": [
        "/ABSOLUTE/PATH/TO/forge-mcp-server/dist/src/main.js"
      ],
      "env": {
        "DATABASE_URL": "postgresql://forge:forge@localhost:5434/forge"
      }
    }
  }
}
```

After editing:
1. `yarn build` — Claude Desktop runs the compiled `dist/src/main.js`, not source
2. **Cmd+Q** Claude Desktop fully (not just close window) and reopen
3. In a new chat, the 🔌 icon should list `forge` with **6 tools** (forge_ping, forge_log_attempt, forge_get_group_stats, forge_measure_outcome, forge_check_pending_outcomes, forge_simulate_outcomes) and **5 prompts**
4. Quick check: ask "Use forge_ping to check if the server is alive"

No API keys required for the demo path. Walhalla / Amplitude MCPs are exposed to Claude Desktop separately via your org's claude.ai gateway.

---

## MCP surface

### Tools

| Tool | Status | When Claude calls it |
|---|---|---|
| `forge_ping` | ✅ live | Smoke test — echoes input + server time |
| `forge_get_group_stats` | 🟡 stub | BEFORE composing — to learn from past attempts in the same user-group |
| `forge_log_attempt` | 🟡 stub | AFTER composing — persists context + reasoning + push + tokens |
| `forge_measure_outcome` | 🟡 stub | After domain outcome window — records reactivation flag + recomputes reward score |
| `forge_simulate_outcomes` | ✅ live | DEMO ONLY — bulk update outcomes to demo the learning shift between iterations |

Status legend: ✅ live = body written and tested · 🟡 stub = registered, returns fake data · ❌ planned = not yet implemented

### Prompts

| Prompt | Purpose |
|---|---|
| `forge_workflow` | Step-by-step guide: which external MCPs to call, in what order, how to assemble contextSnapshot |
| `forge_promova_collect` | Recipe: which Walhalla/Amplitude tools to call for a Promova user |
| `forge_promova_measure` | Recipe: how to close the outcome loop after a campaign matures |
| `promova_voice` | Brand voice + safety rules for Promova **push** composition |
| `promova_email_voice` | Brand voice + email format rules (subject/preheader/body/unsubscribe) for Promova **email** composition |

### Server-level instructions

forge advertises capabilities + a comprehensive `instructions` string in the `initialize` response. Claude Desktop reads it as system context when the MCP first connects.

---

## Architecture

```
Claude Desktop (MVP)         Cron + Anthropic API (production)
        │                              │
        └────── MCP stdio ─────────────┘
                       ↓
        ┌────────────────────────────────────┐
        │ forge NestJS app                   │
        │  ── MCP server (tools + prompts)   │
        │  ── Core services                  │
        │       LoggerService                │
        │       AggregatorService            │
        │       MeasurerService              │
        │  ── Domain adapter                 │
        │       PromovaAdapter (window 7d)   │
        │  ── Drizzle ORM                    │
        └────────────────────────────────────┘
                       ↓
                Postgres 16 + indexes
                (group_key + outcome)
```

External data sources (Walhalla, Amplitude, Zendesk) are accessed by **Claude Desktop**, not forge. forge receives already-assembled `contextSnapshot` objects via MCP.

The DomainAdapter pattern keeps the engine product-agnostic: a new product means one new adapter (~30 lines) and a brand-voice prompt — no changes to core services or DB schema.

### Layered services (see PLAN.md §2)

```
Layer 4 — ORCHESTRATION (post-MVP only)
  └─ ingestion.service / composer.service / cron — added on top, no rewrite

Layer 3 — INTERFACES
  └─ MCP tools (thin callers): log_attempt, get_group_stats, measure_outcome,
     simulate_outcomes (demo only), ping

Layer 2 — CORE BUSINESS LOGIC (no MCP knowledge, no Claude knowledge)
  └─ LoggerService, AggregatorService, MeasurerService

Layer 1 — DOMAIN CONFIG
  └─ DomainAdapter (computeGroupKey, outcomeWindowDays, activationEvents)
       PromovaAdapter (window 7d) — only adapter shipping in MVP
```

To add a new product: implement `DomainAdapter`, register in `AdaptersModule`. ~30 lines.

---

## Stack

- **NestJS 10** + Yarn
- **Drizzle ORM** + drizzle-kit
- **Postgres 16** (pgvector image — `vector` extension installed but not used in MVP, available for future RAG)
- **@modelcontextprotocol/sdk** (stdio transport)
- **Zod** for tool input validation
- **Joi** for env validation
- **NO Anthropic SDK in MVP** — Claude Desktop generates pushes; SDK returns when production cron is added

---

## Documentation

| File | Purpose |
|---|---|
| [`PLAN.md`](./PLAN.md) | Execution plan — what we build, in what order, with what API |
| [`WHY.md`](./WHY.md) | Business case — real Slack citations, real numbers, why now |
| [`DEMO.md`](./DEMO.md) | 15-minute demo script — 8-step learning loop, speaker notes |
| [`RESEARCH.md`](./RESEARCH.md) | Production scaling — how forge gives each user a personal push at scale (cost, infra, fatigue) |
| [`ANALYTICS.md`](./ANALYTICS.md) | Event taxonomy + A/B framework — how we prove forge is better than baseline pushes |

---

## License

UNLICENSED — internal Promova project.