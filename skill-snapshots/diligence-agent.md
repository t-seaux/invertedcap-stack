---
name: diligence-agent
description: >
  Orchestrates three diligence sub-agents: (1) Feedback Outreach Drafter — scans the Feedback relation field for new entries and drafts outreach emails for any not yet sent or drafted; (2) Feedback Outreach Scanner — scans sent mail and inbox for feedback outreach activity, logs Notion notes; (3) Pass Note Drafter — drafts investor pass notes for Pass Note Pending opportunities. Scheduled once daily on weekdays at 17:50 ET inside the evening digest (Monday covers the weekend). Trigger when the user says "run diligence agent", "diligence scan", "check feedback replies", "run pass notes", or any variant requesting the diligence tasks on demand. Registered as Agent 5 in run-all.
---

# Diligence Agent

Runs three diligence sub-agents in sequence, each in its own isolated context via the Task tool. It follows the same architecture as the run-all orchestrator: each sub-agent finds and reads its own SKILL.md at runtime. This orchestrator never pre-reads or inlines skill file contents.

## Schedule and window (one statement — 2026-10-04)

- **Scheduled path:** weekdays at 17:50 ET. The `com.tomseo.scheduled.evening-digest` launchd job runs `~/.claude/scheduled-tasks/diligence-agent/run.sh` with the other evening agents. The diligence section lands in the one 📬 Daily Agents digest. That path follows the scheduled-task copy (`~/.claude/scheduled-tasks/diligence-agent/SKILL.md`), which adds a first-pass dispatcher and builds the alert with `lib/compose_diligence_alert.py`. It does NOT use this file's Step 5. The old standalone `diligence-agent` plist (17:52) is parked in `_disabled-plists/`. The old "8am and 6pm ET" twice-daily cadence no longer runs.
- **Window:** Gmail lookback is **24h, or 72h on Mondays**. Saturday and Sunday are skipped. This is `RUN_LOOKBACK_HOURS` from `run.sh`. Notion-state filters (`📣 Pending Feedback`, `Pass Note Pending`) are state-based and ignore the window. On-demand runs of this file pass the same window to the sub-agents (see the Step 1–3 prompts), so the two paths read the same mail.
- **This file** covers on-demand and run-all invocations: three sub-agents, then `consolidate_alert.py`.

## Sub-Agents

This agent runs three sub-agents in sequence:

1. **Feedback Outreach Drafter** scans the `📣 Pending Feedback` relation across all opportunities for newly added people. It checks whether an outreach email has already gone to each person and drafts Gmail outreach notes for anyone who hasn't been contacted.
2. **Feedback Outreach Scanner** scans Gmail sent mail and inbox for feedback outreach activity. It creates and updates per-person Notion feedback notes.
3. **Pass Note Drafter** queries Notion for Pass Note Pending opportunities and drafts investor pass notes as Gmail drafts.

> **Note:** The Feedback Outreach Drafter can also be triggered manually (e.g. "draft feedback outreach note for X on Y"). When triggered manually, it skips the Notion scan and goes directly to drafting for the named people.

## Notification Behavior

**On demand** (not via run-all): after all three sub-agents finish, run `consolidate_alert.py --send` (Step 5). It sends one consolidated Slack alert to `#claude-alerts` through `send-alert/send.sh`.

**Via run-all:** run `consolidate_alert.py` WITHOUT `--send` and return its stdout verbatim as your result. run-all relays that body as-is and should never re-compose it. An override instruction will be in the prompt when run-all is the caller.

---

## Sub-agent JSON contract (every sub-agent returns exactly this)

Each sub-agent's FINAL message is ONE JSON object and nothing else: no prose and no code fence.

```json
{
  "sub_agent": "feedback-outreach-drafter | feedback-outreach-scanner | pass-note-drafter",
  "status": "ok | error",
  "error": "<one line — required iff status = error>",
  "events": [
    {
      "entity_name": "Fair",
      "entity_url": "https://app.notion.com/p/<opp id>",
      "entity_id": "<opp page id>",
      "person_name": "Paul Drinkwater",
      "person_url": "https://app.notion.com/p/<person id>",
      "transition": "Feedback Outreach: → Drafted",
      "outcome": "wrote | observed_already_in_state | skipped | cleanup",
      "detail": "<short free text>"
    }
  ],
  "pass_note_queue": [{"opp_name": "Radical", "opp_url": "https://…"}]
}
```

