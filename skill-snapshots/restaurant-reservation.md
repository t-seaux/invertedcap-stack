---
name: restaurant-reservation
description: |-
  Find a restaurant table for Tom (or Elsie) and hand him a booking link — VISIBILITY-ONLY (Resy banned the automation account 2026-09-03; nothing auto-books). Surface real open slots across Resy + OpenTable + SevenRooms, give Tom the link to tap; once he confirms he booked, add it to the household calendar. Trigger on "book a table", "make a reservation", "reserve [restaurant]", "get us a table at [X]", "dinner reservation for [N] on [date]", "find a table near [neighborhood]", "is there a table at [X]", "check availability at [X]", or the sms-listener routing a reservation text here. ALSO recommend mode: "recommend a spot", "where should we eat", "anything new we should try", "pick somewhere for Friday" — live editorial sources + the regulars ledger + real availability so every rec is actually open. Defaults: party of 2, dinner, "in the neighborhood" = Brooklyn Heights / Cobble Hill / Boerum Hill. NONE of the three platforms can be booked programmatically — always report open times + a link; never claim a table is booked. NOT the haircut skill (Meevo/reCAPTCHA-walled).

---

# Restaurant Reservations (visibility-only)

Find a real, open table and hand Tom the link — he taps to book. Same *find → present*
front half as before; the *book* half is gone.

> ⛔ **AUTO-BOOK REMOVED 2026-09-03 — Resy banned the automation account.** Resy Platform
> Security terminated the automation account (`thomas.seo@outlook.com`) for a ToS violation
> ("fair reservation process"), deactivated it, and canceled its future reservations — they
> fingerprinted the scripted booking/sniping pattern. **This skill no longer books anything
> on any platform.** `resy.mjs details/book/cancel` now hard-refuse, the date-night
> watcher/sniper are disarmed, and the auth token is retired. **Do NOT** re-capture a Resy
> token or repoint the automation at a fresh account — a new account gets re-flagged (the
> behavioral fingerprint is known) and could put Tom's real personal booking at risk. See
> [[project_appointments]] + [[feedback_restaurant_snipe_policy]].

The **read** half is fully intact and needs no auth: `search`/`find`/`nearby-open` (Resy) and
`availability.mjs check`/`scarcity` (Resy + OpenTable + SevenRooms) all still surface real
open slots with just the public web key. That's the whole job now: **surface real times →
hand Tom the venue link → let him book manually in the app.**

## Platforms — all three are read-only now

Restaurants allocate inventory **per platform**, so a Resy-only view is a partial picture.
Real case: **Gage & Tollner** (Downtown Bklyn) is OpenTable-only and returns "not on Resy."

| Platform | Read | Book | How to read |
|---|---|---|---|
| **Resy** | ✅ | ❌ (banned 2026-09-03) | Public web api_key — `search`/`find`, no user auth |
| **SevenRooms** | ✅ | ❌ | Public widget JSON, no auth — needs a venue slug |
| **OpenTable** | ✅ | ❌ | **Akamai-walled**: curl AND headless Chrome are both blocked; only the real non-headless CDP browser loads it |

For **every** platform: surface the times and hand Tom the link — he taps to book.
**Never claim a table is booked.** Booking under a clean personal Resy account is Tom's job.

## Tooling — `~/.claude/local-agents/agent-browser/`

No secrets needed — every command below is unauthenticated read.

```
# resy.mjs — Resy READ (search + availability only; book/details/cancel are DISABLED)
node resy.mjs search "<name>" [lat] [long]        # → [{name,id,neighborhood,match,name_score}]
node resy.mjs nearby "<query>"                    # → venues near HOME, preferred hoods first
node resy.mjs nearby-open <day> <party> [when] ["<q>"]  # → nearby venues that ACTUALLY have tables
node resy.mjs find <venue_id> <day> <party> [when]      # → {venue, window, slots:[{time,type,token}]}
# node resy.mjs details|book|cancel …             # ⛔ DISABLED 2026-09-03 (account banned) — they hard-refuse

# availability.mjs — cross-platform VISIBILITY
node availability.mjs check "<venue>" <day> <party> [when]   # → merged Resy + OT + SR open times
node availability.mjs scarcity "<venue>"                     # → always-booked fingerprint (below)
node availability.mjs opentable <slug|url> <day> <party> [when]
node availability.mjs sevenrooms <slug> <day> <party> [when]
```

