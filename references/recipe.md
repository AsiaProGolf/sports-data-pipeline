# Sports Data Pipeline — Full Recipe (golf event field, rankings, profiles)

The detailed reference behind `SKILL.md`: a workflow for assembling golf-event fields, rankings, and player profiles. Read this for any actual pull; the SKILL.md is the summary.

## Table of contents
1. The two paths — when to use which
2. Path A — the wegolf / iyoupin live-scoring JSON API (preferred)
3. Path B — source map (what worked, per data type)
4. Path B — Exa queries that actually surfaced the right pages
5. Output schema (per player)
6. Gotchas
7. Step-by-step procedure for event X
8. Monitoring / alerting an in-progress event

---

## 1. The two paths — when to use which

Getting the FIELD is the crux. Everything downstream (ranking, profiling) is the same regardless. Two ways to get the field:

- **Path A — live-scoring JSON API.** Available when the event runs on the China Tour **wegolf / iyoupin** scoring system. Returns the FULL field + live scores + round schedule + weather notice as clean JSON over plain `curl`. This is the single best outcome — it solves the start-list SPA problem. Always check for it first.
- **Path B — Exa research + optional headless-browser scrape.** The general fallback for events not on wegolf/iyoupin. Reconstruct the marquee field from press + preview; render the tour's SPA with a browser only if you need the long tail.

**How to detect Path A:** the public scoring URL contains `iyoupin` / `wegolf` / `scoringlive.iyoupin.top` and carries `?serid={shopId}_{eventId}`. If you see that, you're on Path A — use §2 and skip the field-reconstruction parts of Path B.

---

## 2. Path A — the wegolf / iyoupin live-scoring JSON API (PREFERRED)

The China Tour live-scoring stack (`scoringlive.iyoupin.top`, "wegolf") exposes a clean, public, unauthenticated JSON API. Unlike the OCS-Sport / OCS-Asia Angular SPAs (which return an empty shell to any fetch tool), these endpoints return real data to a normal HTTP GET. This is the discovery that makes the otherwise-hardest artifact — the official full start list — trivial.

### 2.1 The IDs

Everything is keyed on two integers from the public scoring URL:

```
?serid={shopId}_{eventId}
```

`eventRoundId` is a third id you obtain from the profile call (below) — one per round.

### 2.2 Endpoint 1 — event status / schedule / weather notice

```
https://scoringlive.iyoupin.top/api/wegolf/event/simple/profile?eventId={E}&shopId={S}
```

Returns (under `data`):
- `data.notice` — the weather / suspension / schedule banner (the thing to watch during play).
- `data.eventStatus` — numeric event status.
- `data.recentlyEventRoundId` — the id of the round currently live (feed this to endpoint 2 as `eventRoundId`).
- `data.rounds[]` — array of rounds, each with `eventRoundId` and `eventRoundName` (so you can map a round id to "Round 3", etc.).

### 2.3 Endpoint 2 — full field + live scores

```
https://scoringlive.iyoupin.top/api/wegolf/event/leaderboard2?eventId={E}&eventRoundId={R}&boardType=1&shopId={S}&sort=true
```

- `{R}` = `recentlyEventRoundId` from endpoint 1 (or any id from `rounds[]` to pull a specific round).
- `boardType=1`, `sort=true` give the sorted leaderboard.
- Returns the COMPLETE player list — every name, not just the marquee — with scores. This is the full start list + live leaderboard in one call.

### 2.4 Request mechanics

Send a browser-ish `User-Agent` and `Accept: application/json`. No auth. Example (Python; replace the placeholders with IDs from the event URL):

```python
import json, urllib.request
event_id = "EVENT_ID"
shop_id = "SHOP_ID"
API = f"https://scoringlive.iyoupin.top/api/wegolf/event/simple/profile?eventId={event_id}&shopId={shop_id}"
req = urllib.request.Request(API, headers={"Accept": "application/json", "User-Agent": "Mozilla/5.0"})
data = json.load(urllib.request.urlopen(req, timeout=30)).get("data", {})
```

### 2.5 After Path A

You now have the full field + live scores. Hand off to **§7 step 4 onward** — rank each player by OWGR, add tour OoM / WAGR, and run the profile pass. The field-reconstruction steps (press recaps, preview, browser scrape) are unnecessary on Path A.

---

## 3. Path B — source map (what worked, per data type)

For events NOT on wegolf/iyoupin. Best source and access pattern per data type, with whether the Exa search tool can extract it cleanly:

