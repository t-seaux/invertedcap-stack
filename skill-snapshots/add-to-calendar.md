---
name: add-to-calendar
description: >
  Add an event to one of Tom's Google Calendars, with duplicate-checking baked in.
  Trigger whenever Tom says "add to calendar", "add to my calendar", "put this on my
  calendar", "add to personal calendar", "add to work calendar", "calendar this", or
  hands over an event (a flyer/screenshot, an email, a date+time, a school notice)
  with intent to put it on a calendar. Routing: "personal" → the Elsie-Tom shared
  household calendar; "work" → Tom's Inverted calendar. If Tom doesn't say which,
  infer from the event (family/kids/school/home → personal; meetings/deals/investors/
  founders → work) and default to work if genuinely unclear — always naming which
  calendar was used so Tom can correct. ALWAYS check for existing matching events first
  and never create a duplicate. Always trigger inline — no confirmation needed.
  NOT for meetings with other people ("schedule a meeting / call / coffee with X",
  "book 30 min with Z", "move my meeting with X") — those are the scheduler skill's
  direct path (Bot conventions + a 👍 card); holds with no invitees stay here.
---

# Add to Calendar

**The ONE spec for adding an event — every surface reads this file** (a session, the text agent `sms-listener`, any
other lane). Tom 2026-10-11: "text and Claude sessions should refer to the same brain… the outputs should be the same
and the methodology should be the same." There is no separate text "fast path" any more; the rules below are code in
`scripts/cal_add.py`, so both surfaces produce the same event and the same reply.

## Not here: meetings with other people → the scheduler (Tom 2026-10-11)

"schedule / set up / book a meeting / call / coffee / lunch with <person>", "move / cancel my meeting with <person>"
→ **`~/.claude/skills/scheduler/SKILL.md` § Direct bookings** (`scheduler/direct.py`): Bot's conventions (title, length,
Meet, venue, footer, rulebook, ledger) and a 📅 card Tom 👍s before any invite goes out. "Whenever I invoke something
that could also be done via bot@invertedcap.com … follow all the conventions you do with bot." This skill never
invites anyone – `cal_add.py` refuses a request with guests (exit 2) – so it never carries the Bot footer either.
Stays HERE: personal / family events, Tom's solo blocks, and **holds with no invitees** ("hold 2pm for Pat", "block
Thursday afternoon") – a hold is Tom's own time, nobody gets an invite.

## 0. Calendar before web search, always (Tom 2026-08-31)

For ANY scheduling question ("when is X", "do we have anything Saturday", "is X already on the calendar") OR before
proposing an add, read the calendar for that range FIRST — it is the source of truth for what's scheduled (the BFS
feed pre-populates school milestones). Never answer a date question from a web search before checking it.
Haircut date checks ("when's my next haircut") → the `haircut` skill § Lookup (never search the word "haircut").

## 1. Add = one command

