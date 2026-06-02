---
name: run-all
description: "Run all of Tom's scheduled tasks in a single session without overflowing the context window. Each task is spawned as an independent sub-agent using the Task tool with isolation, so each gets its own context budget. Only a brief summary comes back to the parent orchestrator. Trigger this skill whenever the user says 'run all tasks', 'run all my scheduled tasks', 'run everything', 'execute all tasks', 'run all scanners', 'morning scan', 'evening scan', or any variant requesting batch execution of scheduled tasks. Also trigger when the user asks to 'run my tasks now' or 'kick off all jobs'."
---

# Run-All Task Orchestrator

Execute all of Tom's grouped agents sequentially, each in its own isolated sub-agent context. This prevents context window overflow that occurs when running all tasks in a single session.

## Architecture

Each agent runs as an independent sub-agent via the `Task` tool with `subagent_type: "general-purpose"`. The sub-agent discovers and reads its own SKILL.md at runtime using Glob patterns — the orchestrator never pre-reads or inlines skill file contents. The parent orchestrator collects summaries and presents a consolidated report.

**Single Source of Truth**: Each agent's behavior is defined by its standalone SKILL.md file. This orchestrator does NOT duplicate or inline those instructions. Any updates to a skill's SKILL.md automatically propagate to run-all without maintenance.

**Dynamic Path Discovery**: Session paths change each run, so skill files are discovered at runtime using Glob patterns (e.g., `**/pipeline-agent/SKILL.md`) — never hardcoded paths.

## Model Routing

All sub-agents default to **Sonnet** (`model: "sonnet"`). These are scan-and-write workflows — read an inbox or Notion view, detect state changes, write the diff back. Opus is overkill and roughly 3× slower for this shape of work.

| Sub-agent | Model | Rationale |
|---|---|---|
| Intro Qualified / Outreach / Draft / Resolution | `sonnet` | Scan Gmail/iMessage/Notion, classify, write back |
| Pipeline Agent | `sonnet` | 4-stage triage but mostly categorization + Notion writes |
| Portfolio Agent | `sonnet` | Gmail scan + PDF filing |
| Diligence Agent | `sonnet` | 3 scan-and-write sub-workflows |

**Escalation rule:** if a sub-agent returns incomplete results or struggles with reasoning (e.g., a Pipeline triage misclassifies, an intro resolution misreads a deferral), bump that specific sub-agent to `model: "opus"` on the next run and leave the others on Sonnet. Do NOT default the whole run to Opus — that's what we just moved away from.

The **parent orchestrator** (this skill) stays on whatever model the user's session is running, because it owns the Research Agent inline (which benefits from stronger reasoning over web results) and composes the Slack alerts.

## Agent Groups

There are 5 master agents, each encompassing one or more task workflows:

1. **Intro Agent** — Qualified Scanner → Outreach Scanner → Draft Agent → Resolution Scanner (4 sub-skills, each spawned as its own sub-agent to avoid context overflow)
2. **Pipeline Agent** — Deal Opportunity Scanner → Qualified Triage → Outreach Triage → Connected Triage
3. **Research Agent** — Daily Investor Letters (web scan for published letters from investment firms)
4. **Portfolio Agent** — Investor Update Scanner (Gmail scan for portfolio company updates)
5. **Diligence Agent** — Feedback Outreach Drafter → Feedback Outreach Scanner → Pass Note Drafter

> **⚠️ Disambiguation**: "Investor updates" (Portfolio Agent) = emails from portfolio company founders. "Investor letters" (Research Agent) = published letters/memos from other investment firms (Berkshire, Oaktree, Pershing Square, etc.). Never confuse the two.

## Notification Cadence — Per Grouped Agent

**When run via run-all, no sub-agent sends its own alert.** Instead, the orchestrator sends ONE Slack alert as each of the 5 grouped agents completes — not one batched alert at the end. Tom wants visibility as work progresses.

