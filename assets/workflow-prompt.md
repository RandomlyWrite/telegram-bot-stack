# Telegram Bot Workflow Prompt

Copy-paste this prompt when starting a new bot or asking an agent to scaffold/deploy one. It encodes the full stack and workflow.

---

**Prompt:**

```
You are helping me build and deploy a Telegram bot using this exact stack and workflow.

STACK RULES
- Source: private GitHub repo
- Host: Railway (prefer webhook mode in production)
- Data (if needed): Supabase Postgres or Railway Postgres
- Secrets: BOT_TOKEN, WEBHOOK_SECRET, SUPABASE_SERVICE_ROLE_KEY live only in Railway Variables — never commit them
- Python preferred library: aiogram 3
- Node preferred library: grammY
- Frontend / Mini App (if needed): TypeScript + React + Vite
- API (if separate web service): Express first, Fastify later
- ORM: Drizzle
- 2D games: Phaser (fuller) or PixiJS (custom)
- Background jobs: Postgres first; Redis + BullMQ only when justified
- Real-time: WebSockets or Supabase Realtime; server is source of truth for scores/outcomes
- AI agents: LLM calls stay server-side; rate-limit; treat model output as untrusted until validated

WORKFLOW (follow in order)
Phase 0 – Decide shape: pure bot vs bot+Mini App vs game. Choose language.
Phase 1 – Scaffold: BotFather → private repo → correct template → .gitignore + lockfile + .env.example
Phase 2 – Local first: MODE=polling, verify /start works before any deploy
Phase 3 – GitHub → Railway: push, set BOT_TOKEN + MODE=webhook + WEBHOOK_SECRET, confirm domain, deploy, check logs for webhook registration + /healthz 200
Phase 4 – Webhook validation: auto-register on boot with RAILWAY_PUBLIC_DOMAIN + secret_token. On every request validate X-Telegram-Bot-Api-Secret-Token (reject 403 on mismatch). Health endpoint always 200.
Phase 5 – Data: telegram_id primary key, service_role only on server, RLS on, important writes server-side only
Phase 6 – Mini App (optional): separate Railway service from bot, validate initData HMAC, never trust client user id or scores
Phase 7 – Iterate: local → push → watch logs. After domain changes re-check getWebhookInfo. Revoke token immediately if leaked.
Phase 8 – Real-time / AI: server authority for game state; LLM keys never in the client

CONSTRAINTS
- Prefer webhook over polling in production
- Single replica while on polling
- No secrets in git history
- drop_pending_updates=True on first production webhook set unless we explicitly want the backlog
- Keep bot process and web/API process separate once complexity grows

My bot goal: [DESCRIBE THE BOT HERE]

Language preference: [Python / Node / either]
Needs Mini App or game UI: [yes/no]
Needs persistent data: [yes/no]
Needs real-time features: [yes/no]
Needs AI/LLM: [yes/no]

Start at Phase 0 and walk me through the next concrete steps. When generating code, use the production templates (webhook + secret token + healthz). Do not invent extra infrastructure unless the requirements justify it.
```

---

**How to use**
1. Paste the prompt.
2. Fill the bracketed fields at the bottom.
3. Let the agent drive from Phase 0.
4. When it generates code, insist it follows the webhook + secret-token + healthz patterns from the skill templates.
