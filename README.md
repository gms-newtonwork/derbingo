# Derbingo

Team bingo game for the Derbyshire trip. A single static `index.html` — no
build step, no framework — with a shared team board backed by Supabase, and
`me` (your display name) kept in your own browser's `localStorage`.

This is a faithful port of the original Claude-artifact version: same 25
board activities, scoring, fine presets, house tiers, hills banner, board
editor and suggestions/voting. The only change is the storage layer, which
now talks to a real database instead of Claude's built-in `window.storage`,
so the board loads fast and reliably for everyone with the link.

## 1. Create a free Supabase project

1. Go to [supabase.com](https://supabase.com) and sign up (free tier is enough).
2. Create a new project. Pick any name/region; note the database password
   somewhere safe (you won't need it for this app).
3. Wait for the project to finish provisioning (a minute or two).

## 2. Create the data table

In the Supabase dashboard, open **SQL Editor** → **New query**, paste the
following, and run it:

```sql
create table if not exists kv (
  key text primary key,
  value jsonb,
  updated_at timestamptz default now()
);

alter table kv enable row level security;

create policy "Allow anonymous read" on kv
  for select
  to anon
  using (true);

create policy "Allow anonymous insert" on kv
  for insert
  to anon
  with check (true);

create policy "Allow anonymous update" on kv
  for update
  to anon
  using (true)
  with check (true);

-- Enables realtime sync (live ticks/fines/suggestions without a refresh)
alter publication supabase_realtime add table kv;
```

## 3. Get your project URL and anon key

In the Supabase dashboard: **Project Settings → API**. Copy:

- **Project URL** (e.g. `https://xxxxxxxx.supabase.co`)
- **anon public** key (a long JWT-looking string)

## 4. Paste them into `index.html`

Open `index.html` and edit the two constants right at the top of the file:

```js
window.SUPABASE_URL = "YOUR_SUPABASE_URL";
window.SUPABASE_ANON_KEY = "YOUR_SUPABASE_ANON_KEY";
```

## 5. Deploy

### GitHub Pages

This repo includes `.github/workflows/pages.yml`, which publishes
`index.html` on every push to `main`.

1. Push your edited `index.html` to the `main` branch of this repo.
2. In the repo's **Settings → Pages**, set **Source** to **GitHub Actions**.
3. The workflow runs automatically and gives you a `https://<org>.github.io/derbingo/` URL.

### Netlify or Vercel (same idea)

Both work the same way for a single static file:

- **Netlify**: "Add new site" → drag-and-drop the repo folder (or connect the
  git repo) → no build command, publish directory `/`.
- **Vercel**: import the repo → framework preset "Other" → no build command,
  output directory `/`.

Either will give you a URL immediately; share that link with the team.

## Acceptance checklist

- [ ] Board loads within a second or two of opening the page.
- [ ] Ticking a square in one browser shows up in a second browser shortly after.
- [ ] Reloading the page keeps all state.
- [ ] Editing the board saves for everyone and keeps existing ticks.
- [ ] Fines update the leaderboard and house tiers; suggestions and upvotes persist.
- [ ] No console errors; the app never hangs on "Loading…" (if the keys are
  wrong or Supabase is unreachable, you'll see a clear banner instead, and the
  board still renders with local-only defaults).

## Security trade-off — please read

The Supabase anon key is meant to be public (it's embedded in the page
source, same as it is now), and that's fine. What's *not* fine by default is
the database policy above: it allows **anyone with the link** to read *and
write* every row in the `kv` table, with no authentication.

For a low-stakes internal team game, this is an acceptable trade-off — it
keeps the app simple with no login step. But be aware:

- Anyone who has the URL can see the whole board, all names, and all fines.
- Anyone can also overwrite the board, add fake fines, or delete/replace
  data, whether by using the app normally or by calling the Supabase API
  directly with the anon key.
- There's no way to prove who actually ticked a square — it's on the honour
  system, same as the original.

Don't put anything in the board, fines, or suggestions that you wouldn't
want a stranger with the link to see or change. If you ever want tighter
control, the next step up is adding Supabase email/magic-link auth and
rewriting the RLS policies to check `auth.uid()` instead of allowing `anon`.
