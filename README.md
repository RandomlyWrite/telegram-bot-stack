# telegram-bot-stack

Grok Agent Skill for building, scaffolding, and deploying Telegram bots on the **GitHub + Railway + Supabase** stack.

## What this is

A complete operational playbook and production templates for:

- **aiogram 3** (Python) and **grammY** (Node) bots
- Webhook-first deploys on Railway with secret-token validation
- Supabase (or Railway Postgres) patterns
- Mini App / expanded stack guidance (React + Vite, Drizzle, Phaser/PixiJS, etc.)
- Full phased workflow from BotFather → production
- Reusable prompt you can paste into any new chat

## Install / use with Grok

1. Clone or copy this directory into your Grok skills folder:
   ```
   ~/.grok/skills/telegram-bot-stack/
   ```
2. The skill activates on phrases like “deploy my bot to Railway”, “webhook vs polling”, “aiogram Railway”, etc.
3. Or paste the contents of `assets/workflow-prompt.md` into a new conversation to drive a full scaffold.

## Structure

```
SKILL.md                 # Main instructions + workflow
references/
  aiogram3.md            # Full Python production skeleton
  grammy.md              # Full Node production skeleton
  ptb.md                 # python-telegram-bot notes
  railway.md             # Railway-specific env & deploy notes
  supabase.md            # Schema + RLS starters
  expanded-stack.md      # Mini App / game / AI agent architecture
assets/
  env.example
  workflow-prompt.md     # Copy-paste prompt for new bots
```

## Core rules enforced by the skill

- BOT_TOKEN and service_role keys never committed
- Webhook + `X-Telegram-Bot-Api-Secret-Token` validation in production
- Server is source of truth for scores / game outcomes
- Split bot worker and web service when complexity grows
- Prefer aiogram 3 / grammY over older libraries for new work

## License

MIT — do whatever you want with it. Attribution appreciated but not required.
