---
name: add-reminder
description: >
  Add a reminder for Tom. In Tom's usage, "add a reminder" means a NATIVE Apple
  Reminder (the "Reminder" tab in the iOS/macOS Calendar app — syncs via iCloud,
  pings his phone). Trigger whenever Tom says "add a reminder to X", "remind me to
  X", "add X to a reminder", "reminder: X", "add a reminder for X", or hands over
  one or more to-do items with intent to reminder them. One reminder per item — if
  Tom lists several things ("remind me to do A and B"), create a separate reminder
  for each. Defaults to today's date unless Tom names a day ("tomorrow", "Friday",
  "on the 12th"). Always trigger inline — no confirmation needed before acting.
---

# Add Reminder

Tom's shorthand: **"add a reminder to X" = a native Apple Reminder.** This is what he
showed in the Calendar app's "Reminder" tab — an iCloud reminder that pings his phone,
not a calendar event.

## Primary path — Apple Reminders (via the native `eventkit` helper)

Use the native EventKit CLI at `~/.claude/tools/eventkit/eventkit`. It talks to
Reminders directly instead of through AppleScript's slow bridge (10-50x faster; it
never marshals the whole 584-item list). Default list is **"Tasks"** (Tom's default
Reminders list — `eventkit add` targets it automatically).

**ALWAYS set an all-day due date** — a reminder with no due date does NOT appear in the
Calendar app's reminders row, which is where Tom looks for it. Default the due date to
**today**; use another day only if Tom names one. `--due` accepts `today`, `tomorrow`,
or `YYYY-MM-DD` (all-day, no timed alarm).

Today (default):

```bash
~/.claude/tools/eventkit/eventkit add --title "[TS] Send Shivan \$15" --due today
```

A named future day — build `YYYY-MM-DD` from `currentDate` in context:

```bash
~/.claude/tools/eventkit/eventkit add --title "[TS] Text Sidney" --due 2026-08-08
```

Returns JSON (`{"ok":true,"reminder":{...}}`); on failure `{"ok":false,"error":...}`
and exit 1. Other verbs: `list [--all]`, `count`, `complete/uncomplete --id`,
`delete --id`, `move --to LIST --id`, `clear-completed` (see `eventkit help`). The
`--id` verbs accept multiple `--id` flags and commit as one batch — that's the fast
path for bulk moves/edits/clears.

**One reminder per item.** Batch the `eventkit add` calls in one turn for multiple items.

### Fallback — AppleScript (`osascript`)

If the `eventkit` binary is missing (e.g. not yet rebuilt on a fresh machine), fall
back to `osascript`. Recompile with
`swiftc -O -Xlinker -sectcreate -Xlinker __TEXT -Xlinker __info_plist -Xlinker Info.plist -o eventkit main.swift`
from the tool dir if needed.

```bash
osascript -e 'tell application "Reminders" to set myR to make new reminder with properties {name:"Send Shivan $15"}' -e 'tell application "Reminders" to set allday due date of myR to (current date)'
```

Use `allday due date` (date only) rather than `due date` (adds a timed alarm) unless Tom
asks for a specific time. Scope any lookups to `list "Tasks"`.

### Permissions (one-time, already granted)

`eventkit` uses EventKit's Reminders access (System Settings › Privacy & Security ›
Reminders) — already granted. If it ever reports access denied, Tom re-enables it there.
The AppleScript fallback uses a separate Automation grant; a first `-1712` timeout means
Tom must click **OK** on the consent dialog at his Mac, then retry.

## Fallback path — Google Calendar all-day event

**Retry before you fall back — don't skip straight to the calendar on one hiccup**
(Tom 2026-09-02, after a stray "[TS] Check email from pink room" calendar event got
created despite `eventkit` being perfectly healthy at the time — root cause was
skipping straight to the calendar fallback instead of retrying/diagnosing). Before
using this path:
1. Confirm `~/.claude/tools/eventkit/eventkit` actually exists — if it's missing,
   that's real, go to AppleScript.
