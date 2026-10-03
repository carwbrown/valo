# VALO — Valorant Rank-Up Routine Tracker

A single-user static web app for tracking progress through a 30-day Valorant
improvement routine. Pick a day (1–30), work three tabs — **Warm Up**, **In Game**,
**Training** — and mark it complete. Progress persists in the browser via IndexedDB.

Modeled closely on the `two-five-fifteen` project (single-file static site, program
switcher, GitHub-hosted). Look there for the sibling pattern.

## How Claude should work on this project

- **Show code before writing.** Explain the change and show the diff/snippet; wait for
  explicit approval before editing `index.html`. (Follows the root `github/CLAUDE.md`.)
- **Don't commit or push without explicit approval.**
- **Preview before declaring done.** `index.html` uses IndexedDB and JS, which do **not**
  run from a `file://` snapshot. Serve it over HTTP to verify
  (`python3 -m http.server` in the repo dir, then open `http://localhost:PORT`), and
  test both programs + the day dropdown.
- **Keep it simple.** No build step, no framework, no dependencies, no router. One file.
- **No Riot IP.** The theme is an original Valorant-*inspired* look (palette, condensed
  type, angular HUD corners). Never embed Riot/Valorant art, agent likenesses, logos, or
  copyrighted images. Text content of the routines is the user's own material.

## Stack & deploy

- **Pure static site:** one `index.html` with inline `<style>` and `<script>`. External
  assets are CDN-only: fonts from Google Fonts (Barlow + Bebas Neue) and
  `@supabase/supabase-js@2` from jsDelivr. No bundler, no local deps.
- **Host:** GitHub repo `carwbrown/valo`, deployed on **Netlify**.
- **Netlify build settings** (static, no build):
  - Branch to deploy: `main`
  - Base directory: *(blank)*
  - Build command: *(blank)*
  - Publish directory: `.`
  - Functions directory: *(blank — no functions)*
- To update the live site: edit `index.html` → commit to `main` → Netlify auto-deploys.

## Architecture & technical decisions

- **Two programs, one engine.** The app ships two 30-day plans selected by the header
  switcher, mirroring two-five-fifteen's 2515/SSC pattern:
  - **Slayerkey** — "SkyHigh Routine," transcribed word-for-word from the published plan.
  - **Mesos** — the user's own consistency-first plan, built from their improvement doc +
    if-then plans.
- **Program config objects.** Each program is a plain object (`SLAYERKEY`, `MESOS` in
  `PROGRAMS`) providing: `id`, `subtitle`, `weeks`, `days`, and three panel-builder
  functions `warmup(day,info,w)`, `ingame(...)`, `training(...)` that return HTML strings.
  Adding a new program = add one object; the picker, detail view, dropdown, and storage
  all key off the active `PROG`. **This is the main extension seam.**
- **HTML built from data, not hand-written sections.** The 30 days live in a `days` map
  (`{t, a, date?, bench?}`); the detail view renders from the active program's builders.
  Shared building blocks: `C/P/UL/L/I/TAG` helpers and `refSection()` (the Rule of 2 /
  LEAD / 2SS reference, used by both programs).
- **Tab mapping** (how the source plans map onto the 3 tabs):
  - **Warm Up** — the program's warm-up drills (Slayerkey: 2SS drill set; Mesos: 7-drill
    ~19-min routine + fight-by-range).
  - **In Game** — the day's goal/comp focus *verbatim*, plus always-on reference cards
    (Rule of 2, LEAD, 2SS) and program extras (Mesos: standing rules, if-then, intention).
  - **Training** — study / mechanics. Mesos "Benchmark Sunday" days (`bench:true`) render
    the full Voltaic benchmark flow instead of maintenance.
- **Theming via CSS vars + a `--accent` swap.** `:root` holds the Valorant palette
  (`--red #ff4655`, `--teal #19e0cf`, `--cream #ece8e1`, navy surfaces). `--accent` is red
  for Slayerkey; `body.mesos` flips it to teal. Angular HUD corners are a reusable
  `--clip` / `--clip-sm` `clip-path`. Headings: Bebas Neue; body: Barlow (DIN-like).
- **Searchable day switcher.** The detail view has a `<datalist>`-backed input
  (`#daySearch`) — filter-as-you-type, no dependency. `handleJump()` matches full label,
  then bare day number, then title substring.

## Data model & sync (Supabase cloud + IndexedDB cache)

**Supabase is the source of truth; IndexedDB is an offline cache.** This gives
cross-device sync (PC ↔ phone) and durable backup, with instant local loads.

- **Supabase** project `ktbszqcjyvpljrzlldrg` (`val-improve`). Table `public.valo_state`:
  `user_id uuid` + `id text` (composite PK), `data jsonb`, `updated_at timestamptz`.
  **RLS is per-user** — every policy is `auth.uid() = user_id`, so a signed-in user can only
  read/write their own rows. Client is vanilla `@supabase/supabase-js@2` from CDN; the
  **publishable key** + URL are hardcoded in `index.html` (safe — publishable keys are meant
  for client code, and RLS is what protects the data; no build step to inject env vars).
- **Auth: magic-link email login** (`signInWithOtp`). A full-screen `#authGate` overlay
  blocks the app until there's a session; `onAuthStateChange` + `getSession` drive
  `enterApp()`. Session persists (`persistSession:true`), so it's one sign-in per device.
  `cloudGet`/`cloudPut` tag every row with `USER.id`. No `USER` → cloud calls no-op (local
  cache still works). Redirect URLs must be allow-listed in Supabase Auth config per
  environment (localhost + the prod Netlify URL).
- **Records** (same `id`s in both Supabase and the IndexedDB `valo`/`progress` store; in
  Supabase they're additionally scoped by `user_id`):
  - `settings` → `{ id, data:{ rank, goal, agent, start } }` (shared across programs)
  - `prog:slayerkey` / `prog:mesos` → `{ id, data:{ completed:[dayNumbers] } }` (per-program)
- **Flow:** load renders from IndexedDB instantly, then `pullSettings`/`pullProgram`
  reconcile from Supabase (**cloud wins**) and re-render. Saves dual-write: IndexedDB +
  `cloudPut` upsert. A header sync chip shows `Saving… / ● Synced / ○ Offline`. Offline,
  it degrades to the local cache and re-syncs on next successful save.
- **Storage seam:** everything goes through the `DB` helper + `cloudGet/cloudPut` +
  `load*Cache`/`save*`/`pull*` functions — the only place storage is touched.
- **Security:** data is protected by per-user RLS, so the publishable key being public (in
  the repo / on the site) is fine — nobody can read or write a user's rows without being
  signed in as that user. Each person's progress is isolated.
- **Email caveat:** uses Supabase's built-in email sender, which is rate-limited (a few/hour,
  testing-grade). For reliable magic links, configure custom SMTP in Supabase Auth settings.
- **Schema + auth config were applied via the Management API** (`supabase` CLI token →
  `POST /v1/projects/{ref}/database/query` for DDL; `PATCH /v1/projects/{ref}/config/auth`
  for the redirect allow-list), not the dashboard. SQL lives in scratch `valo_auth.sql`.

## Adding to the routine

- **New day content:** edit the program's `days` map and (if needed) its panel builders.
- **New program:** add a config object to `PROGRAMS`, add a header `prog-btn`, done.
- **New reference card shared by both:** extend `refSection()`.
