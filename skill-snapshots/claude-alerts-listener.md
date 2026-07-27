---
name: claude-alerts-listener
description: "Processes thread replies in #claude-alerts as feedback on Claude's scheduled-skill output. Tom posts free-form replies (e.g. \"drop Pershing Square from Tier 2\", \"stop including Round details\"); this skill reads the (parent alert, reply) pair, figures out what change is being requested, applies it across skill files / memory / Notion, and posts a close-loop reply in-thread. Webhook-only — invoked by claude-job-queue dispatching jobs from the slack-retro-webhook Cloudflare Worker."
---

# Claude Alerts Listener

When Tom replies to an alert in `#claude-alerts`, act on his feedback. Post a `Working on it...` reply as your very first action (Step 0 below) so Tom sees confirmation that this skill — not just the Worker — has picked up the job. Then do the work and post a close-loop reply.

**Webhook-only.** No sweep mode, no manual mode. Invoked exclusively by the claude-job-queue processor dispatching jobs from `slack-retro-webhook`.

---

## Args (passed by the Worker)

```json
{
  "mode": "webhook",
  "channel_id": "C0B06385BP1",
  "thread_ts": "<parent message ts>",
  "reply_ts": "<Tom's reply ts>",
  "user": "<Slack user ID>",
  "text": "<Tom's reply text>",
  "files": [
    {
      "id": "F...",
      "name": "citi.csv",
      "mimetype": "text/csv",
      "url_private": "https://files.slack.com/files-pri/...",
      "url_private_download": "https://files.slack.com/files-pri/.../download/..."
    }
  ]
}
```

`files` is `[]` when Tom replies with text only. When present, branches that ingest dropped files (e.g. coop-finances) should download via:

```bash
curl -sSL -H "Authorization: Bearer $SLACK_USER_TOKEN" "<url_private>" -o /tmp/<name>
```

Requires `SLACK_USER_TOKEN` in env with `files:read` scope (same token used by the Slack MCP — user-scoped works since Tom is the only uploader).

---

## Unattended execution rules

- NEVER ask questions in-session. Headless. If the reply is genuinely ambiguous, post a clarifying question **in the Slack thread** (via `post_close_loop.sh`) and exit.
- NEVER fall back to other notification channels. Close-loop reply goes ONLY to the originating thread in `#claude-alerts`.
- On failure, log to `audit-log/YYYY-MM-DD.log` and post a brief failure note in-thread (`⚠️ couldn't apply this — <one-line reason>`). Do not retry from this skill.
- **Exit promptly.** Once Step 5 (audit log) is written, STOP. No re-reading edited files to verify, no "let me also check…", no exploring adjacent skills, no proactive cleanup of unrelated content. The Edit tool's success return IS the verification. The `claude --print` wrapper has a 600s ceiling — past runs have hit it after the work was already done, producing a false-positive ⚠️ failure alert on top of a successful run. Sequential Step 0 → 1 → 2 → 3 → 4 → 5 → exit. Nothing after Step 5.

---

## Step 0. Ack Tom's message with a reaction

The Worker no longer posts a synchronous ack — that confirmation now comes from this skill, so it only fires when claude has actually started working on the task.

Before doing anything else, add a 👀 reaction to Tom's reply message (the one that triggered this job):

```bash
/Users/tomseo/.claude/skills/claude-dm-listener/react.sh \
  "C0B06385BP1" \
  "<reply_ts from args>" \
  eyes
```

This is quieter than a text "Working on it..." reply — no extra message in the thread, just a reaction visible on Tom's reply. If the reaction fails, log to audit and continue. The close-loop reply at Step 4 is still required.

---

## Step 1. Read the thread context

Use the Slack MCP to fetch the full thread:

- Tool: `mcp__claude_ai_Slack__slack_read_thread`
- Inputs: `channel_id` and `thread_ts` from the args
- If the tool isn't attached in the current session, attach it via `ToolSearch` with `query: "select:mcp__claude_ai_Slack__slack_read_thread"`.

