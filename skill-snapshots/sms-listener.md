---
name: sms-listener
description: "Processes inbound SMS/MMS texts sent to Tom's Twilio number (978-733-7893). An allowlisted sender (Tom or Elsie) texts a command — most often a calendar query or add — and this skill executes it and texts back the reply (Twilio REST, iMessage fallback while A2P registration is pending). Webhook-only — invoked by claude-job-queue dispatching jobs from the sms-webhook Cloudflare Worker on inbound-sms events."
---

# SMS Listener

An allowlisted person texted Tom's Twilio number; the `body` arg is their command. Execute it and text back the result. This is a **calendar-first personal agent** — most commands are calendar queries/adds. Full tool access (filesystem + all MCP).

**Speed matters — minimize round trips.** Batch independent tool calls in one turn. A routine command should finish in ≤5 tool-use turns total. Don't read other skills' SKILL.md for calendar work (fast path below covers it); only read another skill for non-calendar commands that clearly invoke it (reminders → `add-reminder`, CRM → `add-to-crm`, **buy/order/book anything → `purchase-agent` — quote first, money moves ONLY on an explicit YES**, etc.).

## Args (from the Worker)

```json
{
  "mode": "webhook",
  "from": "+1XXXXXXXXXX",      // sender — also who you reply to
  "to": "+1YYYYYYYYYY",
  "body": "<the command>",
  "message_sid": "SM...",
  "num_media": 0,
  "media_urls": [],            // MMS: fetch with curl -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN"
  "media_types": []
}
```

**Who is texting:** resolve `from` against `~/.claude/skills/sms-listener/.allowlist` (lines `+E164 = Name`; read it in the same turn as your first action). **Tom** → full authority; "my work calendar" = Inverted. **Elsie** (Tom's wife) → **full peer on the Elsie-Tom calendar** — adds, queries, moves, explicit deletes, identical rules; her only fence is Tom's work side (no investor/deal/CRM actions unless clearly asked and obviously fine). Unknown-but-dispatched → household scope.

**Pronouns resolve to the sender.** From Elsie, "my therapy Tuesday 5" = an `EK` event; from Tom, "my run" = `TS`. Availability is ALWAYS computed against Tom's time regardless of sender (EK/kid events = FREE, per the fast path below).

## Calendar fast path

(Digest of `add-to-calendar` — the full skill is the source of truth; keep in sync.)

**Calendars** (`mcp__claude_ai_Google_Calendar__*`):
- **Personal / family / kids / school** → Elsie-Tom shared: `cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com`
- **Work** → `tom@invertedcap.com` (also: Dash `tom@dashfund.co`, Primary `tseo@primary.vc`)
- "my calendar today" with no cue → check personal + Inverted work in ONE parallel turn.

**Rules for adds:**
1. **Dedup first, always** — `list_events` over the day; same date + overlapping time + equivalent title (judge semantically) → skip, report as existing. BFS school feed pre-populates milestones.
2. **Times** — America/New_York; bare hour: soccer/sports = PM; default duration 1h.
3. **Title prefixes (personal cal)** — `TS` = Tom solo · `EK` = Elsie solo · kid's name (`Andy Soccer`, `Benny Music Class`) = kid activity · no prefix = family/joint. School-feed style: `BFS: <event>`.
4. **Busy/Free** — one test: does it occupy *Tom*? Tom-solo/joint/parent-required-school/family-OOO-trips → BUSY. Kid activities, EK events, informational all-day markers → FREE. Unsure → FREE.
5. Confirm which calendar + time + availability in the reply.

## Family folder (shared Drive)

"Our folder" / "family folder" / "the Kenyon-Seo folder" = Google Drive folder
**Kenyon-Seo**, id `10BJN5vS5Xld8B3sE29oqrsbchs6K0hVK` (shared Tom + Elsie, both
editors). "Save this to our folder" (incl. MMS attachments — download the media
first) → upload there per `drive-save/SKILL.md`'s Apps Script endpoint, reply with
the file's Drive link. Listing/fetching: `mcp__claude_ai_Google_Drive__*` scoped to
that folder id.

## Execute

- Do exactly what was asked; respect the sender's scope. Headless — never ask questions except a single `❓` reply-and-exit when a wrong guess would cause real harm (wrong recipient, ambiguous deletion, wrong page). Low stakes → decide and proceed.
- **Deletes ARE allowed on explicit command.** The sender texting "delete X" IS the
  confirmation — the always-confirm rule gates agent-initiated destruction, not the
  sender's direct order. If the referent is unambiguous (named event, or the thing this
  conversation just created/moved), delete it. Ambiguous referent → one `❓` and exit.
  The Google Calendar MCP has NO delete tool, and the osascript Calendar.app delete
  does NOT reliably sync back to Google (verified 2026-08-31 — local delete never
  propagated). So "delete" = **neutralize via `update_event`**: title → `(deleted)`,
  availability FREE, description noting who/when. Reply `✅ Deleted <title>`. Tom can
  purge `(deleted)` husks whenever; a real API delete needs an Apps Script endpoint
  (future improvement).
- Reply ONLY by text to `from`. Keep it SHORT — a text message, plain text, ≤3 lines typical (`send_sms.sh` truncates >1500 chars).

## Reply + verify + audit (ONE Bash call)

```bash
S=$(/Users/tomseo/.claude/skills/sms-listener/send_sms.sh "+1..." "✅ <short reply>") && SID=${S#ok }
ST=queued; for i in 1 2 3 4; do sleep 2
  ST=$(/Users/tomseo/.claude/skills/sms-listener/check_delivery.sh "$SID")
  [ "$ST" != "queued" ] && [ "$ST" != "sending" ] && [ "$ST" != "sent" ] && break; done
echo "status=$ST"
echo "[$(date '+%Y-%m-%d %H:%M:%S')] sid=<message_sid> from=<from> intent=<tag> outcome=applied status=$ST notes=<what>" \
  >> /Users/tomseo/.claude/skills/sms-listener/audit-log/$(date +%F).log
```

- `delivered`/`sent` → done.
- `undelivered`/`failed` (carrier block 30034 — A2P registration pending) → send the SAME text via iMessage: `mcp__imessages__tool_send_message`, `recipient` = `from`. Append ` via=imessage` to the audit line (one more tiny Bash call is fine).
- Reply formats: `✅ <result>` · `❓ <question>` · `⚠️ couldn't — <reason>`.

## Notes

- Config: `.twilio_config` (SID + From number); `TWILIO_AUTH_TOKEN` injected by the processor env.
- Idempotency: the queue dedups on `message_sid`; if a job reprocesses, grep the audit log for the sid and exit 0 if handled.
- Long work (>10 min): text an interim "on it — few min" and continue.
