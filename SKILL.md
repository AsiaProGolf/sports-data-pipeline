---
name: sports-data-pipeline
description: "Reusable recipe for gathering a golf tournament's field, rankings, and player profiles for any event — the data backbone for golf media coverage, preview articles, and interview prep. Two paths: (A) a curl-pollable live-scoring JSON API when the event runs on the China Tour wegolf/iyoupin system (solves the official-start-list SPA problem), and (B) an Exa-research + optional headless-browser fallback, with a per-data-type source map, the OWGR/OoM/WAGR queries that work, a per-player output schema, and name-collision / stale-ranking gotchas. Use whenever the task is: pull a golf event field or start list, build a leaderboard scrape, rank a field by OWGR, get Order-of-Merit or WAGR standings, assemble player profiles, or set up a live-scoring/weather monitor. Triggers on golf event field, start list, tee times, rankings, OWGR, leaderboard scrape, tournament data pull, player data pull, player profiles, Order of Merit, WAGR, live scoring, sports data pipeline."
category: sports-data
catalog_summary: "Golf-event data pipeline: field + start list + OWGR/OoM/WAGR rankings + player profiles for any tournament. Preferred path = the wegolf/iyoupin live-scoring JSON API (curl-pollable, solves the start-list SPA problem); fallback = Exa web search + headless-browser scrape. Includes the per-player schema, the queries that work, and the name-collision/stale-ranking gotchas."
display_order: 50
---

# Sports Data Pipeline — Golf Event Field, Rankings & Profiles

The job: for any golf tournament, assemble a clean, ranked dataset of **who's in the field**, **how they rank** (OWGR, tour Order of Merit, WAGR for amateurs), and **a profile per player** — the raw material for tournament previews, leaderboards, and interview prep. The output is a per-player record sorted by world ranking, with every figure source-stamped and every unconfirmed entry flagged.

There are two ways to get the field. **Always check for Path A first** — it's dramatically cleaner when it exists.

## Path A (PREFERRED) — the wegolf / iyoupin live-scoring JSON API

When an event runs on the **China Tour "wegolf / iyoupin" scoring system** (e.g. China Tour events and their co-sanctions, like the ADT Bangkok Classic), the official start-list problem is **solved**: instead of an Angular SPA you can't scrape, there's a clean, public, **curl-pollable JSON API** that returns the full field, live scores, round schedule, and a weather/suspension notice. No headless browser needed.

How to tell you're on Path A: the public scoring URL contains `iyoupin` / `wegolf` / `scoringlive.iyoupin.top`, and carries a `?serid={shopId}_{eventId}` param. Those two IDs are the keys to everything.

Two endpoints do the work:

- **Event status / schedule / weather notice:**
  `https://scoringlive.iyoupin.top/api/wegolf/event/simple/profile?eventId={E}&shopId={S}`
  → `data.notice` (weather/suspension banner), `data.eventStatus`, `data.recentlyEventRoundId` (the live round), `data.rounds[]` (round IDs + names).
- **Full field + live scores:**
  `https://scoringlive.iyoupin.top/api/wegolf/event/leaderboard2?eventId={E}&eventRoundId={R}&boardType=1&shopId={S}&sort=true`
  → the complete player list with scores. `eventRoundId` comes from `recentlyEventRoundId` (or any id in `rounds[]`) from the profile call.

`eventId` and `shopId` come straight from the public URL's `?serid={shopId}_{eventId}`. Fetch with a normal `User-Agent` and `Accept: application/json` header.

**This gets you the FULL field (every name, not just the marquee 10–20) plus live scores — the single hardest artifact on Path B, for free.** Then jump to step 4 of the Path B procedure (rank + profile the players you just pulled).

**Polling / alerting:** to watch an in-progress event for weather, suspensions, round changes, or score swings, poll the profile endpoint on a cron and alert only on change. The pattern: GET the profile endpoint, hash `notice | status | round` into a small state file, and send an alert to a chat app / webhook only when the hash changes (first run sends a baseline "monitor live" ping). Reuse this structure for any wegolf/iyoupin event (swap `eventId`/`shopId`, run on cron, remove the cron entry after the event).

