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

## Primary path — Apple Reminders (via AppleScript)

Create the reminder in the macOS Reminders app with `osascript`. Default list is the
default Reminders list; use a `remind me on <date>` due date only if Tom names a day,
otherwise create it with no alarm (a plain to-do for "today").

```bash
osascript -e 'tell application "Reminders" to make new reminder with properties {name:"Send Shivan $15"}'
```

With a due date (only when Tom names one — build the date string, do NOT use
`current date` since it's nondeterministic; construct from `currentDate` in context):

```bash
osascript -e 'tell application "Reminders" to make new reminder with properties {name:"Text Sydney", due date:date "Friday, August 8, 2026 9:00:00 AM"}'
```

**One reminder per item.** Batch the `osascript` calls in one turn for multiple items.

### First-run permission (one-time)

The first Reminders AppleScript call triggers a macOS automation-consent dialog and
will fail with `AppleEvent timed out (-1712)` until Tom clicks **OK** at his Mac. If
you hit `-1712`: tell Tom to approve the dialog on his computer, then retry. Don't
silently fall back if he's at the machine — the grant is one-time and unblocks
everything after.

## Fallback path — Google Calendar all-day event

If AppleScript isn't available (headless run, permission not yet granted and Tom is
away), create an **all-day event marked FREE** on the calendar instead, and tell Tom
it's the fallback:

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
- Confirm back as a compact ✅ checklist, noting the date if not today and noting if
  the fallback path was used.
