---
name: claude-dm-listener
description: "Processes direct messages sent to the `claude` Slack bot. Tom DMs the bot with arbitrary commands (e.g. \"draft a pass note for Acme\", \"add this LinkedIn URL to -1 Sourcing\", \"what's on my pipeline today?\"); this skill reads the message, figures out what to do (often by invoking another skill), executes it, and posts a reply. Webhook-only — invoked by claude-job-queue dispatching jobs from the slack-retro-webhook Cloudflare Worker on `message.im` events."
---

# Claude DM Listener

When Tom sends a direct message to the `claude` Slack bot OR posts a top-level message in `#claude-alerts`, treat the message as a command and execute it. This is the headless equivalent of asking Claude Code to do something — full skill access, broad authority, do whatever Tom asked for, post a reply when done.

Claim the job with a 👀 reaction as your very first action (Step 0 below), so Tom sees that this skill — not just the Worker — has picked it up. Then do the work and post the result.

**Webhook-only.** No sweep mode, no manual mode. Invoked by the claude-job-queue processor dispatching jobs from `slack-retro-webhook` on two paths:
- `message.im` events (Tom DM'd the bot) — `channel_id` starts with `D`
- Top-level posts in `#claude-alerts` (channel `C0B06385BP1`) — `channel_id == "C0B06385BP1"`, no `thread_ts` set

For both, post replies threaded under Tom's command using `reply_ts` as the `thread_ts`. The reply will appear as a threaded response in DMs OR as a thread under his top-level alert post.

---

## Args (passed by the Worker)

```json
{
  "mode": "webhook",
  "channel_id": "D0XXXXXXXXX",
  "thread_ts": null,
  "reply_ts": "<ts of Tom's message>",
  "user": "<Tom's Slack user ID>",
  "text": "<Tom's message text>",
  "files": [
    {
      "id": "F...",
      "name": "filename.csv",
      "mimetype": "text/csv",
      "url_private": "https://files.slack.com/...",
      "url_private_download": "https://files.slack.com/.../download/..."
    }
  ]
}
```

`thread_ts` is `null` (no parent context). Reply by passing `reply_ts` as the `thread_ts` to `post_reply.sh` so your responses thread under Tom's command.

`files` is `[]` when no attachment, populated when Tom drops a file. Download via:

```bash
curl -sSL -H "Authorization: Bearer $SLACK_USER_TOKEN" "<url_private>" -o /tmp/<name>
```

`SLACK_USER_TOKEN` is injected by the processor (`_skill_env()` in processor.py) from `~/.claude/.slack-bot-token.enc`. The token has `files:read` scope.

---

## Unattended execution rules

- NEVER ask questions in-session. Headless. If the request is genuinely ambiguous and would cause harm to guess, post a clarifying question in-thread (via `post_reply.sh`) and exit.
- NEVER fall back to other notification channels. Reply ONLY in the originating DM thread.
- On failure, log to `audit-log/YYYY-MM-DD.log` and post a brief failure note in-thread. Do not retry from this skill.

---

## Step 0. Claim the job with a reaction

Reactions carry the status signal — never post an up-front text "Working on it…" reply, which is just noise in the DM. (A mid-run *progress* update on genuinely long work is a different thing and still fine — see Step 2.)

| | meaning | who adds it |
|---|---|---|
| ⏳ `hourglass_flowing_sand` | queued | **slack-retro-webhook**, at ingest (sub-second) |
| 👀 `eyes` | working | this skill, Step 0 |

There's no completion tombstone here — Step 3's reply *is* the completion signal. (`claude-alerts-listener` additionally adds 🏁, because it needs a machine-readable claim/complete pair for idempotency; this skill doesn't.)

Before doing anything else, add 👀 to Tom's message, then clear the Worker's ⏳ — it has served its purpose the moment 👀 lands:

```bash
/Users/tomseo/.claude/skills/claude-dm-listener/react.sh \
  "<channel_id from args>" \
  "<reply_ts from args>" \
  eyes

/Users/tomseo/.claude/skills/claude-dm-listener/react.sh \
  "<channel_id from args>" \
  "<reply_ts from args>" \
  hourglass_flowing_sand remove
```

If either call fails, log to audit and continue; the result reply at Step 3 is still required. `remove` is a no-op-safe call (a missing ⏳ exits 0).

---

## Step 1. Read the command

The args `text` field is Tom's command. It can be anything:

| Example command | What to do |
|---|---|
| `draft a pass note for Acme` | Read `pass-note-drafter/SKILL.md` and execute its manual mode for Acme |
| `add https://linkedin.com/in/jane to -1 sourcing` | Read `neg1-scanner/SKILL.md` and run it on the URL |
| `what's the status of [opp]?` | Notion fetch + summarize, reply with a one-paragraph status |
| `run the diligence agent` | Read `diligence-agent/SKILL.md` and execute its sweep mode |
| `remember that I prefer X` | Save a memory entry per `auto memory` rules in CLAUDE.md |
| `what skills do I have for outreach?` | Grep skills directory, reply with a list |
| `ingest co-op financials` / `update coop financials` + CSV file attached | Read `coop-finances/SKILL.md` Sub-flow A in the "Special branch: Coop finances ingest / classify" section of `claude-alerts-listener/SKILL.md`. Download the CSV (and any reserve attachment) via the bot token, invoke `python3 ~/.claude/skills/coop-finances/update_pl.py --csv /tmp/<name>` (add `--reserve-csv` if a reserve file is also attached), capture stdout JSON, format flagged-summary reply per that branch's spec. |
| `CHECK <N> = <Category>` / `VENDOR <substring> = <Category>` (no files) | Coop-finances classification confirmation. Append to `~/.claude/skills/coop-finances/references/CHECK_LEDGER.md` or `VENDOR_CLASSIFICATIONS.md` per the same Sub-flow B spec in `claude-alerts-listener/SKILL.md`. |

Treat yourself as a routing layer: identify the right skill (or direct action) and execute it. You have Read access to every skill's SKILL.md — use that to learn what's available.

---

## Step 2. Execute

You have full tool access. Use whatever is needed:
- Read / Edit / Write / Bash on the local filesystem
- Notion via `mcp__claude_ai_Notion__*`
- Gmail via `mcp__claude_ai_Gmail__*`
- Slack via `mcp__claude_ai_Slack__*` (read-only operations — DO NOT use this to post replies; use `post_reply.sh` so the reply comes from the `claude` bot identity, not Tom's user)
- Any other MCP attached to the session

**Scope discipline:**
- Do exactly what Tom asked. Don't expand scope.
- For long-running work (>2min), post an interim "still working on this..." reply so Tom knows you're alive.
- For commands that invoke another skill, follow that skill's SKILL.md verbatim — don't shortcut its steps.

**When uncertain:**
- If the request is ambiguous and a wrong guess could cause real harm (deleting data, sending an email to the wrong person, editing the wrong Notion page), post a clarifying question in-thread and exit.
- For low-stakes ambiguity, make a reasonable interpretation and proceed — Tom can correct in a follow-up DM.

---

## Step 3. Reply

Use the helper:

```bash
/Users/tomseo/.claude/skills/claude-dm-listener/post_reply.sh \
  "<channel_id from args>" \
  "<reply text>" \
  "<reply_ts from args>"
```

The third argument (`reply_ts`) makes the reply thread under Tom's command. Threading keeps each command's response self-contained.

Format:
- `✅ done — <one-line summary>` for successful actions, with file paths / Notion links / etc. so Tom can verify
- `❓ <question>` for clarifying questions
- `⚠️ couldn't do this — <reason>` for failures

For multi-action commands, list as a tight bullet list. Reference paths so Tom can audit.

---

## Step 4. Audit log

Append to `~/.claude/skills/claude-dm-listener/audit-log/YYYY-MM-DD.log`:

```
[<ISO timestamp>] reply_ts=<ts> intent=<short tag> outcome=<applied|clarification|failed> notes=<what got done>
```

---

## Notes

- **Bot identity for posting back:** the reply posts as the `claude` Slack app via the bot token at `~/.claude/skills/claude-dm-listener/.bot_token` (mode 600). Do NOT use the Slack MCP for replies — that posts as `tom`, defeating the bot identity split (Tom would be talking to himself).
- **Idempotency:** if the queue file gets reprocessed, check the thread first via `mcp__claude_ai_Slack__slack_read_thread` (channel=channel_id, thread_ts=reply_ts). If the bot has already posted a reply in this thread, exit 0 with audit note — don't duplicate work.
- **Long tasks:** the per-job timeout is 900s (15 min) — `timeout_sec` set by `slack-retro-webhook` when it enqueues, matching the processor's own default (raised from 600s on 2026-08-03). For longer work, post an interim reply early so Tom knows the task is in flight, then continue. If you genuinely need >15min, raise `timeout_sec` in the Worker or break the work into multiple commands.
