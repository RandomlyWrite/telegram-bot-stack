---
name: telegram-bot-stack
description: Use when building, scaffolding, deploying, or debugging Telegram bots that use GitHub for source and CI, Railway for hosting, and/or Supabase for database, auth, storage, or realtime. Triggers on Railway deploy, webhook vs polling, BOT_TOKEN secrets, Supabase + Telegram, Mini App backends, aiogram/python-telegram-bot/grammY/telegraf stacks, or deploy my bot to Railway.
---

# Telegram Bot Stack (GitHub + Railway + Supabase)

## Overview

Practical playbook for shipping Telegram bots that live on GitHub, run on Railway, and optionally persist state in Supabase. Covers scaffolding, secrets, polling vs webhook, health checks, common 409/401/5xx failures, and clean Supabase patterns. Prefer webhook + auto-registration on Railway for production; keep polling only for local or very low-traffic bots.

## Core Stack Decisions

| Layer | Role | Default choice |
|-------|------|----------------|
| Source | Code, PRs, secrets via env only | GitHub private repo |
| Host | Always-on process or webhook receiver | Railway (auto-detect Python/Node) |
| Data | Users, scores, sessions, logs | Supabase Postgres (or Railway Postgres if simple) |
| Secrets | BOT_TOKEN, SUPABASE_SERVICE_ROLE_KEY | Railway Variables only — never commit |

**Never** put BOT_TOKEN, service_role keys, or webhook secrets in code, README, or committed .env.

## Decision Tree — Polling vs Webhook

1. Local development or single-user test → **polling**.
2. Production on Railway → **webhook** (lower idle cost, survives restarts cleanly, required for multi-replica safety).
3. If you need multiple replicas or zero-downtime deploys → webhook only. Two pollers on the same token = 409 Conflict.

Railway injects `PORT` and usually `RAILWAY_PUBLIC_DOMAIN`. Use them.

## Telegram Bot Workflow

End-to-end operational flow from idea to production and iteration.

### Phase 0 — Decide the shape
1. Pure command / notification bot → start with aiogram 3 or grammY only.
2. Bot + Mini App or game → plan for split services early (see Expanded stack).
3. Needs persistent users/scores → add Supabase (or Railway Postgres) from day one.
4. Choose language: Python (aiogram preferred) or Node (grammY preferred). Use the matching template in `references/`.

### Phase 1 — Scaffold
1. Talk to @BotFather → `/newbot` → copy token. **Never** paste the token into chat, commit it, or put it in a public gist.
2. Create a **private** GitHub repo.
3. Copy the appropriate template (`references/aiogram3.md` or `references/grammy.md`).
4. Add `.gitignore` (must include `.env`, `node_modules`, `__pycache__`, `.venv`).
5. Add a lockfile (`requirements.txt` / `package-lock.json` or `pnpm-lock.yaml`).
6. Create `.env.example` with documented variable names (no real secrets).

### Phase 2 — Local first
1. Copy `.env.example` → `.env` and fill `BOT_TOKEN`.
2. Set `MODE=polling`.
3. Run the bot locally and verify `/start` (and any core commands) work.
4. Only after local success proceed to deploy. A broken local bot will just be a broken Railway service.

### Phase 3 — GitHub → Railway
1. Push to `main`.
2. Railway → New Project → Deploy from GitHub repo.
3. Add Variables:
   - `BOT_TOKEN` (or `TELEGRAM_BOT_TOKEN`)
   - `MODE=webhook`
   - `WEBHOOK_SECRET` = `openssl rand -hex 32`
   - Supabase keys if used (`SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`)
4. Confirm or generate a public domain (Settings → Networking).
5. Deploy. Immediately open Deploy Logs and confirm:
   - Process started
   - Webhook registration succeeded (look for the setWebhook log line)
   - `/healthz` is responding 200

### Phase 4 — Webhook registration & validation
- Prefer **auto-registration on boot** using the current `RAILWAY_PUBLIC_DOMAIN`.
- Generate a strong secret once: `openssl rand -hex 32` → store as `WEBHOOK_SECRET` in Railway Variables.
- Call `setWebhook` with both the full HTTPS URL **and** `secret_token`.
- On every incoming POST to the webhook path:
  - Read the header `X-Telegram-Bot-Api-Secret-Token`.
  - Compare it (constant-time if possible) to `WEBHOOK_SECRET`.
  - Reject with **403** (or 401) if missing or mismatched. Do not process the update.
- Expose `GET /healthz` (and preferably `/`) that returns 200 even during brief startup.
- Keep a single replica while using polling. Webhook can scale later.
- After deploy or domain change, verify with `getWebhookInfo`.

**aiogram 3** (handled by `SimpleRequestHandler` when you pass `secret_token`):
```python
webhook_handler = SimpleRequestHandler(
    dispatcher=dp,
    bot=bot,
    secret_token=WEBHOOK_SECRET or None,
)
```

**grammY**:
```js
const handleUpdate = webhookCallback(bot, "http", {
  secretToken: WEBHOOK_SECRET || undefined,
});
```

Manual check pattern (any framework):
```
if (header !== process.env.WEBHOOK_SECRET) → 403
```