Filter out bot messages (`bot_id` set or `subtype == "bot_message"`) — you only care about Tom's text. The first non-bot message in the thread should typically be the bot's original alert (the parent), and Tom's reply is in the args (also visible in the thread).

If the parent isn't a bot message — i.e. someone other than the `claude` Slack app posted the thread root — log `[<ts>] WARN: thread root not from bot, ignoring` to audit and exit 0. The listener is scoped to feedback on Claude alerts, not arbitrary channel chatter.

---

## Step 2. Understand the request

You now have:
- **Parent alert** — the original Claude-bot post Tom replied to. Includes a header that names the source skill (e.g. `📬 Pipeline Agent — 2026-04-25`, `📬 Research Agent`, etc.) and per-entity rows.
- **Tom's reply** — the args `text` field.

### Special branches — check in this order, BEFORE the generic taxonomy

Each branch fires on a specific parent-alert header (+ reply shape). Check them top-to-bottom; the first whose trigger matches wins. When one fires, **read its full playbook in `references/special-branches.md` now** and follow it through to its own audit-log handoff — skip Step 3 (generic taxonomy). If none match, fall through to the generic taxonomy below.

1. **Coop finances ingest / classify** — parent header contains `📒 Coop finances` (Sub-flow A: files attached; Sub-flow B: `CHECK`/`VENDOR` text classification). Procedure in `references/special-branches.md#special-branch-coop-finances-ingest--classify` — read it now.
2. **First-Pass Diligence feedback** — parent header starts with `🪏` AND contains `First Pass Diligence:`. Procedure in `references/special-branches.md#special-branch-first-pass-diligence-feedback` — read it now.
3. **Three-way intro confirmation** — parent starts with `❓ Three-way intro signal from` or `📩 Three-way intro from`, AND Tom's reply confirms an intro (checked before the NEW DEAL branch; → flip resolved Opp Status to `Connected`). Procedure in `references/special-branches.md#special-branch-three-way-intro-confirmation` — read it now.
4. **NEW DEAL opt-in / opt-out** — parent header contains `NEW DEAL` AND reply matches `/^\s*opt[\s-]?(in|out)\b\.?\s*$/i`. Procedure in `references/special-branches.md#special-branch-new-deal-opt-in--opt-out` — read it now.
5. **SOI mark confirm** — parent header starts with `📈 Priced round —` or `💸 Exit —` (valuation-judgment mark; writes `fund_inputs.json` via the engine). Procedure in `references/special-branches.md#special-branch-soi-mark-confirm` — read it now.
6. **SOI rebuild confirm (Tier 1 gate)** — parent header starts with `🧾 **SOI rebuild pending your confirm**` (Notion-derived recompute; no `fund_inputs.json` writes). Procedure in `references/special-branches.md#special-branch-soi-rebuild-confirm-tier-1-gate` — read it now.
7. **Skill-map function assignment** — parent is a `skill-map-refresh` pending-categorization alert (header `🗺️`, body names an uncategorized skill) AND Tom's reply names a Function. Procedure in `references/special-branches.md#special-branch-skill-map-function-assignment` — read it now.

---

### Generic taxonomy

Decide what change is being requested. The taxonomy is open-ended — examples:

| Example reply | Action |
|---|---|
| "drop Pershing Square from Tier 2 going forward" | Edit `research-agent/SKILL.md` to add Pershing Square to the Tier 2 deny-list |
| "stop including Round details in pipeline alerts" | Edit `pipeline-agent/SKILL.md` (or `send-alert/SKILL.md` if format-wide) to drop Round details from the alert format spec |
| "this is wrong, mark Acme as pass" | Update the Acme Notion Opportunity to `Status = Pass (DNM)` via `notion-update-page` |
| "remember that Sequoia uses 'partner' for everyone with VC in their LinkedIn title" | Save to memory (`feedback_*.md` or `reference_*.md`) |
| "good catch on flagging Beta Co" | Reinforcement, no action needed beyond acknowledgement |

When picking the target file, anchor on the parent alert's source skill — that name in the header tells you which skill's behavior Tom is critiquing. If unclear, prefer the more specific location (per-skill SKILL.md over `send-alert/SKILL.md`).

