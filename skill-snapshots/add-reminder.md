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
never marshals the whole 584-item list). Default list is **"Work"** (Tom's default
Reminders list — `eventkit add` targets it automatically).

**ALWAYS set an all-day due date** — a reminder with no due date does NOT appear in the
Calendar app's reminders row, which is where Tom looks for it. Default the due date to
**today**; use another day only if Tom names one. `--due` accepts `today`, `tomorrow`,
or `YYYY-MM-DD` (all-day, no timed alarm).

**List + prefix convention (Tom 2026-09-03, corrected same day):**
- **`Work`** = Tom's own work list (renamed from "Tasks" 2026-09-03). **No `[TS]`/`[EK]` prefix here** — everything in
  it is implicitly his, and the prefix is redundant noise (Tom: "these work tasks
  shouldn't have [TS] in front").
- **`Kenyon-Seo`** = shared personal/household list (groceries, kids/school, home,
  paying the chef/cleaner/sitter — anything domestic). Prefix convention DOES apply
  here since Elsie also uses it: `[TS]` = Tom's, `[EK]` = Elsie's, no prefix = shared/
  either. Check existing items on the list for an established pattern before guessing
  (e.g. groceries here has always been `[TS] Grocery Order` — match it, don't invent
  a new convention). Personal/household reminders route here via `--list "Kenyon-Seo"`
  — don't let them land in Work by default.
- **Recurring**: `--repeat weekly --weekday <name>` (e.g. `wednesday`) attaches a real
  EKRecurrenceRule — due date auto-snaps to the next occurrence of that weekday.
- **Renaming**: `eventkit rename --id ID --title T`.
- **Moving between lists**: `eventkit move --to LIST --id ID` (recreates the item in
  the destination and deletes the original under the hood — EventKit can't reassign
  an existing iCloud reminder's list via a plain save, error -3002; the CLI handles
  this transparently, just know a moved reminder gets a new id).

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
asks for a specific time. Scope any lookups to `list "Work"`.

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
- **Calendar: ALWAYS the Elsie-Tom shared calendar**
  (`cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com`) — reminders are personal/
  family items and that's where both of them look. **NEVER the primary (work) calendar**
  (Tom 2026-09-03, after a "milk and bread" fallback landed on tseo@primary.vc). No
  resolution logic, no "if absent" branch — the id is fixed; if the create fails, report
  the failure instead of retargeting.
- **Alert on first use:** falling back at all means Reminders is broken — say so in the
  reply (⚠️ + the real eventkit error), don't degrade silently.

## Conventions (both paths)

- **Date**: default to **today** (see `currentDate` in context). Only use another day
  if Tom names it. Never ask — infer and proceed.
- **Title**: the reminder text verbatim, lightly cleaned ("remind me to send Shivan
  $15" → "Send Shivan $15"). Sentence case, imperative.
- **"re" → "re:"** — when a title contains the word "re" meaning "regarding" (e.g.
  "ping X re Y"), format it as "re:" with a colon ("Ping X re: Y"), never bare "re"
  (Tom, 2026-09-10).
- **One reminder per item**: split conjoined asks ("A and B") into separate reminders.
- **Assignee prefix** — see the **List + prefix convention** above: applies on
  `Kenyon-Seo`, not on `Work` (Tom's own list, no prefix). Where it applies, bracketed
  with a space before the title (Tom, 2026-09-01 — was bare `TS `/`EK ` before):
  - `[TS] ` prefix = Tom's task ("remind ME" from Tom, or Tom named explicitly).
  - `[EK] ` prefix = Elsie's task ("remind ME" from Elsie, or Elsie named explicitly).
  - No prefix = shared/family task (either of them could do it, or it's ambiguous).
  - Resolve "me" to the sender (see sms-listener's allowlist resolution). If a name
    other than the sender is given ("remind Elsie to..."), prefix for THAT person.
  Example: Elsie texts "remind me tonight 8:30 to book a rental car" →
  title `[EK] Book a rental car for Pittsburgh` on `Kenyon-Seo`.
- Confirm back as a compact ✅ checklist, noting the date if not today and noting if
  the fallback path was used.

## Autonomous creation AND completion — always alert (Tom 2026-09-02, broadened 2026-09-03)

**Blanket rule, applies to every skill/script that can call `eventkit add` or
`eventkit complete`:** whenever a reminder is CREATED or checked off WITHOUT Tom or
Elsie directly commanding it in that turn — i.e. some background/watcher process
decided on its own (an invoice email arrived, a payment confirmation was spotted in
a text thread, a vendor texted that work is done) — it must send a short text alert
announcing what happened and why. Tom 2026-09-03: "as a blanket policy, whenever
you create a reminder you need to send an alert." This is distinct from an
interactive create/complete ("remind me to X" / "mark the Katya reminder done"),
which already gets a reply in the normal request/response flow.

**Channel routing — per flow, not per action** (both steps of a flow use the SAME
channel; iMessage thread names: "KSeo Bot" = the family group, "Bot" = Tom 1:1):
- **Family group** (`send_imessage.sh --group "<family_group_id>"`) — anything
  personal/household that Elsie would care about: Katya invoices (both arrival and
  paid), Zelle/maintenance payment confirmations, family calendar-adjacent items.
- **Tom 1:1** (`send_imessage.sh "+12012567714"`) — Tom-only operational stuff:
  Lupe office-cleaning (both creation and paid — Tom explicitly wants this one out
  of the family thread), anything on his Work list.
When adding a new autonomous flow, ask which side of that line it falls on (or ask
Tom); never mix channels within one flow.

Minimum shapes: create → `📋 <Title> — <what arrived / what triggered it>`;
complete → `✅ <Title> — <one-line reason>` (e.g. `✅ Katya Invoice Paid — saw the
payment confirmation in the thread`). Silent autonomous action is the failure mode
Tom is guarding against — a reminder that appears or vanishes with no explanation
is worse than one that lingers. Live instances: Katya two-step
(`check_katya_invoice.py` → group; `check_katya_paid.py` → group), Lupe two-step
(`check_lupe_paid.py` → 1:1 both sides), Citizens Zelle confirmations
(`check_payment_confirmations.py` → group). Any new autonomous logic follows the
same pattern.

## Family TODO digest — reads Kenyon-Seo, NOT Work

**Corrected 2026-09-03** (caught before `family-todo-digest` was ever actually built —
no scheduled task exists on disk yet, so this was a doc bug, not a live incident). Any
family-facing TODO digest ("what's on the todo list", an 8am family-group send) must
read the **`Kenyon-Seo`** list, never `Work` — `Work` is Tom's own fund/business list
(property tax, LP transfers, fund admin $ figures) and has no business going out over
the family thread. `~/.claude/skills/sms-listener/todo_digest.py`
(`fetch_reminders()`) targets `Kenyon-Seo`, grouping by due-date bucket then by the
TS/EK/no-prefix convention above. No separate TODO store — Apple Reminders IS the
TODO list, scoped to the shared list. Keep this pointed at `Kenyon-Seo` if the
mechanism is ever built out.
