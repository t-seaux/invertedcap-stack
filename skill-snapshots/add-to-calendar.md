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
---

# Add to Calendar

Create a Google Calendar event for Tom via `mcp__claude_ai_Google_Calendar__create_event`.
**Duplicate-checking is not optional — it runs on every add.**

> `sms-listener/SKILL.md` carries a compact "Calendar fast path" digest of these rules
> for speed — if you change conventions here, update that digest too.

## 1. Pick the calendar

| Tom says | Calendar | ID |
|---|---|---|
| "personal", "home", "family", "Elsie/Tom", or a family/kids/school/home event | **Elsie-Tom** (shared household) | `cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com` |
| "work", "Inverted", or a meeting/deal/investor/founder event | **Tom (Inverted)** | `tom@invertedcap.com` |

- If Tom names **Dash** (`tom@dashfund.co`) or **Primary** (`tseo@primary.vc`), use those.
- If no cue and the content doesn't clearly signal personal vs work, **default to work (Inverted)**.
- Always state which calendar you used in the confirmation so Tom can move it if wrong.
- If unsure of an ID, resolve with `list_calendars` (match by `summary`).

## 2. Check for duplicates FIRST (mandatory)

Before creating anything, run `python3 ~/.claude/scripts/calendar_write/calendar_write.py find-dupes both --day <YYYY-MM-DD> [--start HH:MM] [--keyword <title/venue/person word>]...` — BOTH calendars (Elsie-Tom + tom@invertedcap.com), not just the target
(Tom 2026-09-19). exit 0 = no match → create · 10 = exactly one → reconcile THAT event in place, on the calendar it lives on (calendar-event-handling.md invariant 3: fill missing, correct conflicts, never a copy on the other calendar) · 11 = two+ → ask, don't guess · 2 = calendar unreachable → don't create. Judge the candidates semantically:

- **Match = same date (or overlapping start time) + same/equivalent title**, on EITHER calendar.
  Titles need not be byte-identical — "ECLS Family Visit Day" ≡ "EC: Family Visit Day",
  "First Day of School" ≡ "First Day of School (ECLS & MS)".
- If a match exists → **reconcile it in place** (fill what's missing, correct what conflicts — calendar-event-handling.md
  invariant 3), don't create a copy. Nothing to change → report it as already-on-calendar.
- Watch for feed-populated entries: the Elsie-Tom calendar auto-imports Brooklyn Friends
  School (BFS) milestones (Labor Day, Family Visit Day, First/Second Day). Those will
  already be present — don't re-add.
- For a batch (e.g. a school flyer with many rows), dedup each row independently; add
  only the net-new ones.

## 3. Create the event(s)

- **Timed event**: `startTime` / `endTime` as ISO 8601 with `timeZone: "America/New_York"`.
  If only a start time is given, default the duration to 1 hour.
- **AM/PM inference**: when Tom gives a bare hour with no am/pm, infer from context.
  **Soccer (and sports/games generally) is always PM unless Tom says otherwise** — a
  bare "soccer at 8" = 8:00 PM. For other events use common sense (a "call at 8" during
  the workday leans AM; a dinner/social event leans PM). State the assumption in the
  confirmation so Tom can correct.
- **All-day event**: `allDay: true`, `startTime` = the date, `endTime` = next day.
- **"Full day" / "all day" is a literal event-type instruction, not a duration hint**
  (Tom 2026-09-15). When Tom says "add a full day block/event for X" or "all day on X",
  create it with `allDay: true` — never a timed range (e.g. 9am–5pm) standing in for the
  whole day. The phrase names the event TYPE Tom wants, the same way "timed" or a clock
  time would.
- **No date named → TODAY, all-day** (Tom 2026-09-03). When Tom asks to add something
  without naming a day or time, it goes on today as an all-day event — never assume
  tomorrow. (Don't confuse this with the proactive-reminder flow, where a to-do spotted
  while reading threads defaults to the NEXT day — that rule is for unprompted creates
  only, never for something Tom explicitly asked to add.)
- **Busy vs Free** — set `availability` by ONE test: *does this actually make Tom
  himself unavailable for work?* It's about **Tom's** time, not whether something is
  happening.
  - **BUSY** = Tom is personally occupied: his own game/appointment/dinner (typically the
    `TS`-prefixed or joint Tom+Elsie events), or an all-day event where Tom is genuinely
    OOO — a family trip / travel / vacation (e.g. "we're all in Vermont"), a personal
    day off.
  - **FREE** = Tom keeps working: **the kids' activities** (music class, gym class,
    soccer practice, lessons — the kid attends, Tom's at work), and informational
    all-day markers (first day of school, school closed, birthdays, "applications
    open," reminders).
  - **Exception — parent-attended school events are BUSY**: anything explicitly for
    parents/grown-ups (Grown-Up Gathering, parent-teacher conferences, family visit
    day when timed, curriculum night) means Tom shows up → BUSY. The test is who
    attends: kid-only → FREE, parent-required → BUSY.
  - When unsure, default FREE — better to under-block Tom's work calendar. Note the
    availability in the confirmation.
- **Title**: clean and specific. For recurring institutional sources, prefix the source
  (e.g. `BFS: EC Grown-Up Gathering`) to match how existing entries read.
- **Prefix convention (Elsie-Tom calendar)**: `TS` = just Tom (e.g. `TS Soccer`,
  `TS Dentist`); `EK` = just Elsie (e.g. `EK Therapy`, `EK Robyn`, `EK Teal's Baby
  Shower`); **no prefix** = family event or shared Tom+Elsie activity (e.g. `Nutman
  wedding`). Kids' events use the kid's name (`Andy Soccer`, `Benny Music Class`).
  When unclear whether solo or joint, default to no prefix.
  Prefix ties into availability: `EK`-prefixed and kid-named events are FREE for Tom;
  `TS` and joint events are BUSY (per the Busy/Free rules above).
- **Location / description**: fill when the source gives them; otherwise omit.
  If the location is a person's home, offer to save it to their contact (`shared-references/calendar-event-handling.md` rule 10).
- Batch multiple `create_event` calls in one turn.

## 4. Confirm back

Compact ✅ checklist: what was added, the date/time, and **which calendar**. List anything
skipped as a duplicate, and anything intentionally left off (with a one-line reason), so
Tom can course-correct. Never ask before acting — infer, add, and report.
