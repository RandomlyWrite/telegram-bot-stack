# Supabase Patterns for Telegram Bots

## Minimal Schema Starters

```sql
-- users
create table public.users (
  telegram_id bigint primary key,
  username text,
  first_name text,
  last_name text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- example game / score table
create table public.scores (
  id bigint generated always as identity primary key,
  telegram_id bigint references public.users(telegram_id) on delete cascade,
  score integer not null,
  game_id text not null,
  created_at timestamptz default now()
);

create index on public.scores (telegram_id, game_id);
create index on public.scores (game_id, score desc);
```

## RLS Guidance

- Enable RLS on every table.
- Bot process uses `service_role` key → bypasses RLS (convenient and correct for server-side writes).
- Mini App frontend uses `anon` key + RLS policies that check `auth.uid()` or a custom claim if you implement Telegram auth mapping.
- Never put service_role in browser or Mini App code.

Example policy for client-readable scores (adjust to your auth model):

```sql
alter table public.scores enable row level security;

create policy "Public read scores"
  on public.scores for select
  using (true);  -- or more restrictive
```

## Client Initialization

**Python (bot side):**

```python
import os
from supabase import create_client, Client

url: str = os.environ["SUPABASE_URL"]
key: str = os.environ["SUPABASE_SERVICE_ROLE_KEY"]
supabase: Client = create_client(url, key)
```

**Node (bot side):**

```js
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_ROLE_KEY
)
```

## Common Operations

Upsert user on /start:

```python
supabase.table("users").upsert({
  "telegram_id": user.id,
  "username": user.username,
  "first_name": user.first_name,
  "last_name": user.last_name,
}).execute()
```

Insert score:

```python
supabase.table("scores").insert({
  "telegram_id": user_id,
  "score": points,
  "game_id": "farkle",
}).execute()
```

Leaderboard:

```python
res = supabase.table("scores")\
  .select("telegram_id, score, users(username)")\
  .eq("game_id", "farkle")\
  .order("score", desc=True)\
  .limit(10)\
  .execute()
```

## Realtime

Subscribe from the bot or from a separate admin service if you need live updates. For most degen bots, periodic polling of leaderboards or simple inserts are enough.

## Auth Mapping (advanced)

If you want full Supabase Auth tied to Telegram:

1. Validate Telegram `initData` on your Railway backend.
2. Create or link a Supabase user (custom claim or external id).
3. Issue a short-lived JWT the Mini App can use.

Most simple bots skip this and just use `telegram_id` as the primary key.

## Security Rules

- service_role lives only in Railway Variables.
- Validate every Telegram update origin (secret token or origin check).
- Rate-limit expensive Supabase writes if the bot is public.
- Prefer server-side score calculation; never trust client-reported high scores.