- **Intro Agent**: After all 4 sub-scanners (Qualified / Outreach / Draft / Resolution) complete, compose ONE grouped alert using the **Shared Intro Alert Format** in `intro-agent/SKILL.md` (organized by person, bold names, `✨ moved` markers).
- **Pipeline Agent**: Use the format in `pipeline-agent/SKILL.md` (organized by opportunity, only opportunities that moved or flagged — never dump the full roster).
- **Research Agent**: Use the format in `research-agent/SKILL.md` (unchanged — tier-grouped, one bullet per letter with URL).
- **Portfolio Agent**: Use the format in `investor-update/SKILL.md` (organized by company, split into Portfolio / Non-Portfolio sections).
- **Diligence Agent**: Use the format in `diligence-agent/SKILL.md` (organized by opportunity).

Each agent's prompt includes an override instruction to suppress its own notification. The orchestrator composes the Slack message from the sub-agent's returned structured summary.

## Canonical Skill File Discovery

Session paths change each run, so skill files are discovered at runtime using Glob patterns — never hardcoded.

| Agent | Glob Pattern |
|---|---|
| Intro Agent (Qualified) | `**/intro-agent/SKILL.md` |
| Intro Agent (Outreach) | `**/intro-outreach-agent/SKILL.md` |
| Intro Agent (Draft) | `**/intro-draft-agent/SKILL.md` |
| Intro Agent (Resolution) | `**/intro-resolution-agent/SKILL.md` |
| Pipeline Agent | `**/pipeline-agent/SKILL.md` |
| Research Agent | `**/research-agent/SKILL.md` |
| Portfolio Agent | `**/investor-update/SKILL.md` |
| Diligence Agent | `**/diligence-agent/SKILL.md` |

## Execution Steps

### Step 0: Announce Start

Tell the user: "Running all 5 grouped agents (Intro, Pipeline, Research, Portfolio, Diligence). Alerts fire to Slack as each grouped agent completes — not batched at the end."

Do NOT pre-read any SKILL.md files. Each sub-agent discovers and reads its own skill file at runtime.

### Execution Order

- **Intro Agent sub-scanners** (Qualified → Outreach → Draft → Resolution) run **sequentially** — they operate on stages of the same pipeline and have implicit ordering dependencies.
- **Pipeline / Portfolio / Diligence** run in **parallel** (independent domains, independent Notion writes). Launch them as background sub-agents in a single batch.
- **Research Agent** runs **inline from the parent session** — NOT as a background sub-agent. Sub-agents spawned by the Agent tool lack WebSearch/WebFetch permissions, and the Research Agent skill is fundamentally dependent on web access. Running inline avoids the permission gap.

### Step 1: Intro Agent (4 Independent Sub-Agents)

The Intro Agent encompasses 4 sub-workflows. To avoid context overflow, each sub-workflow runs in its own independent sub-agent — they are NOT combined into a single sub-agent. Run them sequentially.

#### Step 1a: Qualified Scanner

Spawn a sub-agent with `model: "sonnet"`, `max_turns: 25`:

```
You are running the "Intro Agent — Qualified Scanner" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any messages to Tom — no iMessages (send_imessage), no Beeper messages (send_message), no notifications of any kind. You are being called from the run-all orchestrator, which handles all notifications centrally. Just return your results.

MCP FALLBACK (CRITICAL): Notion (`mcp__claude_ai_Notion__*`) and Gmail (`mcp__claude_ai_Gmail__*`) MCP tools are often NOT available in Task-spawned sub-agent sessions. If a tool call returns "tool not found" or those tools aren't listed, do NOT abort the workflow with "MCP unavailable" — shell out via Bash to `claude --print "<focused task prompt>"` for that single operation. The spawned subprocess has fresh MCP access and will return the structured result on stdout for you to parse. Apply this fallback to ALL roster reads, Opportunity/People lookups, lifecycle field writes, and Gmail inbox/sent scans. Background: the 2026-05-27 run aborted with "Notion MCP unavailable" across all 3 sweep sub-agents — this fallback exists specifically to rescue that case.

STEP 1: Use the Glob tool with pattern **/intro-agent/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file.
STEP 3: Return structured data (one entry per person touched), plus totals:

### Qualified Scanner
- **New intros logged**: [count]
  - [Person Name] → [Opportunity] — [detection context, e.g. "detected in forwardable email from founder, 2026-04-17"]
- **Duplicates skipped**: [count]
  - [Person Name] → [Opportunity] — [which lifecycle field they're already in + date/evidence]
- **Errors**: [any issues]
```

