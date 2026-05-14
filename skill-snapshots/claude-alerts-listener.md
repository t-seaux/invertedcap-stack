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
  "text": "<Tom's reply text>"
}
```

---

## Unattended execution rules

- NEVER ask questions in-session. Headless. If the reply is genuinely ambiguous, post a clarifying question **in the Slack thread** (via `post_close_loop.sh`) and exit.
- NEVER fall back to other notification channels. Close-loop reply goes ONLY to the originating thread in `#claude-alerts`.
- On failure, log to `audit-log/YYYY-MM-DD.log` and post a brief failure note in-thread (`⚠️ couldn't apply this — <one-line reason>`). Do not retry from this skill.

---

## Step 0. Ack the thread

The Worker no longer posts a synchronous ack — that confirmation now comes from this skill, so it only fires when claude has actually started working on the task.

Before doing anything else, post `Working on it...` to the originating thread:

```bash
/Users/tomseo/.claude/skills/claude-alerts-listener/post_close_loop.sh \
  "C0B06385BP1" \
  "<thread_ts from args>" \
  "Working on it..."
```

If the post fails, log to audit and continue — the close-loop reply at Step 4 is still required.

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

### Special branch: NEW DEAL opt-in / opt-out

Check this **before** the generic taxonomy below. If the parent alert header contains `NEW DEAL` AND Tom's reply text matches `/^\s*opt[\s-]?(in|out)\b\.?\s*$/i`, run the dedicated handler in this section and skip Step 3 + the generic Step 4 close-loop.

**What this branch does:** drafts a Gmail reply to the referrer (warm acceptance for opt-in, polite decline for opt-out), flips the Notion Opp Status (opt-in → Outreach, opt-out → Pass (DNM)), and posts a close-loop with a link to the draft. The status flip is also covered as a fallback by `outreach-detector` / `outreach-decliner` webhooks once Tom sends the email — but flipping here is faster and avoids haystack-match fragility for vague referrer subjects.

**Handler steps:**

1. **Parse parent alert text:**
   - Notion URL — regex `https://www\.notion\.so/[A-Za-z0-9-]+`
   - Opp name — text on the 🏢 line, before the ` |` separator
   - Referrer name — text after `referred by ` up to ` @` or end of line

   If any of the three are missing, post `⚠️ couldn't parse alert — <missing field>` and exit.

2. **Find the original referral thread in Gmail:**
   - Tool: `mcp__claude_ai_Gmail__search_threads`
   - Primary query: `from:"<referrer name>" "<opp name>" newer_than:2d`
   - Fallback if 0 results: `from:"<referrer name>" newer_than:2d` and pick the most recent thread whose subject or body mentions the opp name
   - If still 0 results, post `⚠️ couldn't find referral thread for <opp name> from <referrer name>` and exit

3. **Draft the Gmail reply** via `mcp__claude_ai_Gmail__create_draft`, replying to the most recent message in that thread. Use the canonical templates **verbatim** — substitute `NAME` with the referrer's first name (parsed from the alert's `referred by <First Last>` text). Use en dash (`–`), never em dash. Don't invent details about the company.

   **Opt-in template:**
   ```
   NAME – would love to chat. Appreciate you thinking of me!

   Best,
   Tom
   ```

   **Opt-out template:**
   ```
   NAME,

   Appreciate you thinking of me! Unfortunately going to sit this one out as it's a bit out of scope for me. Hope that's okay.

   Best,
   Tom
   ```

4. **Flip Notion Opp Status:**
   - Extract page ID from the Notion URL
   - Tool: `mcp__claude_ai_Notion__notion-update-page` updating only the `Status` property
   - opt-in → `Outreach`
   - opt-out → `Pass (DNM)`

5. **Post close-loop reply** via `post_close_loop.sh`, including the draft link (`https://mail.google.com/mail/u/0/#drafts/<draft_id>`):
   - opt-in: `✅ Draft saved + Status → Outreach. Review: <draft url>`
   - opt-out: `✅ Draft saved + Status → Pass (DNM). Review: <draft url>`

6. **Audit log** per Step 5 (intent tag: `intro-opt-in` or `intro-opt-out`), then exit.

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

---

## Notes

- **Bot identity for posting back:** the close-loop reply posts as the `claude` Slack app via the bot token at `~/.claude/skills/claude-alerts-listener/.bot_token` (mode 600). Do NOT use the Slack MCP for the close-loop — that posts as `tom`, which defeats the whole point of the bot identity split.
- **Idempotency:** if the queue file gets reprocessed (rare but possible), the listener should be safe — most edits are idempotent (adding to a deny-list, updating a Notion property). For non-idempotent actions, check the thread first: if the close-loop reply is already in the thread, exit 0 with audit note.
- **Don't reply for ack-only feedback:** if Tom says "thanks" or "good catch" with no actionable content, still post a close-loop reply (`✅ noted, no change needed`) so he knows you read it. Tom said he won't send filler replies, so this branch is rare.