- `events` uses the same record shape as the reconciliation manifests in `~/.claude/scheduled-tasks/reconciliation/inbox/diligence-*.jsonl`. The harness fixtures are real lines from those files. Required: `entity_name`, `transition`, `outcome`. `person_name` and `person_url` are required on every `Feedback …` event, because an event without a resolved person is dropped. An empty `events` list is valid.
- `transition` values: `Feedback Outreach: → Drafted` (drafter), `Feedback Outreach: → Sent`, `Feedback Reply: → Substantive|Deferral` (scanner), `Pass Note: → Drafted`, `Status: Pass Note Pending → Pass (Met)` (pass-note-drafter). `->` is accepted for `→`.
- `outcome`: `wrote` = this run did it. `observed_already_in_state` = done already and seen within the window. `skipped` or `skipped_*` = no action, and it is never rendered. `cleanup` = maintenance.
- `pass_note_queue` is for the pass-note-drafter only: the Opps still in `Status = Pass Note Pending` after the run. Return `[]` when none are left.
- If the sub-agent failed, return `{"sub_agent": …, "status": "error", "error": "<why>", "events": [<whatever it did manage>]}`. A sub-agent that returns nothing still gets its own `⚠` line.

---

## Execution Steps

### Step 1: Feedback Outreach Drafter

Spawn a sub-agent with `max_turns: 30`:

```
You are running the "Feedback Outreach Drafter" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any notifications of any kind. You are being called from the Diligence Agent orchestrator, which handles all notifications centrally. Just return your results.

STEP 1: Use the Glob tool with pattern **/feedback-outreach-drafter/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute in SCHEDULED SCAN MODE — scan the Opportunities DB 📣 Pending Feedback relation for newly added people, check whether outreach emails have been sent (Gmail lookback: 24h, or 72h if today is Monday), and draft Gmail outreach notes for any outstanding.
STEP 3: Your FINAL message is ONE JSON object per the diligence-agent "Sub-agent JSON contract" with "sub_agent": "feedback-outreach-drafter" — one event per (person, opportunity) you acted on or checked, transition "Feedback Outreach: → Drafted". No prose.
```

Wait for completion. Save its JSON to `/tmp/diligence-feedback-outreach-drafter.json`.

### Step 2: Feedback Outreach Scanner

Spawn a sub-agent with `max_turns: 25`:

```
You are running the "Feedback Outreach Scanner" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any notifications of any kind. You are being called from the Diligence Agent orchestrator, which handles all notifications centrally. Just return your results.

STEP 1: Use the Glob tool with pattern **/feedback-outreach-scanner/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file (sent scan + reply scan). Gmail lookback override: newer_than:24h, or newer_than:72h if today is Monday.
STEP 3: Your FINAL message is ONE JSON object per the diligence-agent "Sub-agent JSON contract" with "sub_agent": "feedback-outreach-scanner" — transitions "Feedback Outreach: → Sent" / "Feedback Reply: → Substantive|Deferral". No prose.
```

Wait for completion. Save its JSON to `/tmp/diligence-feedback-outreach-scanner.json`.

### Step 3: Pass Note Drafter

Spawn a sub-agent with `max_turns: 30`:

```
You are running the "Pass Note Drafter" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any Slack/iMessage/Beeper notifications. You are being called from the Diligence Agent orchestrator, which handles all notifications centrally. Just return your results.

STEP 1: Use the Glob tool with pattern **/pass-note-drafter/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file (Gmail sent-check lookback: 24h, or 72h if today is Monday).
STEP 3: Your FINAL message is ONE JSON object per the diligence-agent "Sub-agent JSON contract" with "sub_agent": "pass-note-drafter" — transitions "Pass Note: → Drafted" / "Status: Pass Note Pending → Pass (Met)", plus "pass_note_queue" = Opps still Pass Note Pending after the run. No prose.
```