| Data type | Best source(s) | URL pattern / access | Extractable via Exa? |
|---|---|---|---|
| Event basics (dates, venue, purse, co-sanction, field size) | **Wikipedia season page** + **local sports press** | `en.wikipedia.org/wiki/{Year}_{Tour}`; Bangkok Post, Philstar, Sina golf | Yes — clean |
| Field confirmation / who's playing | **Round-1 recap articles** (name the leaders + notable names) + **official pre-event preview** | Philstar / Bangkok Post / Sina match reports; tour press release | Partial — leaders + marquee names only, NOT the full N |
| Official FULL start list / tee times | **Tour TIC / live-scoring** (OCS-Sport / OCS-Asia) | `wp-adt.ocs-sport.com/report?tourn={CODE}&season={YYYY}&report={rep}`; `adt.ocs-asia.com/tic/tmscores.cgi?tourn={CODE}`; `asiantour.com/adt/leaderboard` | **NO** — Angular/JS app, empty shell. Needs a headless browser to render + scrape. |
| OWGR | **owgr.com player profile** | `owgr.com/playerprofile/{slug}-{id}`. Search `"{Name} Official World Golf Ranking OWGR profile"` → Exa returns the profile with current rank in the highlight | Yes — rank, best rank, avg/total points all in the snippet |
| Tour Order of Merit (ADT etc.) | **OCS-Asia TIC OoM table** | `adt.ocs-asia.com/tic/tmoom.cgi?oom=AD~…~season={YYYY}~…` — this ONE OCS table DID return as plain text via Exa | Yes (the OoM table specifically; the live-scoring pages did NOT) |
| Asian Tour ranking | `asiantour.com/atoom?id={YYYY}&oom=Y` + player profiles | JS-rendered profile pages (empty), but OoM position is usually quoted in press | Partial — get from press quotes |
| WAGR (amateurs) | **wagr.com profile** (often empty via fetch) → **Wikipedia / Grokipedia / college roster / APGC articles** for the number | `wagr.com/playerprofile/{slug}-{id}` (JS, empty); fall back to text sources that cite the WAGR figure + date | Partial — wagr.com itself didn't render; secondary sources gave the number |

**Key takeaway:** OWGR and the OoM TIC table extract cleanly via Exa; the live-scoring / entry-list SPAs do not. That's why a full Path-B start list needs a browser — and why Path A is worth checking for first.

---

## 4. Path B — Exa queries that actually surfaced the right pages

Exa is semantic — write each query as a description of the ideal page, not keywords.

- **Event + field:** `"{Tour} {Event} {Year} {Venue} start list field tee times"` and `"{Event} {Year} first round report full scores players news"`. The R1 (or latest-round) recap is the highest-yield single page for "who's in + who's hot."
- **OWGR (one per player):** `"{Full Name} Official World Golf Ranking OWGR profile"` → top hit is the owgr.com profile; the rank is right in the highlight, so often no fetch needed.
- **Tour OoM:** `"Asian Development Tour Order of Merit {Year} standings leaders ranking"` (adapt tour name) → surfaced the OCS TIC OoM table as plain text.
- **WAGR / amateur:** `"{Name} World Amateur Golf Ranking WAGR {Year}"` + `"{Name} current WAGR position {Year} ranked"` → Wikipedia / Grokipedia / college roster carry the figure with a date.
- **Romanization tip:** search BOTH name orders for Asian names (`"{Given} {Family}"` AND `"{Family} {Given}"`), plus hyphen and spacing variants. OWGR slugs are unpredictable on order.

---

## 5. Output schema (per player)

```
full_name, also_known_as (nickname / alt romanization),
nationality, status (Pro|Am),
owgr_rank (int|null), owgr_best (int|null), owgr_as_of (date),
adt_oom_pos (int|null), asian_tour_oom_pos (int|null),
wagr_rank (int|null, amateurs only), wagr_as_of (date),
birth_date|birth_year, age, turned_pro_year,
notable_wins[], recent_form (this season),
in_field_confidence (confirmed-official | confirmed-press | notes-only | unconfirmed),
field_source (which article / preview confirmed it)
```

**Sort:** OWGR ascending (lower = better). Players with null/weak OWGR sort by tour OoM **with the basis labeled per row**. NEVER merge OWGR and tour-OoM into one number — they are different rankings on different populations.

**Confidence ladder for `in_field_confidence`:**
- `confirmed-official` — named in the official entry list or tour preview/press release.
- `confirmed-press` — named in an independent round recap or match report.
- `notes-only` — from internal/prior notes, not re-verified against a source for THIS field.
- `unconfirmed` — plausible but no source confirms entry to this specific event (treat as not-in-field until verified; especially for amateurs).