Never skip secret-token validation in production. It is the only practical way to stop random internet traffic from injecting fake updates.

### Phase 5 — Data layer (when needed)
1. Create Supabase project (or Railway Postgres).
2. Put only the **service_role** key in Railway Variables.
3. Use `telegram_id` as primary key for users.
4. All important writes (scores, inventory, game outcomes) happen server-side.
5. Enable RLS; bot bypasses via service_role. Mini App uses anon key + policies.

### Phase 6 — Mini App / web service (optional)
1. Keep the bot process and the web/API process as **separate Railway services** once complexity grows.
2. Validate Telegram `initData` (HMAC with bot token) on every authenticated request.
3. Never trust client-reported user id or scores for writes.
4. Frontend default: TypeScript + React + Vite. Add Radix UI only when needed.
5. 2D games: Phaser (fuller framework) or PixiJS (custom rendering).

### Phase 7 — Iterate & operate
1. Every change: local test → push → watch Railway logs.
2. After domain or major config changes: confirm webhook is still registered (`getWebhookInfo`).
3. Token leak response: revoke via BotFather immediately, update Railway variable, redeploy.
4. Prefer small, reversible changes. Kill features that don’t get used within a session or two.
5. When background work appears: start with Postgres jobs; only add Redis + BullMQ when retries or volume demand it.

### Phase 8 — Real-time games & AI agents (when relevant)
- **Real-time games**: WebSockets (or Supabase Realtime). Server is source of truth for scores, turns, and outcomes. Client can be optimistic; server rejects invalid state.
- **AI agents**: Keep LLM calls in the bot or a dedicated worker service. Never put API keys in the Mini App. Log prompts/responses for debugging, rate-limit per user, and treat model output as untrusted until validated.

See `references/expanded-stack.md` for deeper architecture and the language-specific templates for copy-paste starters.

## Common Failure Modes & Fixes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| 409 Conflict | Two processes polling same token | Kill local process; switch production to webhook; single replica |
| Bot silent after deploy | Webhook not registered or wrong URL | Check deploy logs for setWebhook; confirm RAILWAY_PUBLIC_DOMAIN |
| 401 / unauthorized | Token wrong or revoked | Re-issue via BotFather, update Railway variable, redeploy |
| Healthcheck failing | No /healthz or process crashes on missing env | Add always-200 health route; make token optional at import time |
| Supabase 401 / RLS | Using anon key for server writes | Switch to service_role on Railway only |
| Cold starts / missed messages | Free-tier sleep (Render) or multi-poller | Stay on Railway; use webhook |
| Domain changes break webhook | Hard-coded URL | Always re-register on startup using current public domain |

## Recommended Project Layout

```
bot/
├── main.py / index.js          # entry + webhook/polling switch
├── handlers/                   # command & callback routers
├── db/ or services/            # Supabase client + queries
├── .env.example                # documented vars, no secrets
├── requirements.txt / package.json
├── Dockerfile                  # optional; Railway auto-detects most cases
└── README.md                   # setup steps only
```

## Language Notes

| Layer / Language | Preferred | Template / Notes |
|------------------|-----------|------------------|
| **Python bot** | **aiogram 3** | `references/aiogram3.md` — best Railway fit |
| Python bot (alt) | python-telegram-bot v21+ | `references/ptb.md` |
| **Node bot** | **grammY** | `references/grammy.md` |
| Node bot (alt) | telegraf | Still fine; grammY preferred for new work |
| **Frontend / Mini App** | TypeScript + React + Vite | See `references/expanded-stack.md`. Add Radix UI when needed |
| **API (web service)** | Express → Fastify later | Express is straightforward; Fastify for higher structure/throughput |
| **ORM** | Drizzle | Typed schema + migrations on PostgreSQL |

Bot runtimes (Python/Node) work on Railway without a Dockerfile when lockfile + start command are clear. For fuller products, split bot worker and web service.

## Security Checklist (run before every public deploy)

- [ ] BOT_TOKEN only in Railway Variables
- [ ] service_role key only in Railway Variables
- [ ] Webhook secret validated on every request
- [ ] No secrets in Git history (check with `git log -p` if unsure)
- [ ] .env in .gitignore
- [ ] drop_pending_updates=True on first webhook set if backlog is unwanted
- [ ] Single replica for polling mode

## When to Reach for References

- **aiogram 3** full production skeleton → `references/aiogram3.md`
- **grammY (Node)** full production skeleton → `references/grammy.md`
- python-telegram-bot notes → `references/ptb.md`
- Expanded stack (React/Vite, Express/Fastify, Drizzle, split services) → `references/expanded-stack.md`
- Railway env vars & start commands → `references/railway.md`
- Supabase schema + RLS → `references/supabase.md`
- Safe `.env.example` → `assets/env.example`
- **Reusable workflow prompt** (paste into a new chat) → `assets/workflow-prompt.md`

## Anti-patterns

- Committing tokens "just for the first deploy"
- Running polling + webhook on the same token
- Putting service_role in Mini App client code
- Assuming Railway public domain never changes
- Skipping health endpoint on webhook bots
- Using free sleeping hosts (Render free) for production bots

Ship small, watch the logs, kill what does not respond within one session.
