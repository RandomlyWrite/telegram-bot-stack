# Expanded Stack (Mini App + Game Backend)

Use this when the project is no longer “just a bot” and needs a real web surface, typed database, or server-authoritative realtime.

## Recommended defaults

| Layer | Choice | Why |
|-------|--------|-----|
| Frontend | **TypeScript + React + Vite** | Fast DX, first-class Telegram Mini App support, easy to ship static assets |
| UI primitives | Radix UI (when needed) | Accessible, unstyled, composable. Add only when you actually need complex components |
| 2D Mini App games | **Phaser** or **PixiJS** | Phaser for a fuller game framework (scenes, physics, input). PixiJS when you want lower-level custom rendering control |
| API | **Express** first, Fastify later | Express is the path of least resistance for most existing Node projects. Fastify when you want stricter schemas + higher throughput |
| ORM / schema | **Drizzle** | Lightweight, TypeScript-native, excellent migrations, works cleanly with Supabase or Railway Postgres |
| Database | PostgreSQL | Prefer Supabase-hosted Postgres for auth + realtime + storage in one place, or Railway Postgres for pure simplicity |
| Background jobs | Postgres first → **Redis + BullMQ** | Start with Postgres-backed jobs / `SKIP LOCKED` where practical. Introduce Redis + BullMQ (or similar) only when you need reliable retries, delayed jobs, or higher throughput |
| Hosting | **Railway** | Deploy bot worker and web service as **separate services**. Share the same Postgres. |
| Realtime | WebSockets (or Supabase Realtime) | Critical game state and outcomes are **always validated server-side** |

## Architecture rule

```
┌─────────────────┐     ┌──────────────────────┐
│  Telegram Bot   │     │  Web / Mini App API  │
│  (grammY or     │     │  (Express/Fastify)   │
│   aiogram)      │     │                      │
│  Railway svc A  │     │  Railway svc B       │
└────────┬────────┘     └──────────┬───────────┘
         │                         │
         └──────────┬──────────────┘
                    │
            ┌───────▼────────┐
            │  PostgreSQL    │
            │  (Supabase or  │
            │   Railway)     │
            └────────────────┘
```

- Bot and web service share the database.
- They do **not** share a process or deploy unit once you start scaling or iterating independently.
- Mini App talks only to the web service. The bot talks to Telegram and the database.

## Frontend notes (React + Vite)

- Telegram Mini Apps expect a standard SPA. Vite is ideal.
- Validate `window.Telegram.WebApp.initData` on the **server** (HMAC with bot token). Never trust the client-side user object alone for writes.
- Keep the Mini App thin: UI + optimistic updates. Authoritative state lives on the server.

## Database + Drizzle

- Define schema in TypeScript with Drizzle.
- Run migrations from CI or a one-off Railway job.
- Use the same Postgres URL in both bot and web services (service_role / server key only).
- For game scores, leaderboards, inventory, etc.: write from the server after validation. Client never gets to say “I scored 999999”.

## Realtime / game authority

- WebSockets (or Supabase Realtime channels) for live updates.
- **Server is the source of truth** for:
  - scores / outcomes
  - turn order
  - inventory / currency changes
  - anti-cheat checks
- Client can predict for responsiveness; server rejects invalid moves.

## When to split services on Railway

Split bot worker from web service when any of these become true:

- You want to deploy frontend/API changes without restarting the bot
- Bot traffic and API traffic have different scaling needs
- You need independent healthchecks / restarts
- Multiple people are working on bot vs Mini App

Until then, a single service is fine for early prototypes.

## Quick decision guide

| Situation | Stack |
|-----------|-------|
| Pure command bot, no UI | aiogram or grammY only (see other references) |
| Bot + simple Mini App | grammY/aiogram + React/Vite on Railway (or static host) |
| 2D game inside Mini App | Add Phaser (fuller framework) or PixiJS (custom rendering) |
| Full game / high interaction | Split services + Drizzle + server-authoritative WebSockets |
| Background / scheduled work | Start with Postgres jobs; add Redis + BullMQ when retries or volume demand it |
| Heavy auth / storage / realtime already needed | Lean on Supabase features; keep bot + API as thin clients |