---

## Step 3. Apply the change

Edit / write / update directly. You have full access to:
- `~/.claude/skills/**/*.md` — edit skill files
- `~/.claude/projects/-Users-tomseo/memory/**` — write memory entries (also update `MEMORY.md` index)
- Notion via `mcp__claude_ai_Notion__*` — update pages, properties, etc.
- Read-only on everything else

> **Anti-hallucination — Edit/Write tools just work here.** This skill runs under `claude --print --dangerously-skip-permissions` via the claude-job-queue processor. Edit and Write on `~/.claude/skills/**/*.md` and `~/.claude/projects/-Users-tomseo/memory/**` never prompt for approval and never need it. Tom's Slack reply IS the human consent (see memory: framework writes need explicit consent). NEVER post `⚠️ couldn't apply this — skill file write requires permission not auto-granted in this session` or any variant of that message — it has been observed as a pure hallucination. If Edit truly fails, the close-loop reply must include the real stderr, never an abstract "permission" reason.

**Scope discipline:**
- Make the smallest change that satisfies Tom's intent. Don't refactor surrounding code or "improve" while you're there.
- Don't change skills unrelated to the parent alert's source skill unless Tom's reply explicitly references them.
- If Tom's request would break documented behavior elsewhere (e.g. a memory entry contradicts the change), surface that in the close-loop reply rather than silently overruling.

**When uncertain:**
- If you can't confidently identify the target file or the change semantics — post a clarifying question in-thread and exit. Examples: `"which skill's deny-list — research-agent or pipeline-agent?"`, `"do you want this applied retroactively or just going forward?"`. Don't guess on file edits; the cost of getting it wrong is harder to undo than a one-line round-trip.

---

## Step 4. Post the close-loop reply

Use the helper:

```bash
/Users/tomseo/.claude/skills/claude-alerts-listener/post_close_loop.sh \
  "C0B06385BP1" \
  "<thread_ts from args>" \
  "✅ done — <one-line summary of what was changed, with file path>"
```

Format conventions:
- `✅ done — <summary>` for successful changes
- `❓ need more info — <question>` for clarifying questions
- `⚠️ couldn't apply this — <reason>` for failures

Keep close-loop reply to **one line** when possible. If you made multiple edits, list them as a tight bullet list:

```
✅ done:
• added Pershing Square to research-agent Tier 2 deny-list
• updated send-alert/SKILL.md to drop Round from per-entity format
```

Reference file paths (e.g. `research-agent/SKILL.md`) so Tom can verify the change without leaving Slack.

---

## Step 5. Audit log

Append a one-line summary of the run to `~/.claude/skills/claude-alerts-listener/audit-log/YYYY-MM-DD.log`:

```
[<ISO timestamp>] thread=<thread_ts> reply=<reply_ts> intent=<short tag> outcome=<applied|clarification|failed> files=<comma-separated paths edited>
```

Tags should be short: `format-tweak`, `denylist-edit`, `notion-update`, `memory-write`, `ack-only`, etc.

**This is the terminator.** After the audit-log line is appended, exit 0. Do not re-read the edited file. Do not verify the Slack reply landed. Do not look at adjacent skills "while you're here". The wrapper is on a 600s clock and the work is done.

---

## Notes

- **Bot identity for posting back:** the close-loop reply posts as the `claude` Slack app via the bot token at `~/.claude/skills/claude-alerts-listener/.bot_token` (mode 600). Do NOT use the Slack MCP for the close-loop — that posts as `tom`, which defeats the whole point of the bot identity split.
- **Idempotency:** if the queue file gets reprocessed (rare but possible), the listener should be safe — most edits are idempotent (adding to a deny-list, updating a Notion property). For non-idempotent actions, check the thread first: if the close-loop reply is already in the thread, exit 0 with audit note.
- **Don't reply for ack-only feedback:** if Tom says "thanks" or "good catch" with no actionable content, still post a close-loop reply (`✅ noted, no change needed`) so he knows you read it. Tom said he won't send filler replies, so this branch is rare.