- Everything here is auth-free read. There is no booking command anymore.
- **`check` fires ALL THREE platforms in parallel, every time** (Tom's standing directive).
  Unmapped venues get slug **auto-discovery** folded into each leg — SR probes slug guesses
  against its own API (cheap; its 400-vs-200 response is the validity oracle), OT guesses
  slugs and **name-gates on the page `<title>`** (browser-priced). Hit or miss, the result
  is cached in `venue_map.json` (`false` = confirmed absent — the negative cache is what
  keeps repeat checks at ~0.5s). Delete a venue's entry to force re-discovery.
- **SR-discovered slugs are name-UNVERIFIED** (SR has no name endpoint — its pages are a JS
  shell), so a same-named venue elsewhere could collide. `check` flags this in the note;
  confirm the venue once with Tom before treating its SR times as authoritative.
- **To hand off a Resy link:** build `https://resy.com/cities/new-york-ny/venues/<slug>` from
  the `slug` on the `search`/`find` hit (or just link the venue's Resy page); include the date
  + party so Tom lands on the right day.

### `when` (time) semantics — shared by every command

| arg | means |
|---|---|
| *(omitted, `nearby-open`/`check`)* | **dinner** 17:30–21:30 — a bare "book us a table" is dinner, not 11am brunch |
| *(omitted, `find`)* | the whole day (one named venue → full picture is small + useful) |
| `19:00` | **around 7** — ±45 min, ranked closest-first |
| `18:00-21:00` | explicit range |
| `dinner` / `lunch` / `any` | named windows |

An exact-match filter would be wrong: a venue holding 7:15 but not 7:00 must still surface.

## Defaults (Tom's standing prefs)

- **Party size = 2** unless he names a number ("table for 4") or clearly implies one. Never
  ask "how many?" on a bare "book us a table" — assume 2 and **state it in the handoff** so a
  wrong assumption is correctable.
- **"in the neighborhood" / "nearby" / no location given = Brooklyn Heights, Cobble Hill,
  Boerum Hill** (home = 25 Garden Pl, the Garden & State corner). Use `nearby-open`, which centers
  on home — Garden Pl & State St (40.6924,-73.996) — and **ranks those three first** while still showing adjacent
  spots (Carroll Gardens, Downtown Bklyn) rather than hard-cutting them. Anywhere else he'll
  name — then use `search`/`find`.
- Dinner is the default meal (see the window table).
- **Date nights run EARLY: 6:15/6:30 is ideal, 7:00/7:30 gladly taken.** For a date-night
  lookup use window `18:00-19:30` (a range ranks earliest-first, which matches the pref)
  and lead with the early slots. Don't surface 8:30+ for a date night unless nothing earlier
  exists — and say so when that's the case.

## ⚠️ Venue-name gate — reservation search is FUZZY and always returns something

Searching a venue that **isn't on the platform** returns a different restaurant as hit #1,
silently. Real example: Resy `search "Dos Caminos"` → **"Casino"** (Lower East Side).
Quoting hits[0] on faith points Tom at the wrong restaurant. So `search` returns
`match`/`name_score`:

- **`exact`** (≥0.85) → proceed.
- **`close`** (0.6–0.85) → **do NOT auto-pick. Confirm first.** Real trap: `"Frankies 457"`
  scores 0.7 against **"Lil' Frankie's"** — a different restaurant in the East Village.
- **`weak`** (<0.6) → treat as **not on that platform**. Say so, then run `check` (it may be
  on OpenTable) or offer the closest listed alternatives / the venue's phone number.

Use the `neighborhood` field to disambiguate when names collide — it's usually the tell.

## ⚠️ OpenTable auto-discovery LIES — cross-check every OT hit against Resy

`availability.mjs check`'s OpenTable leg (slug auto-discovery) produces **false-positive
availability** and must NOT be trusted on its own. Two failure modes, both seen live
2026-09-18 (West Village sell-dinner search):

1. **Wrong-venue slug match.** `"Kingfisher"` → `king-new-york` — that's **King**, a
   different restaurant; the OT `<title>` name-gate did NOT catch it. An OT "hit" can be a
   *different venue's* inventory entirely.
2. **Uniform/default slot grid.** Several unrelated venues (King, Semma, Via Carota,
   Wallflower) all returned the **identical** grid `6:30, 6:45, 7:00, 7:15, 7:30` for the
   same date/party. When you see the same five-slot grid repeat across different venues,
   it's synthetic, not real inventory.

**Rule: Resy is the source of truth. Treat any OT-only slot as UNVERIFIED until a Resy `find`
on an EXACT-match venue corroborates it.** When Resy shows *none in window* but OT shows a
full grid → believe Resy (every one of the four above was actually booked or a wrong-venue
match). Only trust an OT hit when (a) the slug is unmistakably the right venue AND (b) the
grid isn't the uniform default. Otherwise hand Tom the OT *link* to check himself — never
assert the slot exists. This is the "never fabricate availability" guardrail in practice:
the earlier "King is wide open" answer that had to be retracted came from trusting this leg.

**Confirm the Resy `find` plumbing is alive before trusting a batch of empties.** If a sweep
returns "none" across many venues, sanity-check by re-running ONE at `any`/full-day (or
party-of-2): real late slots coming back (e.g. King → 8:30pm+) proves the tool works and the
window is genuinely booked — distinguishes "truly full" from "tool silently erroring."

**Via Carota is effectively unbookable on Resy** — returns truly-none even party-of-2
full-day; it holds tables for walk-ins. Don't waste a check on it; note it as walk-in.

**Resy fuzzy wrong-venue matches seen this run** (reinforces the name-gate): Semma→**Gemma**
(Bowery, `close`), Wallflower→**Wildflower** (Chelsea, `weak`), Don Angie→**Don Don**
(Midtown, `weak`). Always read `match` + `neighborhood`; skip `close`/`weak` unless confirmed.
**Not on Resy at all** this run: Kingfisher, Fairfax (OT-only, 14-day rolling window),
Graciela — for these, Resy `search` returns a wrong venue or nothing, so the ONLY honest
path is the platform's own link, not an asserted slot.

## Flow

Two shapes of request:

- **"somewhere in the neighborhood"** → `nearby-open <day> <party> [when]` (one call, checks
  ~12 venues in parallel, ~0.5s). Surface the best 2–3 with times + links. (Resy-only — OT/SR
  have no cheap geo search; that's fine for a sweep, the named-venue path covers the rest.)
- **A NAMED venue** ("look for a reservation at X", "is there a table at X", "book X") →
  **`availability.mjs check "<name>" <day> <party> [when]` — all three platforms in
  parallel, always.** Never conclude "no tables" from Resy alone.

Then:

1. **Convert dates.** "Friday"/"tomorrow" → a real `YYYY-MM-DD` (today is injected in
   context; NYC time). **Party size defaults to 2** (see Defaults) — state it in the handoff,
   don't ask.
   - No slots anywhere → say so plainly and offer a different date/time, or a similar venue
     (`nearby-open` by cuisine). **Never fabricate availability.** And when a dead end has a
     *reason* (booking window not open yet, sold out, notify-me), surface the reason — see the
     "No tables" section below.
   - `check`'s resy leg returns an `uncertain` object instead of slots on a `close` name
     match — confirm the venue with Tom before quoting (see the name gate).
2. **Name cancellation terms when they matter.** `check`/SR output carries a `cancellation`
   note when present — include it in the handoff so Tom knows the policy before he taps.
3. **Hand off — report real times + a link, never imply it's held:**

   ```
   🍽️ Found a table — tap to book

   Venue: Lilia
   Party: 2
   Open: 7:00 PM, 7:15 PM — Fri Sep 12
   Platform: Resy
   Cancel: free up to 24h before        ← only if there's a policy worth naming
   Link: https://resy.com/cities/new-york-ny/venues/lilia?date=2026-09-12&seats=2
   ```

   No bold (iMessage renders Unicode bold in a fallback font — Tom rejected it; see
   [[project_sms_command_path]]). Plain text, emoji header, blank line, then fields.
   Same shape for OpenTable/SevenRooms — just the link goes to their page. **Never say
   "booked."** Tom books manually under a clean personal account.
4. **Log it once Tom confirms he booked.** When Tom says he tapped it through, append one
   line to `reservation_log.jsonl` in this skill's dir:
   `{"ts":"<now ISO>","venue":"…","day":"YYYY-MM-DD","time":"HH:MM","party":N,"platform":"resy|opentable|sevenrooms","status":"booked","requested_by":"tom|elsie","source":"named|recommendation|nearby"}`
   Also append `"status":"cancelled"` events when a booking is called off. This ledger is what
   powers the **regulars** axis of recommend mode — no log, no regulars.
5. **Add to calendar — only after Tom confirms he booked, and ROUTE BY EVENT NATURE.** Hand
   off to `add-to-calendar`, which picks the calendar: **personal/date/family dinners →
   household (Elsie-Tom); work dinners (recruiting, founder, investor, portfolio, biz) →
   Tom's Inverted work calendar.** Don't default everything to household — that was wrong
   (Tom 2026-09-18: a founding-engineer recruiting dinner is a WORK event; I mis-filed it on
   the household cal and had to delete it). Title `Dinner — <Venue> (<party>)`, start = slot
   time, 1.5h default, location = venue, Busy. Dedup first.
   **For a work dinner with other attendees, Tom usually creates + sends the invite himself**
   (external invitees) — so for a clearly-work dinner, ASK before auto-adding rather than
   silently creating a parallel event he'll duplicate. The platform also emails a booking
   confirmation regardless. NB: the claude.ai Calendar MCP can't delete — cleanup goes
   through `~/.claude/scripts/calendar_write/calendar_write.py delete` ([[reference_calendar_write]]).

## Recommend mode ("recommend a spot", "where should we eat", "anything new to try?")

Three axes. Pick by what Tom asked; blend when he's open-ended. Every recommendation must be
**actionable** — cross the shortlist with availability (`nearby-open` for Resy names is
~free; `check` for specific picks) so he gets *"X has 7:30 Friday, here's the link"*, not just
a list.

**Axis 1 — High-rated.** Editorial sources, fetched live (plain curl + the standard UA
works; pages are 0.4-2.5MB, pull names from `<h2>` tags):
- Eater NY 38 (the essentials): `https://ny.eater.com/maps/best-new-york-restaurants-38-map`
- The Infatuation NYC hub (reviews carry /10 ratings): `https://www.theinfatuation.com/new-york`
  — neighborhood + cuisine guides linked off it.

**Axis 2 — New / should-try.** The "what just opened and is good" lists:
- Eater NYC Heatmap: `https://ny.eater.com/maps/best-new-nyc-restaurants-heatmap`
  (find borough variants off `https://ny.eater.com/maps`)
- Infatuation Brooklyn Hit List: `https://www.theinfatuation.com/new-york/guides/best-new-brooklyn-restaurants-hit-list`
  (+ `best-new-new-york-restaurants-hit-list` for the city-wide one)
- **Grub Street** (NY Mag): `https://www.grubstreet.com/` — sharpest openings coverage;
  the "Absolute Best" franchise lives under it too (both axes).

Source landscape (settled with Tom 2026-09-01): **Time Out = skip** (listicle-heavy, rarely
adds over the above). **NYT Top 100** (annual) + **Michelin** matter for axis 1 but block
plain fetch (NYT 403s, Michelin bot-challenges) — pull their contents via web search when
relevant; don't wire. New Yorker Tables for Two: too low-volume, skip.

All wired URLs verified fetchable + parseable 2026-09-01. **Guides rot** — if one 404s, find
the successor from the site's index rather than guessing slugs. Cite which list a rec came
from ("on Eater's Heatmap this month"). Default geography = the home cluster (see Defaults)
unless Tom names an area.

**Axis 2a — NEIGHBORHOOD-NEW (Tom's priority axis).** He especially wants to *learn about*
new places near home — even before they're easy to get into (Frenzie, Zig Zag Oyster Bar are
the archetypes). Method, proven 2026-09-01:
1. Fetch the Infatuation BK Hit List (and Eater Heatmap/Brooklyn map). The Hit List embeds
   **JSON-LD** with street address + lat/long per venue — parse `{"@type":"Restaurant"`
   windows, don't string-match neighborhoods.
2. Distance-filter from home (Garden Pl & State St, `40.6924,-73.996`): **≤ ~3.5km** counts —
   Tom's news radius is brownstone-Brooklyn-wide (Zig Zag at 3.3km in Prospect Heights
   counts), wider than the strict booking-default three.
3. Track lifecycle in `agent-browser/neighborhood_new_seen.json` — it's a **TO-TRY list,
   not a suppression list**. A spot stays new until `status: visited` (they went —
   cross-check `reservation_log.jsonl`) or `dismissed` (Tom said no thanks) or it ages out
   (~6mo and off the lists). "Anything new?" = fresh finds **plus reminders**, labeled:
   *"new this time: X"* vs *"still on the to-try list: Frenzie, Zig Zag"*. Being mentioned
   before is never a reason to omit — Tom wants the reminder until they've been.
4. For each new spot: `check` it + `scarcity` when it matters. New+hot spots are usually
   always-booked (Frenzie and Zig Zag both: ZERO open tables across 45 days) — so the right
   surfacing is *"Frenzie opened on the Heights — it books out instantly; I'll keep an eye on
   it and flag the moment a Saturday opens so you can grab it."* Note the honest limit: we can
   *watch and alert*, but Tom does the actual booking (see Watching below).

**Axis 3 — Regulars.** `reservation_log.jsonl` (this dir) is the memory: every confirmed
booking appends there (see Flow step 4). Read it to know (a) the rotation — places booked
repeatedly; (b) "haven't done X in a while"; (c) what's already booked this week (don't
recommend Friday's spot for Saturday). Blend: a regular he loves + one new-list place to
try is a better answer than three of either. The log starts near-empty — until it has
history, say so and lean on axes 1–2 (optionally ask Tom to seed a few favorites).

## Scarcity classes — "is this an always-booked place?"

`availability.mjs scarcity "<venue>"` takes one snapshot and classifies what getting a
prime (6–9pm) table actually takes. Resy venues → full 45-day scan (~10s, read-only);
OT venues → sparse probe of 3 Saturdays + 2 weekdays (~40s, browser-priced). Cached snapshots
in `agent-browser/scarcity.json` (dated — re-snapshot ~monthly or when a check contradicts
the class).

| class | meaning | what to do |
|---|---|---|
| `book-on-demand` | prime tables sitting open (Cafe Spaghetti) | link Tom the day he asks |
| `plan-ahead` | prime exists but thin (Lilia midweek) | tell him to grab it days ahead, not day-of |
| `snipe-required` | prime inventory never sits open (Don Angie — 0 tables across 3 Sats + 2 weekdays) | the drop/cancellations are the only door → offer to watch + alert |

Check `saturday_class` too — Lilia is `plan-ahead` overall but **0 of 4 Saturdays** had
prime slots: Saturday-specific `snipe-required`.

**Use it reactively:** when Tom names a venue and `check` comes back empty for his date,
run `scarcity` before answering. "No tables Friday" is a weak answer; *"Don Angie never has
open tables — it's a book-at-the-drop place; want me to watch it and ping you the moment a
Saturday opens?"* is the right one. Venues that moved platforms surface here too (Don Angie's
Resy row is inactive; it lives on OT now).

## Watching drop-based / always-booked venues (alert-only)

For venues that never sit open (`snipe-required`), the mechanism is now **watch + alert, Tom
books**. We can poll availability (Resy `find` / OT / SR reads are all auth-free) and ping Tom
the moment a target slot appears — but the actual booking is his tap, on his own account.

- **Do NOT** re-arm the old auto-booking `date_night_watch.mjs` / `sniper.mjs snipe` path —
  those depended on the banned account and are disarmed (their `book` calls hard-refuse).
- A useful watch = a light poll (e.g. via `schedule`) on a target venue+date+window that,
  on the first in-window slot, alerts Tom (family thread / Slack) with the link. Speed still
  matters for a hot Saturday, so alert immediately and let Tom tap fast.
- Cadence still follows [[feedback_restaurant_snipe_policy]] (~monthly date night, 4-week
  floor) when picking which Saturday to chase — but as an alerting cadence, not an auto-book.

## Guardrails

- **This skill never books.** No platform can be booked programmatically anymore — always
  report times + a link and say plainly that Tom taps to book. Never say "booked."
- **Never surface a slot `find`/`check` didn't return.** No hallucinated times or venues.
- **Elsie** may request (she's on the SMS allowlist); Tom does the actual booking. Apply the
  standard Elsie fence for everything else ([[project_sms_command_path]]).
- **Cancellation:** tell the requester to cancel from the Resy/OpenTable app or the
  confirmation email — we can't cancel via API.
- **SR slug collisions are real and the TZ gate is the defense.** The slug `popina` is a
  London venue, not Cobble Hill's Popina — caught because SR slots carry `time_iso` +
  `utc_datetime`, and NYC must be UTC-4/-5. Both discovery and every read enforce this.
  (SR parsing is verified against live venues as of 2026-09-01: mixed 12h/24h time
  formats, request-only filtering, seating areas, per-slot cancellation policies.)
- **Surface SR cancellation policies** — the adapter returns `cancellation` when present
  (e.g. Cafe Spaghetti charges $25/head for late cancels). Name it in the handoff message.
- Keep replies short and conversational on the SMS path; minimize round trips.

## ⚠️ "No tables" always needs a WHY, not just an empty slot list

Empty slots can mean genuinely sold out, but on some OpenTable/Resy venues it means the
booking window simply hasn't opened yet — e.g. **Fairfax (West Village)** only opens each
date exactly **14 days out at 11:00 AM EST** (its own reservation widget states this
explicitly: "Reservations for next 14 days open on `<date>` at 11:00 AM EST"). Reporting
"zero open tables" without that context is a materially incomplete answer — Tom flagged
this live 2026-09-02 ("you should've told me why"). Before reporting a dead end:
- If the platform page/API response includes an explanation (rolling window, notify-me,
  "sold out", "not accepting online reservations"), surface it verbatim/paraphrased.
- If the venue has a known fixed booking horizon, set a reminder for the exact window-open
  moment so Tom can grab it himself (`eventkit add` only takes a date, not a time — put the
  exact time in the title/notes), or offer the alert-only watch above.

## Notes

- Read endpoints proven live 2026-09-01, still valid: `search` (venuesearch) + `find`
  (/4/find) return real venues + open slots using the public web key.
- **Booking is gone (2026-09-03):** Resy terminated the automation account for a ToS
  violation. `resy.mjs details/book/cancel` hard-refuse; the launchd jobs
  (`com.invertedcap.date-night-watch`, `com.invertedcap.resy-token-refresh`) are unloaded and
  their plists + the dead token/`.resy_config` live in
  `~/.claude/_disabled-resy-automation-2026-09-03/`. Do not resurrect. See
  [[project_appointments]] + [[feedback_restaurant_snipe_policy]].
