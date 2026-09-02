---
name: restaurant-reservation
description: >
  Book a restaurant reservation on Resy for Tom (or Elsie) via the scriptable Resy API —
  no browser needed for the booking. Find a venue, surface real open slots, and place the
  reservation ONLY on an explicit YES (same quote→YES→book gate as purchase-agent), then
  add it to the household calendar. Trigger on "book a table", "make a reservation",
  "reserve [restaurant]", "get us a table at [X]", "dinner reservation for [N] on [date]",
  "find a table near [neighborhood]", "is there a table at [X]", "check availability at [X]",
  or the sms-listener routing a reservation text here. ALSO recommend mode: "recommend a
  spot", "where should we eat", "anything new we should try", "pick somewhere for Friday" —
  blends live editorial sources (Eater NY 38 + Heatmap, Infatuation Hit Lists), the
  reservation_log.jsonl regulars ledger, and real availability so every rec is bookable.
  Defaults: party of 2, dinner, and "in the neighborhood" = Brooklyn Heights / Cobble Hill /
  Boerum Hill. Named-venue lookups fire ALL THREE platforms in parallel (availability.mjs
  check, with slug auto-discovery); Resy is the only BOOKABLE one — OpenTable/SevenRooms are
  visibility-only (report open times + a link, Tom taps); never claim a non-Resy table is
  booked. NOT the haircut skill (that's Meevo/reCAPTCHA-walled). Never books silently.
---

# Restaurant Reservations (Resy)

Book a table on Resy. Unlike the haircut flow (Meevo, reCAPTCHA-walled → pre-stage-and-tap),
Resy's REST API is plain token auth with **no reCAPTCHA wall**, so the whole
find → confirm → book flow runs headlessly. Same confirm-gate philosophy as
[[project_purchase_agent]]: **surface options → explicit YES → book → add to calendar.**

## The hard gate (never skip)

**A reservation is placed ONLY after the requester replies YES to a specific quoted slot.**
A reservation is an outward-facing commitment — many venues carry cancellation windows or
no-show fees. So:

- Never book without an explicit YES naming (or clearly pointing at) one slot.
- Never invent a slot — only ever offer times that came back from `find`.
- If the requester is vague ("dinner Friday"), quote the closest real slots and let them pick.
- Booking is per-account (Tom's Resy). Elsie may request; the reservation is under Tom's name.

## Platforms — one bookable, two visibility-only

Restaurants allocate inventory **per platform**, so a Resy-only view is a partial picture.
Real case: **Gage & Tollner** (Downtown Bklyn) is OpenTable-only and returns "not on Resy."

| Platform | Read | Book | How |
|---|---|---|---|
| **Resy** | ✅ | ✅ | Clean token API, no bot wall |
| **SevenRooms** | ✅ | ❌ | Public widget JSON, no auth — needs a venue slug |
| **OpenTable** | ✅ | ❌ | **Akamai-walled**: curl AND headless Chrome are both blocked; only the real non-headless CDP browser loads it |

For OpenTable/SevenRooms: surface the times and hand Tom the link — he taps to book.
**Never claim a non-Resy table is booked.**

## Tooling — `~/.claude/local-agents/agent-browser/`

`RESY_AUTH_TOKEN` is injected as env by the daemon/processor (from `.resy-auth-token.enc`).

```
# resy.mjs — Resy (bookable)
node resy.mjs search "<name>" [lat] [long]        # → [{name,id,neighborhood,match,name_score}]
node resy.mjs nearby "<query>"                    # → venues near HOME, preferred hoods first
node resy.mjs nearby-open <day> <party> [when] ["<q>"]  # → nearby venues that ACTUALLY have tables
node resy.mjs find <venue_id> <day> <party> [when]      # → {venue, window, slots:[{time,type,token}]}
node resy.mjs details <config_token> <day> <party>      # → {book_token, cancellation, payment_methods}
node resy.mjs book <config_token> <day> <party>         # → {booked, resy_token, reservation_id}

# availability.mjs — cross-platform VISIBILITY
node availability.mjs check "<venue>" <day> <party> [when]   # → merged Resy + OT + SR
node availability.mjs scarcity "<venue>"                     # → always-booked fingerprint (below)
node availability.mjs opentable <slug|url> <day> <party> [when]
node availability.mjs sevenrooms <slug> <day> <party> [when]
```

- Everything except `details`/`book` needs no auth.
- `book` re-fetches a fresh `book_token` internally (they expire fast) — pass it the
  `config_token` from `find`/`nearby-open`/`check`'s resy leg, not a book_token.
- `book` uses `RESY_PAYMENT_METHOD_ID` if set (see `.resy_config`), else the account default.
  Resy requires a card on file even for free reservations.
- **`check` fires ALL THREE platforms in parallel, every time** (Tom's standing directive).
  Unmapped venues get slug **auto-discovery** folded into each leg — SR probes slug guesses
  against its own API (cheap; its 400-vs-200 response is the validity oracle), OT guesses
  slugs and **name-gates on the page `<title>`** (browser-priced). Hit or miss, the result
  is cached in `venue_map.json` (`false` = confirmed absent — the negative cache is what
  keeps repeat checks at ~0.5s). Delete a venue's entry to force re-discovery.
- **SR-discovered slugs are name-UNVERIFIED** (SR has no name endpoint — its pages are a JS
  shell), so a same-named venue elsewhere could collide. `check` flags this in the note;
  confirm the venue once with Tom before treating its SR times as authoritative.

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
  ask "how many?" on a bare "book us a table" — assume 2 and **state it in the quote** so a
  wrong assumption is correctable before the YES.
- **"in the neighborhood" / "nearby" / no location given = Brooklyn Heights, Cobble Hill,
  Boerum Hill** (home = 25 Garden Pl, the Garden & State corner). Use `nearby-open`, which centers
  on home — Garden Pl & State St (40.6924,-73.996) — and **ranks those three first** while still showing adjacent
  spots (Carroll Gardens, Downtown Bklyn) rather than hard-cutting them. Anywhere else he'll
  name — then use `search`/`find`.
- Dinner is the default meal (see the window table).
- **Date nights run EARLY: 6:15/6:30 is ideal, 7:00/7:30 gladly taken.** For a date-night
  booking use window `18:00-19:30` (a range ranks earliest-first, which matches the pref)
  and when quoting options, lead with the early slots. Don't offer 8:30+ for a date night
  unless nothing earlier exists — and say so when that's the case.

## ⚠️ Venue-name gate — reservation search is FUZZY and always returns something

Searching a venue that **isn't on the platform** returns a different restaurant as hit #1,
silently. Real example: Resy `search "Dos Caminos"` → **"Casino"** (Lower East Side).
Quoting hits[0] on faith books the wrong restaurant. So `search` returns `match`/`name_score`:

- **`exact`** (≥0.85) → proceed.
- **`close`** (0.6–0.85) → **do NOT auto-pick. Confirm first.** Real trap: `"Frankies 457"`
  scores 0.7 against **"Lil' Frankie's"** — a different restaurant in the East Village.
- **`weak`** (<0.6) → treat as **not on that platform**. Say so, then run `check` (it may be
  on OpenTable) or offer the closest listed alternatives / the venue's phone number.

Use the `neighborhood` field to disambiguate when names collide — it's usually the tell.

## Flow

Two shapes of request:

- **"somewhere in the neighborhood"** → `nearby-open <day> <party> [when]` (one call, checks
  ~12 venues in parallel, ~0.5s). Quote the best 2–3 with times. (Resy-only — OT/SR have no
  cheap geo search; that's fine for a sweep, the named-venue path covers the rest.)
- **A NAMED venue** ("look for a reservation at X", "is there a table at X", "book X") →
  **`availability.mjs check "<name>" <day> <party> [when]` — all three platforms in
  parallel, always.** Never conclude "no tables" from Resy alone. The resy leg applies the
  name gate itself and returns bookable config tokens, so on a Resy hit go straight to
  quote → YES → `book`; on an OT/SR-only hit use the visibility handoff below.
Then:

1. **Convert dates.** "Friday"/"tomorrow" → a real `YYYY-MM-DD` (today is injected in
   context; NYC time). **Party size defaults to 2** (see Defaults) — state it in the quote,
   don't ask.
   - No slots anywhere → say so plainly and offer a different date/time, or a similar venue
     (`nearby-open` by cuisine). **Never fabricate availability.**
   - `check`'s resy leg returns an `uncertain` object instead of slots on a `close` name
     match — confirm the venue with Tom before quoting (see the name gate).
2. **Check cancellation terms** before quoting a commit-heavy booking: `details <token> <day>
   <party>` surfaces any `cancellation` fee/window. If there's a fee, name it in the quote.
3. **Quote + gate.** Present the pick and wait for YES:

   ```
   🍽️ Reservation — reply YES to book

   Venue: Lilia
   Party: 2
   When: Fri Sep 12, 7:00 PM
   Seating: Dining Room
   Cancel: free up to 24h before        ← only if there's a policy worth naming
   ```

   No bold (iMessage renders Unicode bold in a fallback font — Tom rejected it; see
   [[project_sms_command_path]]). Plain text, emoji header, blank line, then fields.
4. **Book on YES.** `book <config_token> <day> <party>`. Confirm success with the
   `reservation_id`. If `book` errors (slot gone, token expired), re-run `find`, tell the
   requester it got snapped up, and offer the next open slot — never silently retry a
   different time.
   - **Log every confirmed booking** (Resy bookings AND OT/SR handoffs once Tom confirms he
     tapped) — append one line to `reservation_log.jsonl` in this skill's dir:
     `{"ts":"<now ISO>","venue":"…","day":"YYYY-MM-DD","time":"HH:MM","party":N,"platform":"resy|opentable|sevenrooms","status":"booked","requested_by":"tom|elsie","source":"named|recommendation|nearby"}`
     Also append `"status":"cancelled"` events when a booking is called off. This ledger is
     what powers the **regulars** axis of recommend mode — no log, no regulars.
5. **Add to calendar.** On a confirmed booking, add the event to the **household (personal)
   calendar** via the `add-to-calendar` skill / the sms-listener calendar fast-path:
   title `Dinner — <Venue> (<party>)`, start = slot time, 1.5h default, location = venue.
   Dedup first (don't double-add). Resy also emails a confirmation; the calendar event is ours.

### Visibility-only path (OpenTable / SevenRooms)

When the table is on a platform we can't book, **report and hand off** — never imply it's held:

```
🍽️ Found a table — but you'll need to tap

Venue: Gage & Tollner (Downtown Bklyn)
Party: 2
Open: 9:00 PM, 9:15 PM — Fri Sep 12
Platform: OpenTable (I can see it but can't book it)
Link: <url from the check output>
```

Only offer to add the calendar event **after Tom confirms he booked it**.
OpenTable checks take ~10s and may briefly open a Chrome window — that's the Akamai wall,
and it's the only way to see OT inventory at all.

## Recommend mode ("recommend a spot", "where should we eat", "anything new to try?")

Three axes. Pick by what Tom asked; blend when he's open-ended. Every recommendation must be
**actionable** — cross the shortlist with availability (`nearby-open` for Resy names is
~free; `check` for specific picks) so he gets *"X has 7:30 Friday"*, not just a list.

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
new places near home — even before they're bookable (Frenzie, Zig Zag Oyster Bar are the
archetypes). Method, proven 2026-09-01:
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
4. For each new spot: Resy-check + scarcity when it matters. New+hot spots are usually
   `snipe-required` (Frenzie and Zig Zag both: ZERO open tables across 45 days) — so the
   right surfacing is *"Frenzie opened on the Heights — never has open tables; want it on
   the Saturday watchlist?"* Learning about it and getting in are one motion.

**Axis 3 — Regulars.** `reservation_log.jsonl` (this dir) is the memory: every confirmed
booking appends there (see Flow step 4). Read it to know (a) the rotation — places booked
repeatedly; (b) "haven't done X in a while"; (c) what's already booked this week (don't
recommend Friday's spot for Saturday). Blend: a regular he loves + one new-list place to
try is a better answer than three of either. The log starts empty 2026-09-01 — until it has
history, say so and lean on axes 1–2 (optionally ask Tom to seed a few favorites).

## Scarcity classes — "is this an always-booked place?"

`availability.mjs scarcity "<venue>"` takes one snapshot and classifies what getting a
prime (6–9pm) table actually takes. Resy venues → full 45-day scan (~10s); OT venues →
sparse probe of 3 Saturdays + 2 weekdays (~40s, browser-priced). Cached snapshots in
`agent-browser/scarcity.json` (dated — re-snapshot ~monthly or when a booking attempt
contradicts the class).

| class | meaning | what to do |
|---|---|---|
| `book-on-demand` | prime tables sitting open (Cafe Spaghetti) | just book when asked |
| `plan-ahead` | prime exists but thin (Lilia midweek) | book days ahead, never day-of |
| `snipe-required` | prime inventory never sits open (Don Angie — 0 tables across 3 Sats + 2 weekdays) | the drop is the only door → offer the watchlist |

Check `saturday_class` too — Lilia is `plan-ahead` overall but **0 of 4 Saturdays** had
prime slots: Saturday-specific `snipe-required`.

**Use it reactively:** when Tom names a venue and `check` comes back empty for his date,
run `scarcity` before answering. "No tables Friday" is a weak answer; *"Don Angie never has
open tables — it's a book-at-the-drop place; want me to watch it for a Saturday?"* is the
right one. Venues that moved platforms surface here too (Don Angie's Resy row is inactive;
it lives on OT now). NOTE: OT venues can't be snipe-BOOKED (no book API) — an OT "watch"
means ping Tom the moment inventory appears, link in hand.

## Date-night sniper (drop-based venues) — designed, pending activation

Hot venues release inventory a fixed number of days out (often at a 9:00/10:00am ET drop);
prime Saturday slots go in minutes. This is the one flow where booking runs WITHOUT a
mid-drop YES — because the YES happens **before** the drop:

1. **Watchlist** — `agent-browser/watchlist.json`: per-venue `release_days` (measure with
   `sniper.mjs horizon <venue_id>` — scans 45 days of `/4/find`; re-verify occasionally,
   hot venues undercount), `drop_time_et` (null = snipe at both 9:00 and 10:00 ET until
   observed), and the standing target (dow/window/party). Lilia seeded (~27d horizon,
   verified 2026-09-01: only 2 of 18 open days were Saturdays — sniping is genuinely needed).
2. **Pre-authorization ping** — the evening BEFORE a watched venue's drop covers a target
   Saturday (drop_date = target_saturday − release_days), ping the FAMILY GROUP THREAD
   (sms-listener's `.family_group_id`): *"Lilia drops Sat the 28th tomorrow ~9am — want me
   to grab 6:30–8:30 for 2? YES/NO."* No YES recorded → no snipe. This ping is the consent
   gate; it replaces the usual quote-then-YES because a mid-drop confirm loses the race.
3. **Snipe at drop** — `sniper.mjs snipe <venue_id> <day> <party> <window> [maxMin]`:
   polls every ~3s, books the first in-window slot immediately (closest time first, real
   table preferred over Counter/Outdoor — verified). `--dry-run` tests without booking.
   Needs `RESY_AUTH_TOKEN`. Then: confirm in the thread, log it, calendar it — the usual
   post-book steps.
4. **Miss** → report honestly in the thread ("didn't win a slot, next drop is tomorrow for
   Sunday") and optionally offer the `check`/nearby fallback for that night.

**Activation is pending two things:** the Resy auth token (same One-time setup below), and
Tom picking the watchlist venues + confirming the ping cadence — then wire the scheduled
job (drop-eve ping + drop-time snipe) into the scheduled-runtime/launchd stack. The Resy
`/4/venue/calendar` endpoint 500s on the public key — retest it authed; if it works it
replaces the 45-day scan for horizon detection.

## Guardrails

- **Never book without an explicit YES to a specific slot.** No auto-book at any point.
- **Never surface a slot `find` didn't return.** No hallucinated times or venues.
- **Elsie** may request (she's on the SMS allowlist); the reservation is under Tom's Resy
  account and name. Apply the standard Elsie fence for everything else ([[project_sms_command_path]]).
- **Cancellation:** if the requester later says "cancel that", this skill does not yet cancel
  via API — tell them to cancel from the Resy app/email, or offer to look it up. (Future work.)
- **Only Resy can be booked.** OpenTable/SevenRooms are read-only: report times + link and
  say plainly that Tom has to tap. Never say "booked" for a non-Resy venue.
- **SR slug collisions are real and the TZ gate is the defense.** The slug `popina` is a
  London venue, not Cobble Hill's Popina — caught because SR slots carry `time_iso` +
  `utc_datetime`, and NYC must be UTC-4/-5. Both discovery and every read enforce this.
  (SR parsing is fully verified against live venues as of 2026-09-01: mixed 12h/24h time
  formats, request-only filtering, seating areas, per-slot cancellation policies.)
- **Surface SR cancellation policies** — the adapter returns `cancellation` when present
  (e.g. Cafe Spaghetti charges $25/head for late cancels). Name it in the handoff message.
- Keep replies short and conversational on the SMS path; minimize round trips.

## One-time setup (pending Tom)

The booking half needs Tom's Resy auth token captured once (we never store his password):

1. `~/.claude/local-agents/agent-browser/launch.sh booking 9224`
2. In that Chrome window, sign in at `https://resy.com` (Tom's account) — must have a card on file.
3. `AGENT_BROWSER_PORT=9224 node ~/.claude/local-agents/agent-browser/resy_capture.mjs`,
   then do one authed action in the window (open a restaurant). It prints `RESY_AUTH_TOKEN`
   + payment methods.
4. Store the token encrypted (SOPS/age, same as the other secrets):
   `printf '%s' '<RESY_AUTH_TOKEN>' | sops -e --input-type binary --output-type binary /dev/stdin > ~/.claude/.resy-auth-token.enc`
5. Put the default payment method id in `.resy_config` (see `.resy_config.example`).
6. Kickstart the daemon so it picks up the new env:
   `launchctl kickstart -k gui/$(id -u)/com.invertedcap.sms-agent-daemon`

Until then, `search`/`find` work (proven live), but `book` returns "RESY_AUTH_TOKEN not set."
The token is long-lived; if `book` ever returns 401/419, re-run capture to refresh it.

## Notes

- Endpoints proven live 2026-09-01: `search` (venuesearch) + `find` (/4/find) return real
  venues + bookable slots with config tokens, using the public web key. `details`/`book`
  coded per the documented Resy flow (`/3/details` → `book_token`, `/3/book` with
  `struct_payment_method`), pending the auth token to validate end-to-end.
- Reverse-engineered the same way as Meevo (capture the app's real API headers), but Resy is
  the scriptable case the haircut memo flagged — see [[project_appointments]].
