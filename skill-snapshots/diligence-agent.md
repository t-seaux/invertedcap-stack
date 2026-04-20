---
name: diligence-agent
description: >
  Orchestrates three SCHEDULED diligence sub-agents: (1) Feedback Outreach Drafter — scans the Feedback relation field for new entries and drafts outreach emails for any not yet sent or drafted; (2) Feedback Outreach Scanner — scans sent mail and inbox for feedback outreach activity, logs Notion notes; (3) Pass Note Drafter — drafts investor pass notes for Pass Note Pending opportunities. Runs twice daily at 8am and 6pm ET. Trigger when the user says "run diligence agent", "diligence scan", "check feedback replies", "run pass notes", or any variant requesting scheduled diligence tasks. Registered as Agent 5 in run-all.
---

# Diligence Agent

Orchestrates two diligence sub-agents sequentially, each in its own isolated context via the Task tool. Follows the same architecture as the run-all orchestrator: each sub-agent discovers and reads its own SKILL.md at runtime — this orchestrator never pre-reads or inlines skill file contents.

## Sub-Agents

This agent runs three **scheduled** sub-agents sequentially:

1. **Feedback Outreach Drafter** — scans the `📣 Pending Feedback` relation field across all opportunities for people added in the past 24 hours; checks whether an outreach email has already been sent to each; drafts Gmail outreach notes for any that haven't been sent yet
2. **Feedback Outreach Scanner** — scans Gmail sent mail and inbox for feedback outreach activity; creates and updates per-person Notion feedback notes
3. **Pass Note Drafter** — queries Notion for Pass Note Pending opportunities; drafts investor pass notes as Gmail drafts

> **Note:** The Feedback Outreach Drafter can also be triggered manually (e.g. "draft feedback outreach note for X on Y"). When triggered manually, it skips the Notion scan and goes directly to drafting for the named people.

## Notification Behavior

**When run standalone** (not via run-all): after both sub-agents complete, read the `send-alert` skill (discover via Glob pattern `**/send-alert/SKILL.md`) and send a single consolidated Signal alert.

**When run via run-all**: do NOT send any notifications — the run-all orchestrator handles all alerts centrally. An override instruction will be present in the prompt when called from run-all.

---

## Execution Steps

### Step 1: Feedback Outreach Drafter

Spawn a sub-agent with `max_turns: 30`:

```
You are running the "Feedback Outreach Drafter" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any notifications of any kind. You are being called from the Diligence Agent orchestrator, which handles all notifications centrally. Just return your results.

STEP 1: Use the Glob tool with pattern **/feedback-outreach-drafter/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute in SCHEDULED SCAN MODE — scan the Opportunities DB 📣 Pending Feedback relation for people added in the past 24 hours, check whether outreach emails have been sent, and draft Gmail outreach notes for any outstanding.
STEP 3: Return ONLY the following summary:

### Feedback Outreach Drafter
- **Drafts created**: [count] — [list: Person → Opportunity]
- **Already sent / skipped**: [count] — [list: Person → Opportunity]
- **Nothing to draft**: [true/false]
- **Errors**: [any issues]
```

Wait for completion. Record its summary.

### Step 2: Feedback Outreach Scanner

Spawn a sub-agent with `max_turns: 25`:

```
You are running the "Feedback Outreach Scanner" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any notifications of any kind. You are being called from the Diligence Agent orchestrator, which handles all notifications centrally. Just return your results.

STEP 1: Use the Glob tool with pattern **/feedback-outreach-scanner/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file (sent scan + reply scan).
STEP 3: Return ONLY the following summary:

### Feedback Outreach Scanner
- **New notes created**: [count] — [list: Person (Company) → Opportunity]
- **Replies logged**: [count] — [list: Person → Opportunity]
- **Already existed / skipped**: [count]
- **Errors**: [any issues]
```

Wait for completion. Record its summary.

### Step 3: Pass Note Drafter

Spawn a sub-agent with `max_turns: 30`:

```
You are running the "Pass Note Drafter" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any Signal/Beeper notifications. You are being called from the Diligence Agent orchestrator, which handles all notifications centrally. Just return your results.

STEP 1: Use the Glob tool with pattern **/pass-note-drafter/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file.
STEP 3: Return ONLY the following summary:

### Pass Note Drafter
- **Drafts created**: [count] — [list: Company → founder email]
- **Already sent (moved to Pass Met)**: [count] — [list: Company]
- **Draft already exists / skipped**: [count] — [list: Company]
- **Nothing pending**: [true/false]
- **Errors**: [any issues]
```

Wait for completion. Record its summary.

### Step 4: Consolidated Report

Compile both summaries and present to the user:

```
# Diligence Agent — [Date] [8am / 6pm] Run

## Feedback Outreach Scanner
[Sub-agent summary]

## Pass Note Drafter
[Sub-agent summary]
```

### Step 5: Alert (standalone mode only)

If NOT running via run-all, read the `send-alert` skill and send. Organize by **opportunity** (not by sub-scanner). Bold opportunity names using **standard markdown double asterisks** (e.g. `**Clusia**`). The `mcp__claude_ai_Slack__slack_send_message` tool renders standard markdown — single asterisks produce italic, not bold. Under each opportunity, list only the activity for that opportunity across all 3 sub-scanners (feedback drafts, feedback replies, pass-note status).

```
🔍 DILIGENCE — YYYY-MM-DD

**<Opportunity>**
• Feedback outreach: <Person Name> — <draft created / already sent DATE / no new Pending Feedback entries>
• Feedback scanner: <reply logged from Person / no reply in 12h window / note already in place>
• Pass note: <draft created / already sent / n/a — not in Pass Note Pending queue>

**<Opportunity>**
• ...

**Pass Note Queue**
• _empty_ (if no opportunities currently have Status = "Pass Note Pending")
• OR list each Pass-Note-Pending opportunity above with its pass-note status
```

Rules:
- **Bold the opportunity name** with double asterisks (standard markdown). Never use Slack mrkdwn single-asterisks — they render as italic, not bold.
- Only include the sub-scanner lines that apply to that opportunity. If a Clusia-only run produced zero pass-note activity on Clusia, omit the pass-note line rather than writing `n/a`.
- If all 3 sub-scanners produced zero activity, send a single-line body: `Steady state — 0 writes across all 3 diligence sub-scanners.`
- The header uses the 🔍 emoji, an em dash (—), and ISO date format. Example: `🔍 DILIGENCE — 2026-03-06`.

---

## Error Handling

If a sub-agent fails or times out, record the error and continue to the next. Do not let one failure block the other.

---

## Canonical Skill File Discovery

| Sub-Agent | Glob Pattern |
|---|---|
| Feedback Outreach Drafter | `**/feedback-outreach-drafter/SKILL.md` |
| Feedback Outreach Scanner | `**/feedback-outreach-scanner/SKILL.md` |
| Pass Note Drafter | `**/pass-note-drafter/SKILL.md` |
