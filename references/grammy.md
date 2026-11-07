# grammY (Node.js) Template (Railway + optional Supabase)

Modern Node.js Telegram bot skeleton using grammY. Webhook-first, auto-registers on boot, secret-token validation, `/healthz`. Optional Supabase.

## File Layout

```
bot/
├── index.js                 # or index.ts (entry)
├── handlers/
│   ├── start.js
│   └── common.js
├── services/
│   └── db.js                # optional Supabase
├── package.json
├── .env.example
└── .gitignore
```

## package.json

```json
{
  "name": "telegram-bot",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node index.js",
    "dev": "node --watch index.js"
  },
  "dependencies": {
    "grammy": "^1.30.0",
    "dotenv": "^16.4.0",
    "@supabase/supabase-js": "^2.45.0"
  },
  "engines": {
    "node": ">=20"
  }
}
```

## index.js (complete working skeleton)

```js
import "dotenv/config";
import { Bot, webhookCallback } from "grammy";
import { createServer } from "node:http";
import { startRouter } from "./handlers/start.js";
import { commonRouter } from "./handlers/common.js";

// --- config ---
const BOT_TOKEN = process.env.BOT_TOKEN || process.env.TELEGRAM_BOT_TOKEN;
const WEBHOOK_SECRET = process.env.WEBHOOK_SECRET || "";
const MODE = (process.env.MODE || "webhook").toLowerCase();
const PORT = Number(process.env.PORT || 8080);
const WEBHOOK_PATH = "/webhook";
const BASE_WEBHOOK_URL = process.env.RAILWAY_PUBLIC_DOMAIN; // injected by Railway

if (!BOT_TOKEN) {
  throw new Error("BOT_TOKEN / TELEGRAM_BOT_TOKEN is required");
}

const bot = new Bot(BOT_TOKEN);

// --- routers ---
bot.use(startRouter);
bot.use(commonRouter);

// Catch-all error handler
bot.catch((err) => {
  console.error("Bot error:", err);
});

async function setWebhook() {
  if (MODE !== "webhook" || !BASE_WEBHOOK_URL) {
    console.log("Running in polling mode (or missing RAILWAY_PUBLIC_DOMAIN)");
    return;
  }
  const url = `https://${BASE_WEBHOOK_URL}${WEBHOOK_PATH}`;
  await bot.api.setWebhook(url, {
    secret_token: WEBHOOK_SECRET || undefined,
    drop_pending_updates: true,
  });
  console.log(`Webhook set to ${url}`);
}

async function main() {
  if (MODE === "polling") {
    await setWebhook(); // just logs
    await bot.start({
      drop_pending_updates: true,
      onStart: () => console.log("Polling started"),
    });
    return;
  }

  // --- webhook mode ---
  await setWebhook();

  const handleUpdate = webhookCallback(bot, "http", {
    secretToken: WEBHOOK_SECRET || undefined,
  });

  const server = createServer(async (req, res) => {
    // Health checks
    if (req.method === "GET" && (req.url === "/healthz" || req.url === "/")) {
      res.writeHead(200, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ status: "ok" }));
      return;
    }

    // Webhook
    if (req.method === "POST" && req.url === WEBHOOK_PATH) {
      try {
        await handleUpdate(req, res);
      } catch (err) {
        console.error("Webhook error:", err);
        if (!res.headersSent) {
          res.writeHead(500);
          res.end();
        }
      }
      return;
    }

    res.writeHead(404);
    res.end();
  });

  server.listen(PORT, "0.0.0.0", () => {
    console.log(`Listening on 0.0.0.0:${PORT}`);
  });
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

## handlers/start.js

```js
import { Composer, InlineKeyboard } from "grammy";

export const startRouter = new Composer();

startRouter.command("start", async (ctx) => {
  const kb = new InlineKeyboard()
    .text("Help", "help")
    .text("Ping", "ping");

  await ctx.reply(
    `Hi, <b>${ctx.from?.first_name || "stranger"}</b>.\nBot is alive on the stack.`,
    { parse_mode: "HTML", reply_markup: kb }
  );
});

startRouter.callbackQuery("help", async (ctx) => {
  await ctx.answerCallbackQuery();
  await ctx.editMessageText(
    "/start — greeting\n/help — this message\n/id — your Telegram ID"
  );
});

startRouter.callbackQuery("ping", async (ctx) => {
  await ctx.answerCallbackQuery({ text: "pong" });
  await ctx.reply("pong");
});
```

## handlers/common.js

```js
import { Composer } from "grammy";

export const commonRouter = new Composer();

commonRouter.command("id", async (ctx) => {
  await ctx.reply(`Your Telegram ID: <code>${ctx.from.id}</code>`, {
    parse_mode: "HTML",
  });
});

// Optional fallback
commonRouter.on("message", async (ctx) => {
  await ctx.reply("Unknown command. Try /help");
});
```

## Optional: services/db.js (Supabase)

```js
import { createClient } from "@supabase/supabase-js";

const url = process.env.SUPABASE_URL;
const key = process.env.SUPABASE_SERVICE_ROLE_KEY;

export const supabase =
  url && key ? createClient(url, key) : null;

export async function upsertUser(telegramId, username, firstName) {
  if (!supabase) return;
  await supabase.from("users").upsert({
    telegram_id: telegramId,
    username,
    first_name: firstName,
  });
}
```

Call from the `/start` handler if desired.

## Railway Variables (minimum)

```
BOT_TOKEN=...
MODE=webhook
WEBHOOK_SECRET=<openssl rand -hex 32>
# Railway injects PORT + RAILWAY_PUBLIC_DOMAIN
```

## Local testing

```bash
export BOT_TOKEN=...
export MODE=polling
npm start
```

## Notes specific to grammY on Railway

- `webhookCallback(bot, "http")` works cleanly with Node’s built-in `http` module — no Express required.
- Secret token is passed both to `setWebhook` and to `webhookCallback` for validation.
- Always re-register webhook on startup using the current `RAILWAY_PUBLIC_DOMAIN`.
- Keep `"type": "module"` in package.json if you use the ES module style above.
- Node 20+ recommended.