Wait for completion. Record its summary.

#### Step 1b: Outreach Scanner

Spawn a sub-agent with `model: "sonnet"`, `max_turns: 25`:

```
You are running the "Intro Agent — Outreach Scanner" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any messages to Tom — no iMessages (send_imessage), no Beeper messages (send_message), no notifications of any kind. You are being called from the run-all orchestrator, which handles all notifications centrally. Just return your results.

MCP FALLBACK (CRITICAL): Notion (`mcp__claude_ai_Notion__*`) and Gmail (`mcp__claude_ai_Gmail__*`) MCP tools are often NOT available in Task-spawned sub-agent sessions. If a tool call returns "tool not found" or those tools aren't listed, do NOT abort the workflow with "MCP unavailable" — shell out via Bash to `claude --print "<focused task prompt>"` for that single operation. The spawned subprocess has fresh MCP access and will return the structured result on stdout for you to parse. Apply this fallback to ALL roster reads, Opportunity/People lookups, lifecycle field writes, and Gmail inbox/sent scans. Background: the 2026-05-27 run aborted with "Notion MCP unavailable" across all 3 sweep sub-agents — this fallback exists specifically to rescue that case.

STEP 1: Use the Glob tool with pattern **/intro-outreach-agent/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file.
STEP 3: Return structured data (one entry per person touched), plus totals:

### Outreach Scanner
- **Moved to Outreach**: [count]
  - [Person Name] → [Opportunity] — [detection method + date]
- **Multi-step jumps** (Qualified → Made/Declined, skipping Outreach): [count]
  - [Person Name] → [Opportunity] → [destination stage] — [evidence]
- **Still in Qualified (unchanged)**: [count]
  - [Person Name] → [Opportunity] — [context, e.g. "21d in stage, no outreach detected"]
- **Errors**: [any issues]
```

Wait for completion. Record its summary.

#### Step 1c: Draft Agent

Spawn a sub-agent with `model: "sonnet"`, `max_turns: 25`:

```
You are running the "Intro Agent — Draft Agent" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any messages to Tom — no iMessages (send_imessage), no Beeper messages (send_message), no notifications of any kind. You are being called from the run-all orchestrator, which handles all notifications centrally. Just return your results.

MCP FALLBACK (CRITICAL): Notion (`mcp__claude_ai_Notion__*`) and Gmail (`mcp__claude_ai_Gmail__*`) MCP tools are often NOT available in Task-spawned sub-agent sessions. If a tool call returns "tool not found" or those tools aren't listed, do NOT abort the workflow with "MCP unavailable" — shell out via Bash to `claude --print "<focused task prompt>"` for that single operation. The spawned subprocess has fresh MCP access and will return the structured result on stdout for you to parse. Apply this fallback to ALL roster reads, Opportunity/People lookups, lifecycle field writes, Gmail inbox/sent scans, AND Gmail draft creation. Background: the 2026-05-27 run aborted with "Notion MCP unavailable" across all 3 sweep sub-agents — this fallback exists specifically to rescue that case.

STEP 1: Use the Glob tool with pattern **/intro-draft-agent/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file.
STEP 3: Return structured data (one entry per person touched), plus totals:

### Draft Agent
- **Drafts created**: [count]
  - [Person Name] → [Opportunity] — [subject line, recipients]
- **Skipped (draft/sent exists)**: [count]
  - [Person Name] → [Opportunity] — [which: draft already queued / intro email already sent, date]
- **No opt-in detected**: [count]
  - [Person Name] → [Opportunity] — [context, e.g. "no reply in inbox"]
- **Errors**: [any issues]
```

Wait for completion. Record its summary.

#### Step 1d: Resolution Scanner

Spawn a sub-agent with `model: "sonnet"`, `max_turns: 25`:

