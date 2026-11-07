# python-telegram-bot (PTB) v21+ Notes

Prefer **aiogram 3** for new projects on Railway (cleaner aiohttp integration + secret-token handling).  
Use PTB when you already have existing PTB code or team preference.

## Minimal webhook skeleton differences vs aiogram

PTB’s built-in webhook support runs its own server. Adding a reliable `/healthz` is slightly more awkward than aiogram’s `SimpleRequestHandler` + aiohttp.

Recommended pattern for Railway:

1. Use `Application.builder().token(...).build()`
2. Register handlers
3. On startup call `application.bot.set_webhook(...)` with `secret_token` and current `RAILWAY_PUBLIC_DOMAIN`
4. Run with `application.run_webhook(...)` **or** mount the PTB updater into an aiohttp/Starlette app so you control `/healthz` yourself.

### Simple (PTB-managed) example

```python
import os
from telegram.ext import Application, CommandHandler, ContextTypes
from telegram import Update

TOKEN = os.environ["BOT_TOKEN"]
PORT = int(os.environ.get("PORT", 8080))
DOMAIN = os.environ.get("RAILWAY_PUBLIC_DOMAIN")
SECRET = os.environ.get("WEBHOOK_SECRET")

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Hi from PTB")

def main():
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))

    if DOMAIN:
        url = f"https://{DOMAIN}/webhook"
        app.run_webhook(
            listen="0.0.0.0",
            port=PORT,
            url_path="webhook",
            webhook_url=url,
            secret_token=SECRET,
            drop_pending_updates=True,
        )
    else:
        app.run_polling(drop_pending_updates=True)

if __name__ == "__main__":
    main()
```

### Healthcheck note

`run_webhook` does not expose a separate health route by default. For strict Railway healthchecks, either:

- Switch to aiogram, or
- Mount the PTB application inside an aiohttp/FastAPI app and add `/healthz` yourself.

## When to choose PTB

- Existing large PTB codebase
- Heavy use of `ConversationHandler` / complex job queues that you already know
- Team already proficient in PTB

Otherwise start with the aiogram 3 template in `references/aiogram3.md`.
