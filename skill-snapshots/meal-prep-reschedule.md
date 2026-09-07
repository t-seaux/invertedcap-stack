---
name: meal-prep-reschedule
description: >
  Watches Tom's text thread with Katya Basil (the weekly meal-prep cook) for requests to
  move the cooking day/time. When Katya asks to reschedule, relays the ask into the KSeo Bot
  household thread and waits — it moves the "Katya cooks" calendar block ONLY after Tom or
  Elsie confirms in that thread, then posts a confirmation there. Hard gate: Katya's request
  alone never moves anything. Scheduled sweep only (launchd every 5 min via sweep.sh →
  claude-job-queue); not user-triggered inline. Args: {mode:"scan", messages:[...], pending:{...}}.
---

# Meal-Prep Reschedule Watcher

Katya Basil cooks for the family on a recurring weekly slot (currently **Wednesday mornings,
10–12**, event **"Katya cooks"** on the shared **Elsie-Tom** Google calendar). When she needs
to move a week, she texts Tom. This skill catches that, relays it to Tom + Elsie, and — **only
on their confirmation** — moves the calendar and confirms back.

> This runs headless from the claude-job-queue (dispatched by `sweep.sh`). Treat it as an
> unattended run: act on clear signals, exit silently otherwise, never ask questions.

## Fixed references (provenance: built 2026-09-06)

- **Katya (cook) handle:** `+19178224622` — REQUEST lane (inbound only).
- **KSeo Bot household thread:** chat_identifier `b9ba8cdcad424bb8b2bb9c7c3b6c7308`,
  Sendblue group `sb_group_5c132933-f44d-4e48-a929-2209854f2035`. CONFIRM lane + where all
  relays/confirmations are posted.
- **Confirmers:** Tom (`is_from_me = 1`) or Elsie (`+16179219845`). The bot's own posts
  (`+13603178168`) are never a confirmation.
- **Calendar:** `cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com` (Elsie-Tom).
  Event summary **"Katya cooks"**, recurring series id `jj47t2vmr762pe0tqf9shl2ou4`,
  default window **10:00–12:00 America/New_York**.
- **State files** (in this skill dir): `.pending.json` (the one open request, if any),
  `.actioned` (append-only ledger of completed moves, one line per action), `.last_rowid`
  (owned by sweep.sh — do not touch), `sweep.log`.
- **Send helper:** `./notify_family.sh` (reads body on STDIN; do not pass text as argv).

## Input

`{mode:"scan", messages:[{rowid, ts, sender, chat, from_me, lane, text}], pending:{...}|null}`

- `lane` is `"request"` (from Katya) or `"confirm"` (Tom/Elsie in the family thread).
- `text` is truncated to 500 chars — fine here; re-read the full row by rowid only if a body is cut mid-sentence.
- `pending` is the current `.pending.json` contents (or null). Trust the file on disk over this snapshot if they differ.

## Step 1 — Load state

Read `.pending.json` and `.actioned`. Process REQUEST-lane messages first (they may set/replace
pending), then CONFIRM-lane messages.

## Step 2 — REQUEST lane (messages from Katya)

For each Katya message, decide if it is a **cooking-schedule change** (move/reschedule/swap the
day or time; e.g. "can we move to Thursday morning", "I'll come Friday instead this week",
"push to the afternoon"). If it's ordinary chatter (menu, running late, groceries), **ignore it**
— do not relay, do not touch the calendar.

If it IS a reschedule request:

1. Extract:
   - **target day** (resolve to a concrete date relative to `ts`; "this coming week" = the
     instance in the upcoming Mon–Sun week).
   - **target time** — default to the existing **10:00–12:00** window unless she states a new
     time; preserve the 2-hour duration if she gives only a start.
   - **scope** — default **single instance** (this one week). Treat as a permanent series change
     ONLY on explicit language ("from now on", "going forward", "every week", "permanently").