```
You are running the "Intro Agent — Resolution Scanner" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any messages to Tom — no iMessages (send_imessage), no Beeper messages (send_message), no notifications of any kind. You are being called from the run-all orchestrator, which handles all notifications centrally. Just return your results.

MCP FALLBACK (CRITICAL): Notion (`mcp__claude_ai_Notion__*`) and Gmail (`mcp__claude_ai_Gmail__*`) MCP tools are often NOT available in Task-spawned sub-agent sessions. If a tool call returns "tool not found" or those tools aren't listed, do NOT abort the workflow with "MCP unavailable" — shell out via Bash to `claude --print "<focused task prompt>"` for that single operation. The spawned subprocess has fresh MCP access and will return the structured result on stdout for you to parse. Apply this fallback to ALL roster reads, Opportunity/People lookups, lifecycle field writes, and Gmail inbox/sent scans. Background: the 2026-05-27 run aborted with "Notion MCP unavailable" across all 3 sweep sub-agents — this fallback exists specifically to rescue that case.

STEP 1: Use the Glob tool with pattern **/intro-resolution-agent/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file.
STEP 3: Return structured data (one entry per person touched), plus totals:

### Resolution Scanner
- **Moved to Made**: [count]
  - [Person Name] → [Opportunity] (from [Outreach / Qualified]) — [evidence + date, note if multi-stage jump]
- **Moved to Declined/NR**: [count]
  - [Person Name] → [Opportunity] (from [Outreach / Qualified]) — [reason + date]
- **Kept in Outreach (soft deferral)**: [count]
  - [Person Name] → [Opportunity] — [context, e.g. "replied 'ping me post-Series A'"]
- **Still in Outreach (no resolution)**: [count]
  - [Person Name] → [Opportunity] — [context, e.g. "no reply to follow-up sent 2026-04-14"]
- **Errors**: [any issues]
```

Wait for completion. Record its summary.

#### Step 1e: Compose & Send Intro Alert

After all 4 Intro sub-scanners complete, read `intro-agent/SKILL.md` for the Shared Intro Alert Format. Merge data across the 4 sub-scanners by person, organize into stage sections (Qualified / Outreach / Made / Declined), bold person names with single asterisks, tag transitions with `✨ moved <From> → <To>`, and send ONE Slack message via the `send-alert` skill's channel config. If all 4 sub-scanners produced zero moves and zero unchanged entries worth surfacing, send a single-line body: `Steady state — 0 intro transitions, rosters unchanged.`

### Step 2: Pipeline, Portfolio, Diligence (Parallel Sub-Agents)

Launch all three as **background sub-agents in a single batch** (one assistant message with 3 Agent tool calls, each with `run_in_background: true`). They're independent domains with no shared writes. As each completes, compose and send its Slack alert immediately — do not wait for the other two.

#### Step 2a: Pipeline Agent

Spawn with `model: "sonnet"`, `max_turns: 30`, `run_in_background: true`:

```
You are running the "Pipeline Agent" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any messages to Tom — no iMessages, no Beeper messages, no Slack alerts, no notifications of any kind. You are being called from the run-all orchestrator, which handles all notifications centrally. Just return your results.

MCP FALLBACK (CRITICAL): Notion (`mcp__claude_ai_Notion__*`) and Gmail (`mcp__claude_ai_Gmail__*`) MCP tools are often NOT available in Task-spawned sub-agent sessions. If a tool call returns "tool not found" or those tools aren't listed, do NOT abort the workflow with "MCP unavailable" — shell out via Bash to `claude --print "<focused task prompt>"` for that single operation. The spawned subprocess has fresh MCP access and will return the structured result on stdout for you to parse. Apply this fallback to ALL Notion roster reads, Opportunity lookups, status writes, materials linking, and Gmail inbox/sent scans. Background: the 2026-05-27 intro-agent run aborted with "Notion MCP unavailable" across all 3 sweep sub-agents — this fallback exists specifically to rescue that case.

STEP 1: Use the Glob tool with pattern **/pipeline-agent/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file (all 4 pipeline stages: Deal Scanner, Qualified Triage, Outreach Triage, Connected Triage).
STEP 3: Return structured data organized by opportunity:

### Pipeline Agent
- **New deals this run**: [count]
  - [Opportunity] — [source/detection, logged stage & status]
- **Status changes**: [count]
  - [Opportunity] — moved [From Status] → [To Status] — [trigger/evidence]
- **Renamed**: [list [Old → New] or "none"]
- **Materials linked**: [count]
  - [Opportunity] — [brief description] (mode: full/lightweight)
- **Flagged for manual review**: [count]
  - [Opportunity] — [what's ambiguous]
- **Internal roster totals** (NOT for alert): Qualified: [N], Outreach: [N], Tracking scanned: [N]
- **Errors**: [any issues]
```

