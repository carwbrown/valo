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

- **Pure static site:** one `index.html` with inline `<style>` and `<script>`. Fonts from
  Google Fonts (Barlow + Bebas Neue). No other external assets.
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

## Data model (IndexedDB)

- DB `valo`, object store `progress` (keyPath `id`). Records:
  - `settings` → `{ id:"settings", setup:{ rank, goal, agent, start } }` (shared across programs)
  - `prog:slayerkey` / `prog:mesos` → `{ id, completed:[dayNumbers] }` (per-program progress)
- All reads/writes go through the `DB` helper and the `load*/save*` functions — the only
  place storage is touched. Swapping to a backend later means changing only those.

## Adding to the routine

- **New day content:** edit the program's `days` map and (if needed) its panel builders.
- **New program:** add a config object to `PROGRAMS`, add a header `prog-btn`, done.
- **New reference card shared by both:** extend `refSection()`.
