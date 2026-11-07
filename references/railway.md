# Railway Specifics for Telegram Bots

## Required / Useful Variables

| Variable | Purpose | Notes |
|----------|---------|-------|
| `BOT_TOKEN` or `TELEGRAM_BOT_TOKEN` | BotFather token | Required. Never log the full value. |
| `PORT` | Listen port | Injected by Railway. Bind 0.0.0.0:$PORT. |
| `RAILWAY_PUBLIC_DOMAIN` | Public hostname | Use for webhook URL construction. |
| `WEBHOOK_SECRET` | Secret token for setWebhook | Validate header `X-Telegram-Bot-Api-Secret-Token`. |
| `MODE` | `polling` or `webhook` | Optional switch in your entrypoint. |
| `SUPABASE_URL` | Project URL | From Supabase dashboard. |
| `SUPABASE_SERVICE_ROLE_KEY` | Server key | Railway only. |
| `SUPABASE_ANON_KEY` | Client key | Only if Mini App needs it (rarely). |

## Start Command Hints

Railway auto-detects:

- Python: looks for `requirements.txt` + common entry points, or Procfile / railway.toml
- Node: `package.json` scripts.start or main field

Explicit examples:

```toml
# railway.toml
[build]
builder = "NIXPACKS"

[deploy]
startCommand = "python -m bot"
healthcheckPath = "/healthz"
healthcheckTimeout = 30
restartPolicyType = "ON_FAILURE"
```

Or in package.json:

```json
"scripts": {
  "start": "node dist/index.js"
}
```

## Webhook Auto-Registration Best Practice

On every boot:

1. Read current `RAILWAY_PUBLIC_DOMAIN`
2. Call `setWebhook` with the full HTTPS URL + secret
3. Log success/failure (never the token)
4. Proceed to serve `/webhook` and `/healthz`

This survives domain regeneration and redeploys.

## Single Replica Rule

For long-polling mode set replica count = 1. Multiple pollers fighting over getUpdates produce 409 Conflict and dropped updates.

Webhook mode can scale horizontally if your handlers are idempotent and state lives in Supabase.

## Logs & Debugging

- Railway Deployments → latest → Logs
- Look for "Webhook set" or "Starting polling"
- 409 → kill local bot or other services using the same token
- Missing variable → process usually exits with clear error if you check env at startup

## Cost Notes (2026)

Railway has trial credits then usage-based. Webhook bots stay cheap because idle CPU is near zero between messages. Polling burns more.