---

## 6. Gotchas

- **Official full start lists are the hardest artifact (Path B).** OCS-Sport / OCS-Asia live-scoring + entry-list pages are Angular SPAs — search/fetch tools get an empty shell. Budget a headless-browser render (Playwright `browser_navigate` + `browser_snapshot`, or another browser automation tool) if you need all N names with scores. Press recaps + the official preview cover the marquee 10–20; the long tail (club pros, qualifiers) usually isn't recoverable without the browser scrape. **Path A (wegolf/iyoupin) eliminates this gotcha entirely — always check for it first.**
- **Stale-ranking risk.** OWGR updates weekly (Mondays); WAGR roughly weekly. Always stamp "as of {date}" and re-pull on event week. Different sources disagree by a few spots (e.g. rank 707 in one source vs 733 in another) because they snapshot different weeks — cite the source per figure.
- **Name collisions.** A reversed two-part name matched two different players with distinct tour histories. Disambiguate by age / college / recent results before trusting an OWGR number. Same vigilance for any common CJK-romanized name.
- **Thai (and broad Asian) romanization.** Many spellings; the press uses one, OWGR another. Search variants and match on birth year / tour history, not string equality.
- **Amateur vs pro.** Amateurs (status `Am`) carry BOTH a WAGR and an OWGR (OWGR includes amateurs). Capture both. A college amateur's "in the field" claim is the LEAST reliable — college seasons + invitational status mean they're often not actually entered. Confirm amateurs against an official source specifically.
- **The OoM TIC table is the OCS exception** that DOES render to text — useful, but its URL carried a `season=2023` param while returning 2026 data, so don't trust the URL params; trust the table contents + cross-check the leader against a recent article.

---

## 7. Step-by-step procedure for event X

1. **Frame the event.** Search `"{Tour} {Event} {Year} {Venue} start list field"` + read the Wikipedia season page → lock dates, venue, purse, co-sanctions, field size, round status.
2. **Check for Path A.** Find the event's public scoring URL. If it's on `scoringlive.iyoupin.top` / wegolf / iyoupin (look for `?serid={shopId}_{eventId}`), use §2 to pull the full field + live scores as JSON, then jump to step 4. Otherwise continue on Path B.
3. **(Path B) Get the marquee field.** Fetch the most recent round-recap article(s) + the official pre-event preview. Extract every named player + their confirmation source. Tag each `confirmed-press` / `confirmed-official`. **(If the full list is needed)** render the tour's live-scoring/entry-list SPA with a headless browser and scrape the table; otherwise note the start-list gap explicitly.
4. **Rank every named player.** One Exa query per player: `"{Name} OWGR profile"`. Record rank + best + as-of date. Search both name orders for Asian names; disambiguate collisions by age / college.
5. **Add tour standings.** Pull the relevant OoM table (`…/tic/tmoom.cgi`) for OoM positions; grab Asian Tour OoM from press quotes.
6. **Amateurs.** For any `Am`, add WAGR from wagr.com (or Wikipedia / college roster if wagr.com won't render) + a date; keep their OWGR too.
7. **Profile pass.** Wikipedia / Asian Tour / Data Golf / Minor League Golf for DOB, turned-pro year, notable wins, recent form.
8. **Assemble + sort.** Build the per-player schema, sort by OWGR ascending, label every ranking basis, stamp "as of" dates, and flag every unconfirmed-field player.

---

## 8. Monitoring / alerting an in-progress event (Path A)

To watch a live event for weather holds, suspensions, round changes, or score swings, poll the wegolf/iyoupin profile endpoint (§2.2) on a cron and alert only on change.

**The reusable pattern:**
- GETs `…/event/simple/profile?eventId=…&shopId=…`, reads `data.notice`, `data.eventStatus`, `data.recentlyEventRoundId`, and maps the round id via `data.rounds[]`.
- Hashes `notice | status | round`; stores the hash in a state file; sends an alert to a chat app / webhook only when the hash changes. First run sends a baseline "monitor live" ping.
- Sets `PATH` for cron's minimal environment, exits quietly on transient network errors, and advances state even if the send fails so it doesn't spam on retry.

To reuse: implement the pattern, swap `eventId` / `shopId` for the new event, point it at the right chat/webhook, add a cron entry, and remove the cron entry after the event. For score-change alerting (not just the notice banner), poll the `leaderboard2` endpoint (§2.3) instead and hash the leader / top-N.
