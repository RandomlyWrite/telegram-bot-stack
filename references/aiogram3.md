# aiogram 3 Template (Railway + optional Supabase)

Minimal, production-ready aiogram 3.x skeleton that works on Railway out of the box.
Webhook-first. Auto-registers webhook on boot. Secret-token validation. `/healthz`. Optional Supabase.

## File Layout

```
bot/
├── main.py                 # entrypoint (webhook or polling)
├── handlers/
│   ├── __init__.py
│   ├── start.py
│   └── common.py
├── middlewares/            # optional
├── services/
│   └── db.py               # Supabase client (optional)
├── requirements.txt
├── .env.example
└── .gitignore
```

## requirements.txt

```
aiogram>=3.13,<4
aiohttp>=3.9
python-dotenv>=1.0
supabase>=2.0          # optional
```

## main.py (complete working skeleton)

```python
import asyncio
import logging
import os
from contextlib import asynccontextmanager

from aiohttp import web
from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.types import Update
from aiogram.webhook.aiohttp_server import SimpleRequestHandler, setup_application
from dotenv import load_dotenv

load_dotenv()  # local only; Railway uses Variables

# --- config ---
BOT_TOKEN = os.getenv("BOT_TOKEN") or os.getenv("TELEGRAM_BOT_TOKEN")
WEBHOOK_SECRET = os.getenv("WEBHOOK_SECRET", "")
MODE = os.getenv("MODE", "webhook").lower()  # "webhook" | "polling"
PORT = int(os.getenv("PORT", "8080"))
WEBHOOK_PATH = "/webhook"
BASE_WEBHOOK_URL = os.getenv("RAILWAY_PUBLIC_DOMAIN")  # injected by Railway

if not BOT_TOKEN:
    raise RuntimeError("BOT_TOKEN / TELEGRAM_BOT_TOKEN is required")

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

bot = Bot(
    token=BOT_TOKEN,
    default=DefaultBotProperties(parse_mode=ParseMode.HTML),
)
dp = Dispatcher()

# --- register routers ---
from handlers.start import router as start_router
from handlers.common import router as common_router

dp.include_router(start_router)
dp.include_router(common_router)


async def on_startup(bot: Bot) -> None:
    if MODE == "webhook" and BASE_WEBHOOK_URL:
        url = f"https://{BASE_WEBHOOK_URL}{WEBHOOK_PATH}"
        await bot.set_webhook(
            url=url,
            secret_token=WEBHOOK_SECRET or None,
            drop_pending_updates=True,
        )
        logger.info("Webhook set to %s", url)
    else:
        logger.info("Running in polling mode (or missing RAILWAY_PUBLIC_DOMAIN)")


async def on_shutdown(bot: Bot) -> None:
    await bot.delete_webhook(drop_pending_updates=False)
    await bot.session.close()
    logger.info("Shutdown complete")


async def healthz(request: web.Request) -> web.Response:
    return web.json_response({"status": "ok"})


def main() -> None:
    if MODE == "polling":
        # Local / low-traffic only
        async def run_polling():
            await on_startup(bot)
            try:
                await dp.start_polling(bot, drop_pending_updates=True)
            finally:
                await on_shutdown(bot)

        asyncio.run(run_polling())
        return

    # --- webhook mode (Railway default) ---
    app = web.Application()
    app.router.add_get("/healthz", healthz)
    app.router.add_get("/", healthz)  # some platforms probe /

    # Secret-token validation is handled by SimpleRequestHandler
    webhook_handler = SimpleRequestHandler(
        dispatcher=dp,
        bot=bot,
        secret_token=WEBHOOK_SECRET or None,
    )
    webhook_handler.register(app, path=WEBHOOK_PATH)
    setup_application(app, dp, bot=bot)

    # Register startup / shutdown hooks
    async def _on_startup(app: web.Application):
        await on_startup(bot)

    async def _on_shutdown(app: web.Application):
        await on_shutdown(bot)

    app.on_startup.append(_on_startup)
    app.on_shutdown.append(_on_shutdown)

    web.run_app(app, host="0.0.0.0", port=PORT)


if __name__ == "__main__":
    main()
```

## handlers/start.py

```python
from aiogram import Router, F
from aiogram.types import Message, CallbackQuery
from aiogram.filters import CommandStart, Command
from aiogram.utils.keyboard import InlineKeyboardBuilder

router = Router(name="start")


@router.message(CommandStart())
async def cmd_start(message: Message) -> None:
    kb = InlineKeyboardBuilder()
    kb.button(text="Help", callback_data="help")
    kb.button(text="Ping", callback_data="ping")
    kb.adjust(2)

    await message.answer(
        f"Hi, <b>{message.from_user.full_name}</b>.\n"
        f"Bot is alive on the stack.",
        reply_markup=kb.as_markup(),
    )


@router.message(Command("help"))
@router.callback_query(F.data == "help")
async def cmd_help(event: Message | CallbackQuery) -> None:
    text = (
        "/start — greeting\n"
        "/help — this message\n"
        "/id — your Telegram ID"
    )
    if isinstance(event, CallbackQuery):
        await event.message.edit_text(text)
        await event.answer()
    else:
        await event.answer(text)


@router.callback_query(F.data == "ping")
async def cb_ping(callback: CallbackQuery) -> None:
    await callback.answer("pong", show_alert=False)
    await callback.message.answer("pong")
```

## handlers/common.py

```python
from aiogram import Router
from aiogram.types import Message
from aiogram.filters import Command

router = Router(name="common")


@router.message(Command("id"))
async def cmd_id(message: Message) -> None:
    await message.answer(f"Your Telegram ID: <code>{message.from_user.id}</code>")


@router.message()
async def echo_fallback(message: Message) -> None:
    # Optional: remove or replace with your real handlers
    await message.answer("Unknown command. Try /help")
```

## Optional: services/db.py (Supabase)

```python
import os
from supabase import create_client, Client

_url = os.getenv("SUPABASE_URL")
_key = os.getenv("SUPABASE_SERVICE_ROLE_KEY")

supabase: Client | None = None
if _url and _key:
    supabase = create_client(_url, _key)


async def upsert_user(telegram_id: int, username: str | None, first_name: str | None) -> None:
    if not supabase:
        return
    supabase.table("users").upsert({
        "telegram_id": telegram_id,
        "username": username,
        "first_name": first_name,
    }).execute()
```

Call `upsert_user` from your `/start` handler if desired.

## Railway Variables (minimum)

```
BOT_TOKEN=...
MODE=webhook
WEBHOOK_SECRET=<openssl rand -hex 32>
# Railway injects PORT + RAILWAY_PUBLIC_DOMAIN
```

Optional:

```
SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...
```

## Local testing

```bash
export BOT_TOKEN=...
export MODE=polling
python main.py
```

For webhook locally you need a tunnel (ngrok / cloudflared) and set `RAILWAY_PUBLIC_DOMAIN` (or a custom env) to the tunnel host.

## Notes specific to aiogram 3 on Railway

- Prefer `SimpleRequestHandler` — it embeds cleanly into aiohttp and does secret-token validation for you.
- Always bind `0.0.0.0` and use the injected `PORT`.
- Re-register webhook on every startup using the current `RAILWAY_PUBLIC_DOMAIN`. Domains can change.
- `/healthz` must return 200 even if the bot is still starting; Railway healthchecks gate the deploy.
- Keep a single replica while you are still on polling. Webhook can scale later.
- `drop_pending_updates=True` on first production webhook set is usually what you want so the bot does not answer a backlog of messages from while it was offline.