2. Locate the affected **"Katya cooks"** instance: `list_events` on the Elsie-Tom calendar for
   the target week; find the instance of series `jj47t2vmr762pe0tqf9shl2ou4`. Capture its current
   date/time (the "from") and its instance `eventId`.
   - If no instance is found for that week, post a heads-up to the family thread ("⚠️ Katya asked
     to move this week's cooking but I can't find the event to move — can you check?") and do NOT
     write pending. Stop.
3. Write `.pending.json`:
   ```json
   {"status":"awaiting_confirm","requested_at":"<ts>","katya_rowid":<rowid>,
    "calendar_id":"cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com",
    "event_id":"<instance eventId>","scope":"single|recurring",
    "from":"<Wed 9/9, 10–12>","to_date":"YYYY-MM-DD",
    "to_start":"YYYY-MM-DDTHH:MM:00-04:00","to_end":"YYYY-MM-DDTHH:MM:00-04:00",
    "to_label":"<Thu 9/10, 10–12>"}
   ```
   Overwrite any prior pending (a newer request supersedes an unconfirmed older one).
4. Relay into the family thread (do NOT move the calendar yet):
   ```
   ./notify_family.sh <<'MSG'
   🍳 Katya asked to move this week's cooking: <from> → <to_label>. Reply 👍 (or "yes") here to confirm and I'll update the calendar.
   MSG
   ```

## Step 3 — CONFIRM lane (Tom or Elsie in the family thread)

Only act if `.pending.json` exists with `status:"awaiting_confirm"`. Otherwise ignore
(a family-thread message with nothing pending is not a confirmation).

Classify the Tom/Elsie message **in the context of the pending relay**:

- **Affirmative** (yes, yup, yep, 👍, ok, sounds good, works, go ahead, do it, confirmed, sure,
  "no worries"): perform the move.
- **Counter-proposal** (a different day/time, "actually Friday?"): update `.pending.json` to the
  new target (re-locate the instance/date), re-relay the corrected ask, and keep waiting.
- **Decline** (no, cancel, never mind, leave it): delete `.pending.json`, post a one-line ack
  ("👍 Leaving the cooking schedule as-is."), stop.
- **Unrelated chatter**: ignore.

### On affirmative — move + confirm

1. **Idempotency:** if `.actioned` already contains a line for this pending's `katya_rowid`,
   the move is done — skip (do not double-move or double-post).
2. Move the event via `update_event` on the Elsie-Tom calendar:
   - `single` scope → update the instance `event_id` with `startTime = to_start`, `endTime = to_end`.
   - `recurring` scope → update the series `jj47t2vmr762pe0tqf9shl2ou4` (rare; only if pending.scope=recurring).
3. Post the confirmation to the family thread:
   ```
   ./notify_family.sh <<'MSG'
   ✅ Calendar updated: Katya's cooking moved <from> → <to_label>. <recurring? "Applied going forward." : "Recurring Wednesday slot unchanged otherwise.">
   MSG
   ```
4. Append to `.actioned`: `<YYYY-MM-DD HH:MM> | katya_rowid=<n> | confirm_rowid=<n> | <from> -> <to_label> | scope=<single|recurring> | by=<Tom|Elsie>`.
5. Delete `.pending.json`.

## Step 4 — Housekeeping / edge cases

- **Stale pending:** if `pending.to_date` is already in the past and still unconfirmed, delete
  `.pending.json` silently (the window was missed — do not move retroactively).
- **One open request at a time.** A second reschedule before the first is confirmed replaces it.
- **Never move on Katya's word alone.** The Tom/Elsie confirmation is a hard gate — no exceptions.
- **Never dual-notify.** Confirmations and alerts go to the KSeo Bot thread only (not Slack, not
  Tom's 1:1). Follow the surface.
- **Money/`$` in any message body:** send via the `<<'MSG'` heredoc (as shown) so `notify_family.sh`
  doesn't mangle or refuse it.
- If a calendar call fails (e.g. OAuth lapse), post a heads-up to the family thread and leave
  `.pending.json` in place so the next confirmation retries — do not silently drop.

## Exit

No request and no actionable confirmation → exit silently. No log noise, no family-thread post.