Extract the request (your judgment: what the event is, whose, and personal vs work when there's a real cue), then:

```bash
python3 ~/.claude/skills/add-to-calendar/scripts/cal_add.py add '<request json>'      # or - for stdin
```

Request: `{"title", "date"?, "end_date"?, "time"?, "end_time"?, "all_day"?, "calendar"?: personal|work|dash|primary,
"who"?: tom|elsie|joint|andy|benny, "kind"?: sports|social|meeting|kid|parent_school|ooo|marker, "source"?: "BFS",
"location"?, "seats"?, "description"?, "availability"?: busy|free}` — omit what Tom/the source didn't give. Pass
Tom's words through (bare "8" stays "8"; no date stays absent) so the code applies the defaults, not you.

| Exit | Meaning → what you do |
|---|---|
| 0 | created → reply with the printed `text`, verbatim |
| 10 | one possible duplicate (`duplicates`) → judge it; same event = reconcile it IN PLACE on the calendar it lives on (fill missing, correct conflicts – `shared-references/calendar-event-handling.md` invariant 3; `calendar_write.py patch`), never a copy; then reply with `cal_add.py card` (§ 3). Not the same event → re-run with a sharper title/keyword |
| 11 | two+ candidates → ask which, don't guess |
| 2 | bad request or calendar unreachable → nothing written; say so |
| 3 | location refused (no street address) → write `Venue Name (street address)` and re-run |
| 1 | create failed → report the printed error |

A batch (a school flyer with many rows) = one `add` per row; dedup runs per row, add only the net-new.
`cal_add.py plan '<json>'` shows the event without writing (debug / a proposal card).

## 2. The rules the code applies (the WHY — `tests/test_cal_add.py` pins each)

- **Which calendar.** "personal" / home / family / Elsie-Tom → **Elsie-Tom** (`cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com`);
  "work" / Inverted / meetings, deals, investors, founders → **Inverted** (`tom@invertedcap.com`); Dash
  (`tom@dashfund.co`) and Primary (`tseo@primary.vc`) by name. Family / kids / school content → personal.
  **No cue and no clear signal → work**, and the reply names it so Tom can move it. Tom's soccer, BFS, Elsie, kids
  and joint events are personal by definition.
- **No date named → TODAY; no date and no time → today ALL-DAY** (Tom 2026-09-03) — never tomorrow. (The
  proactive-reminder flow's next-day default is for unprompted creates only.) **"Full day" / "all day" is an
  event-type instruction** (Tom 2026-09-15): `all_day: true`, never a 9–5 range.
- **Bare hour, no am/pm.** Soccer and sports/games are always PM (a bare "soccer at 8" = 8:00 PM); dinner / drinks /
  parties / shows lean PM; otherwise 1–6 → PM, 7–11 → AM. The assumption is stated in the reply.
- **Duration** 1 hour unless an end is given. Time zone America/New_York.
- **Title.** Clean and specific. **BFS events read `BFS: <event>`** (Tom 2026-10-11 — colon form; any `[BFS]` /
  `BFS` in the source name is stripped first: "BFS FallFest" → `BFS: FallFest`). Elsie-Tom prefixes: `TS` = just
  Tom (`TS Dentist`), `EK` = just Elsie (`EK Therapy`), the kid's name for kid events (`Andy Soccer`, `Benny Music
  Class`), no prefix = family / joint (`Nutman wedding`); unclear solo vs joint → no prefix. Never `TS` on the work
  calendar.
- **Tom's soccer games** (Tom 2026-09-16, standing default — never ask): titled `TS Soccer` (keeps `vs. <opponent>`),
  on Elsie-Tom, BUSY, 1 hour from the stated start, location **Brooklyn Bridge Park Pier 5 (334 Furman St, Brooklyn,
  NY 11201)** unless another venue is given. Already on the calendar → never a second one (dedup).
- **Busy vs Free — one test: does this make TOM unavailable for work?** BUSY = Tom's own game / appointment /
  dinner (`TS` and joint events), parent-attended school events (Grown-Up Gathering, conferences, curriculum night),
  a family trip / OOO day, work meetings. FREE = the kids' activities, Elsie's events, informational all-day markers
  (first day of school, closures, birthdays, "applications open"). Unsure → FREE. Shown in the reply.
- **Seats** (concerts, games, shows): always on the event — `Seats: …` leads the description — and in the reply
  (Tom 2026-09-02).
- **Location / description** when the source gives them. A person's home → offer to save it to their contact
  (`shared-references/calendar-event-handling.md` rule 10). Feed-populated BFS milestones are already there — dedup
  catches them.

## 3. The reply (both surfaces, same text)

The confirm is code: `calendar_write.py`'s 📅 card (the canonical renderer every calendar lane uses) + `Where:`,
`Seats:`, `Cal: <calendar> · Busy|Free`, any assumption (calendar defaulted to work, AM/PM, today all-day), and the
**event link** last as `<url> ↗` — always (Tom 2026-09-02). After `add`, send the printed `text`. For a reconcile or
an already-on-cal answer:

```bash
python3 ~/.claude/skills/add-to-calendar/scripts/cal_add.py card '{"calendar":"<id>","items":[{"before":<event>,"after":<event>}]}'
```

Never ask before acting — infer, add, and report; Tom corrects from the reply. Anything you deliberately left off
(a skipped duplicate, a row you didn't add) gets one line with the reason.
