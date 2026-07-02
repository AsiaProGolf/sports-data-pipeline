# Sports Data Pipeline — Golf Event Field, Rankings & Profiles

A Claude Code skill and detailed reference for assembling structured golf tournament data: who's in the field, how each player ranks (OWGR, tour Order of Merit, WAGR for amateurs), and a profile per player.

Built from a live production pull on the ADT Bangkok Classic 2026 (Asian Development Tour / China Men's Professional Golf Tour co-sanction). Covers any event regardless of scoring platform.

## What It Does

For any golf event, this pipeline produces a per-player dataset sorted by world ranking, with every figure source-stamped and unconfirmed entries flagged. The output is the raw material for tournament previews, leaderboards, and interview preparation.

Two paths handle different events:

**Path A — wegolf/iyoupin API (preferred)**
When an event runs on the China Tour "wegolf / iyoupin" live-scoring system, a clean, public, unauthenticated JSON API returns the full field, live scores, round schedule, and weather/suspension notice with a standard HTTP request — no headless browser needed. This solves the start-list SPA problem entirely for supported events.

**Path B — Exa research + headless browser (fallback)**
For events not on wegolf/iyoupin, the pipeline uses Exa web search to reconstruct the field from press coverage and official previews, with an optional headless browser step to scrape complete entry lists from Angular-based tour portals.

## Contents

| File | Description |
|---|---|
| `SKILL.md` | Claude Code skill definition — triggers, summary, and the workflow in brief |
| `references/recipe.md` | Full reference: source map, exact queries, output schema, gotchas, step-by-step procedure |

## Installation

Copy `SKILL.md` (and optionally `references/recipe.md`) into your Claude Code skills directory. The skill activates automatically when a task matches any of its trigger phrases.

## Requirements

- **Claude Code** with skill loading enabled
- **Exa** web search (for Path B research and OWGR lookups)
- **Playwright or equivalent headless browser** (optional — Path B only, for full entry-list extraction from Angular SPAs)

## Usage

Once installed, the skill triggers on tasks matching:

> golf event field, start list, tee times, rankings, OWGR, leaderboard scrape, tournament data pull, player data pull, player profiles, Order of Merit, WAGR, live scoring, sports data pipeline

The skill walks you through:
1. Detecting which path applies (check for the wegolf/iyoupin API first)
2. Pulling the full field and live scores (Path A) or reconstructing from press + browser (Path B)
3. Ranking every player by OWGR, adding tour Order of Merit and WAGR for amateurs
4. Building the per-player profile pass
5. Assembling the sorted, source-stamped output dataset

## Output Schema — Per Player

```
full_name, also_known_as, nationality, status (Pro|Am),
owgr_rank, owgr_best, owgr_as_of,
adt_oom_pos, asian_tour_oom_pos, wagr_rank, wagr_as_of,
birth_date|birth_year, age, turned_pro_year,
notable_wins[], recent_form,
in_field_confidence (confirmed-official | confirmed-press | notes-only | unconfirmed),
field_source
```

Sorted by OWGR ascending. Rankings and standings are never merged into one number — each figure carries its basis and an "as-of" date.

## Key Gotchas

- **Official full start lists (Path B) are Angular SPAs** on most tour portals — search/fetch tools get an empty shell. Path A (wegolf/iyoupin) sidesteps this entirely; on Path B, budget a headless-browser render if you need all names.
- **Rankings go stale** — OWGR updates weekly (Mondays), WAGR roughly weekly. Always stamp "as of {date}" and re-pull on event week.
- **Name collisions** — common CJK-romanized names can match two different real players. Disambiguate by age, college, or recent results before trusting an OWGR number.
- **Romanization variants** — Asian and Thai names have many spellings. Search both name orders and match on birth year / tour history.

See `references/recipe.md` for the full source map, proven Exa queries, and expanded gotcha detail.

## License

MIT — see [LICENSE](LICENSE).