When this returns, read `pipeline-agent/SKILL.md` for its Notification Summary Format. Compose the Slack message (organized by opportunity, bolded names with single asterisks, `✨ moved` markers, omit empty sections). If all 5 sections are empty, send a single-line body: `Steady state — 0 writes across all 4 pipeline stages.` Send via `send-alert`.

#### Step 2b: Portfolio Agent

Spawn with `model: "sonnet"`, `max_turns: 20`, `run_in_background: true`:

```
You are running the "Portfolio Agent" (investor-update skill, scheduled scan mode) for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any messages to Tom — no iMessages, no Beeper messages, no Slack alerts, no notifications of any kind. You are being called from the run-all orchestrator, which handles all notifications centrally. Just return your results.

MCP FALLBACK (CRITICAL): Notion (`mcp__claude_ai_Notion__*`) and Gmail (`mcp__claude_ai_Gmail__*`) MCP tools are often NOT available in Task-spawned sub-agent sessions. If a tool call returns "tool not found" or those tools aren't listed, do NOT abort the workflow with "MCP unavailable" — shell out via Bash to `claude --print "<focused task prompt>"` for that single operation. The spawned subprocess has fresh MCP access and will return the structured result on stdout for you to parse. Apply this fallback to ALL Gmail inbox scans, Notion Opportunity lookups, Notion page creation, and Notion file-property writes. Background: the 2026-05-27 intro-agent run aborted with "Notion MCP unavailable" across all 3 sweep sub-agents — this fallback exists specifically to rescue that case.

STEP 1: Use the Glob tool with pattern **/investor-update/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file (Gmail scan, PDF archiving, Notion page creation, PDF embedding).
STEP 3: Return structured data organized by company, split into Portfolio vs Non-Portfolio:

### Portfolio Agent
- **Portfolio updates processed**: [count]
  - [Company] — "[subject]" — [PDF source: original/email-converted] — [Notion page link]
- **Non-Portfolio (filtered)**: [count]
  - [Company / Sender] — "[subject]" — [reason filtered, e.g. "Status: Scheduled → saved as Diligence Material"; or "Not in Opportunities DB — personal newsletter"]
- **Needs manual review**: [count]
  - [Sender/Company] — [why ambiguous]
- **Errors**: [any issues]
```

When this returns, read `investor-update/SKILL.md` for its Alert format. Compose the Slack message (Portfolio / Non-Portfolio sections, bolded company names with single asterisks, one-line reason per Non-Portfolio entry). Send via `send-alert`.

#### Step 2c: Diligence Agent

Spawn with `model: "sonnet"`, `max_turns: 30`, `run_in_background: true`:

```
You are running the "Diligence Agent" for Tom Seo (tom@invertedcap.com), an early-stage VC investor.

IMPORTANT OVERRIDE: Do NOT send any messages to Tom — no iMessages, no Beeper messages, no Slack alerts, no notifications of any kind. You are being called from the run-all orchestrator, which handles all notifications centrally. Just return your results.

MCP FALLBACK (CRITICAL): Notion (`mcp__claude_ai_Notion__*`) and Gmail (`mcp__claude_ai_Gmail__*`) MCP tools are often NOT available in Task-spawned sub-agent sessions. If a tool call returns "tool not found" or those tools aren't listed, do NOT abort the workflow with "MCP unavailable" — shell out via Bash to `claude --print "<focused task prompt>"` for that single operation. The spawned subprocess has fresh MCP access and will return the structured result on stdout for you to parse. Apply this fallback to ALL Notion roster reads (Pending Feedback, Pass Note Pending), Opportunity lookups, Gmail sent/inbox scans, and Gmail draft creation. Background: the 2026-05-27 intro-agent run aborted with "Notion MCP unavailable" across all 3 sweep sub-agents — this fallback exists specifically to rescue that case.

STEP 1: Use the Glob tool with pattern **/diligence-agent/SKILL.md to find the canonical skill file. Read it with the Read tool.
STEP 2: Execute the full workflow described in that skill file (Feedback Outreach Drafter → Feedback Outreach Scanner → Pass Note Drafter).
STEP 3: Return structured data organized by opportunity:

### Diligence Agent
- **Per-opportunity activity**:
  - [Opportunity]
    - Feedback outreach: [draft created for Person / already sent DATE / no new Pending Feedback entries]
    - Feedback scanner: [reply logged from Person / no reply in 12h window / note already in place]
    - Pass note: [draft created / already sent / not in Pass Note Pending queue]
- **Pass Note Queue state**: [list opportunities currently with Status = "Pass Note Pending", or "empty"]
- **Errors**: [any issues]
```