2. If it exists but the `add` call errored, **retry once.** Most failures at this
   layer are transient (a momentary EventKit/TCC hiccup), not a real outage. Only
   treat it as "Reminders unavailable" after a genuine second failure, and quote the
   actual `{"ok":false,"error":...}` string in your own reasoning rather than a vague
   "hit an issue."
3. Only then try the AppleScript path, and only fall through to the calendar-event
   fallback below if BOTH eventkit and AppleScript genuinely fail.

If both are genuinely unavailable (headless run, permission not yet granted and Tom
is away), create an **all-day event marked FREE** on the calendar instead, and tell
Tom it's the fallback — clearly, with the real reason, not a generic "unavailable":

- `create_event` with `allDay: true`, `availability: "AVAILABILITY_FREE"`,
  `startTime` = target date (`YYYY-MM-DD`), `endTime` = next day.
- Calendar: resolve a `Reminders` calendar via `list_calendars` (match
  `summary == "Reminders"`); fall back to primary if absent.

## Conventions (both paths)

- **Date**: default to **today** (see `currentDate` in context). Only use another day
  if Tom names it. Never ask — infer and proceed.
- **Title**: the reminder text verbatim, lightly cleaned ("remind me to send Shivan
  $15" → "Send Shivan $15"). Sentence case, imperative.
- **One reminder per item**: split conjoined asks ("A and B") into separate reminders.
- **Assignee prefix (title, mirrors the calendar TS/EK convention)** — this is what
  lets the family-todo-digest scheduled task group reminders by who owns them.
  Format is bracketed, with a space before the title (Tom, 2026-09-01 — was bare
  `TS `/`EK ` before):
  - `[TS] ` prefix = Tom's task ("remind ME" from Tom, or Tom named explicitly).
  - `[EK] ` prefix = Elsie's task ("remind ME" from Elsie, or Elsie named explicitly).
  - No prefix = shared/family task (either of them could do it, or it's ambiguous).
  - Resolve "me" to the sender (see sms-listener's allowlist resolution). If a name
    other than the sender is given ("remind Elsie to..."), prefix for THAT person.
  Example: Elsie texts "remind me tonight 8:30 to book a rental car" →
  title `[EK] Book a rental car for Pittsburgh`.
- Confirm back as a compact ✅ checklist, noting the date if not today and noting if
  the fallback path was used.

## Autonomous completion — always alert (Tom 2026-09-02)

**Broad rule, applies to every skill/script that can call `eventkit complete`:**
whenever a reminder is checked off WITHOUT Tom or Elsie directly commanding it in
that turn — i.e. some background/watcher process decided on its own that the
underlying task is done (a payment confirmation spotted in a text thread, an email
that resolved an open item, etc.) — it must send a short text alert to the group
announcing what got checked off and why. This is distinct from an interactive
completion ("mark the Katya reminder done" / "clear X"), which already gets a reply
in the normal request/response flow and needs no separate alert.

Minimum shape: `✅ <Title> — <one-line reason>` (e.g. `✅ Katya Invoice Paid — saw
the payment confirmation in the thread`). Send via the sms-listener reply helpers
(`send_imessage.sh --group "<family_group_id>"` for anything on the household side).
Silent autonomous completion is the failure mode Tom is guarding against — a
reminder that vanishes with no explanation is worse than one that lingers.
First instance: `family-inbox/check_katya_paid.py` + `katya_paid_watch.sh` (see
that skill's SKILL.md section 4b). Any new autonomous-completion logic (future
skills) should follow the same pattern.

## This is the source of truth for the family TODO list

Every reminder created here (list "Tasks") feeds the `family-todo-digest` scheduled
task (`~/.claude/scheduled-tasks/family-todo-digest/`) — the 8am ET daily send to the
family group, and any on-demand "what's on the todo list" query. Both read the SAME
list via `~/.claude/skills/sms-listener/todo_digest.py`, grouping by due date bucket
then by the TS/EK/no-prefix assignee convention above. No separate TODO store — Apple
Reminders IS the TODO list. Keep this convention current if either mechanism changes.