## Path B (FALLBACK) — Exa research + optional headless-browser scrape

For events NOT on the wegolf/iyoupin system, there is no free full-field API. You reconstruct the field from press + previews, rank each named player via Exa web search, and profile them. The detailed source map, the exact queries, the schema, and the gotchas live in **`references/recipe.md`** — read it before starting a Path B pull. The procedure in brief:

1. **Frame the event.** Search `"{Tour} {Event} {Year} {Venue} start list field"` + read the Wikipedia season page → dates, venue, purse, co-sanctions, field size, round status.
2. **Get the marquee field.** Fetch the most recent round-recap article(s) + the official pre-event preview; extract every named player and tag each `confirmed-press` / `confirmed-official`.
3. **(If the full list is needed)** Render the tour's live-scoring/entry-list SPA with a headless browser (e.g. Playwright `browser_navigate` + `browser_snapshot`) and scrape the table. Otherwise note the start-list gap explicitly — don't pretend the long tail is covered.
4. **Rank every named player.** One Exa query per player: `"{Name} Official World Golf Ranking OWGR profile"`. Record rank + best + as-of date. Search BOTH name orders for Asian names; disambiguate collisions by age/college/recent results.
5. **Add tour standings.** Pull the relevant Order-of-Merit table (`…/tic/tmoom.cgi`); grab Asian Tour OoM from press quotes.
6. **Amateurs.** For any `Am`, add WAGR (from wagr.com, or Wikipedia/college roster if wagr.com won't render) + a date; keep their OWGR too.
7. **Profile pass.** Wikipedia / Asian Tour / Data Golf / Minor League Golf for DOB, turned-pro year, notable wins, recent form.
8. **Assemble + sort.** Build the per-player schema, sort by OWGR ascending, label every ranking basis, stamp "as of" dates, flag every unconfirmed-field player.

## Output — per player

Sort by OWGR ascending; players with null/weak OWGR sort by tour OoM **with the basis labeled**. Never merge OWGR and tour-OoM into one number. Full schema and field definitions in `references/recipe.md` §3, but the spine is:

```
full_name, also_known_as, nationality, status (Pro|Am),
owgr_rank, owgr_best, owgr_as_of,
adt_oom_pos, asian_tour_oom_pos, wagr_rank, wagr_as_of,
birth_date|birth_year, age, turned_pro_year,
notable_wins[], recent_form,
in_field_confidence (confirmed-official | confirmed-press | notes-only | unconfirmed),
field_source
```

## The gotchas that bite (full detail in `references/recipe.md` §5)

- **Official full start lists are the hardest artifact on Path B** — OCS-Sport/OCS-Asia entry-list and live-scoring pages are Angular SPAs that return an empty shell to search/fetch. Path A sidesteps this entirely; on Path B, budget a headless-browser render or accept a marquee-only field.
- **Stale rankings** — OWGR updates weekly (Mondays), WAGR roughly weekly. Always stamp "as of {date}", re-pull on event week, and cite the source per figure (different sources snapshot different weeks and disagree by a few spots).
- **Name collisions** — common CJK-romanized names (e.g. "Peng Bo" / "Bo Peng") can match two real players. Disambiguate by age/college/recent results before trusting an OWGR number.
- **Romanization** — Asian and Thai names have many spellings; press uses one, OWGR another. Search variants and match on birth year / tour history, not string equality.
- **Amateurs carry both WAGR and OWGR** (OWGR includes amateurs) — capture both. A college amateur's "in the field" claim is the LEAST reliable; confirm amateurs against an official source specifically.

## Reference

`references/recipe.md` — the full per-data-type source map (what worked, per data type), the exact Exa queries, the complete output schema with field notes, the expanded gotchas, the Path A API field reference, and the step-by-step procedure. Read it for any real pull; this SKILL.md is the map, the recipe file is the territory.