When this returns, read `diligence-agent/SKILL.md` for its Alert format. Compose the Slack message (organized by opportunity, bolded names with single asterisks, one line per relevant sub-scanner per opp). If all 3 sub-scanners produced zero activity, send a single-line body: `Steady state — 0 writes across all 3 diligence sub-scanners.` Send via `send-alert`.

### Step 3: Research Agent (Inline — Not a Sub-Agent)

Run the Research Agent **inline from the parent session** — NOT as a background sub-agent. Sub-agents lack WebSearch/WebFetch permissions, which the Research Agent fundamentally requires.

1. Use the Glob tool with pattern `**/research-agent/SKILL.md` to find the canonical skill file. Read it.
2. Execute the Tier 1 catch-all queries and Tier 2 discovery queries directly via WebSearch/WebFetch from the parent session. Filter strictly to the past 1–2 days.
3. Compose the Slack alert using the output format defined in `research-agent/SKILL.md` (tier-grouped, one bullet per letter with URL, bold firm names with single asterisks). Send via `send-alert`.

### Step 4: Final Chat Report

After all 5 grouped agents have completed and their Slack alerts have been sent, present a single consolidated chat report summarizing all 5 sections:

```
# Daily Task Run — [Today's Date]

## 🤝 Intro Agent
[Per-stage breakdown from the 4 sub-scanners, with any movement called out]

## 🔄 Pipeline Agent
[Stage breakdown; flag any opportunities needing manual review]

## 🔬 Research Agent
[Tier 1 / Tier 2 findings]

## 📬 Portfolio Agent
[Portfolio / Non-Portfolio split]

## 🔍 Diligence Agent
[Per-opportunity status across 3 sub-scanners]

---
**Manual follow-ups**: [any items flagged for review]
**Memory updates saved**: [any new feedback/project memories from this run]
```

This chat report is for the user's in-session review. The Slack alerts already went out incrementally per-agent.

## Error Handling

- If a sub-agent fails or times out, record the error and move on to the next agent. Don't let one failure block the rest.
- If the Task tool is unavailable, fall back to running tasks inline but warn the user about potential context overflow for the full set.
- If the Research Agent sub-agent ever gets spawned by mistake (legacy scheduled-task invocation) and returns "WebSearch/WebFetch denied," fall back to inline execution in the parent session — do not retry the sub-agent.
- Set `max_turns: 25` for each Intro sub-agent, `max_turns: 30` for Pipeline Agent and Diligence Agent, `max_turns: 20` for Portfolio Agent.

## Notes

- Each sub-agent discovers and reads its own SKILL.md at runtime via Glob. The orchestrator never pre-reads or inlines skill file contents. This keeps the orchestrator prompt lightweight and prevents context overflow.
- The Intro Agent is split into 4 independent sub-agents (one per sub-workflow) because the combined intro skills (~1,133 lines) overflow a single context window. This matches the architecture used by the `intro-agent-scan` scheduled task.
- Notion semantic search doesn't filter by status perfectly — sub-agents should fetch individual pages to verify status before making changes.
- iMessages MCP requires Full Disk Access and may fail. Sub-agents should handle this gracefully.
- The `notion-query-database-view` tool has known issues with URL format validation. Sub-agents should prefer `notion-search` with `data_source_url` for finding opportunities by status.
- **No sub-agent ever sends its own Slack alert when called from run-all. The orchestrator composes and sends all 5 alerts itself — one per grouped agent, fired incrementally as each completes.**