Wait for completion. Save its JSON to `/tmp/diligence-pass-note-drafter.json`.

### Step 4: Errors

If a sub-agent fails, times out, or returns something that isn't JSON, write `{"sub_agent": "<key>", "status": "error", "error": "<one line>", "events": []}` to its file and continue to the next sub-agent. One failure must not block the others.

### Step 5: Consolidated alert (as code — 2026-10-04)

```bash
python3 ~/.claude/skills/diligence-agent/consolidate_alert.py --send \
  /tmp/diligence-feedback-outreach-drafter.json \
  /tmp/diligence-feedback-outreach-scanner.json \
  /tmp/diligence-pass-note-drafter.json
```

(Via run-all: drop `--send` and return stdout verbatim.)

**Exit handling:**
- `0`: the body was printed (and sent, with `--send`). Done.
- `2` (REFUSE): one envelope broke the contract. stderr names it. Do NOT hand-compose the alert. Send one plain line instead: `printf '🔍 <u>**Diligence Sweep: Failed**</u> · %s\n\n⚠ consolidate_alert refused: <stderr reason>\n' "$(date +%F)" | ~/.claude/skills/send-alert/send.sh`.
- `3`: `send.sh` failed. The body is still on stdout. Retry `send.sh` once with that body, then stop.

**What the code does.** The rules below are WHY it behaves this way. Do not apply them by hand.

- **Group by opportunity, not by sub-agent.** Tom reads the alert per deal. Under each **bold** Opp there is at most one line per kind: `• Feedback outreach:`, `• Feedback scanner:`, `• Pass note:` (and `Maintenance` / `Other` for cleanup or unknown transitions, which are never silently dropped). Opps are sorted A→Z and people are linked. Names are bolded with standard markdown `**…**`. Slack single asterisks render as italic.
- **No feedback line without a resolved person.** On 2026-08-11, AgentBay, Coverbase, Decisionly, Fair and Rengo portcos surfaced as blank-person feedback items. Those were artifacts of a stale OR view, not action items. The scanner now reads a Pending-Feedback-only view; the code drop is the backstop.
- **Only lines with activity print.** There are no `n/a` lines and no empty Opp blocks. `skipped*` events never render: they repeat daily (the Factir / Justin Sherlock skip was logged every run from May 19 to 29). The **Pass Note Queue** block appears only when the queue is non-empty.
- **Steady state.** When nothing rendered and nothing errored, the body is the single line `Steady state — 0 writes across all 3 diligence sub-scanners.`
- **Errors first.** Each errored or missing sub-agent gets a `⚠ <Sub-agent>: <reason>` line ahead of the Opp blocks, per the alert convention's action-required-first rule. Any events it did return still render.
- **Header** follows the shared alert convention: `🔍 <u>**Diligence Sweep: Feedback + Pass Notes**</u> · YYYY-MM-DD`, then a blank line. 🔍 is the diligence domain emoji, the headline is Title Case with a colon, and the ISO date is there because a sweep is a digest. Output passes `send-alert/alert_lint.py`.

Example (real 2026-07-31 Fair records and the 2026-09-29 Radical pass note):

```
🔍 <u>**Diligence Sweep: Feedback + Pass Notes**</u> · 2026-07-31

**[Fair](…)**
• Feedback outreach: [Paul Drinkwater](…) – already sent; [Anthony Chen](…) – draft created
• Feedback scanner: [Paul Drinkwater](…) – note already in place; [Anthony Chen](…) – note created

**[Radical](…)**
• Pass note: draft created

**Pass Note Queue**
• [Radical](…)
```

Harness: `python3 ~/.claude/skills/diligence-agent/tests/test_consolidate_alert.py`. Fixtures are in `tests/fixtures/real_manifests.jsonl`, censused from the reconciliation inbox. Add every new incident as a case there before changing the code.

---

## Canonical Skill File Discovery

| Sub-Agent | Glob Pattern |
|---|---|
| Feedback Outreach Drafter | `**/feedback-outreach-drafter/SKILL.md` |
| Feedback Outreach Scanner | `**/feedback-outreach-scanner/SKILL.md` |
| Pass Note Drafter | `**/pass-note-drafter/SKILL.md` |
