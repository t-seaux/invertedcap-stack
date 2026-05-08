---
name: claude-dm-listener
description: "Processes direct messages sent to the `claude` Slack bot. Tom DMs the bot with arbitrary commands (e.g. \"draft a pass note for Acme\", \"add this LinkedIn URL to -1 Sourcing\", \"what's on my pipeline today?\"); this skill reads the message, figures out what to do (often by invoking another skill), executes it, and posts a reply. Webhook-only — invoked by claude-job-queue dispatching jobs from the slack-retro-webhook Cloudflare Worker on `message.im` events."
---

# Claude DM Listener

When Tom sends a direct message to the `claude` Slack bot, treat the message as a command and execute it. This is the headless equivalent of asking Claude Code to do something — full skill access, broad authority, do whatever Tom asked for, post a reply when done.

Post a `Working on it...` reply as your very first action (Step 0 below) so Tom sees confirmation that this skill — not just the Worker — has picked up the job. Then do the work and post the result.

**Webhook-only.** No sweep mode, no manual mode. Invoked exclusively by the claude-job-queue processor dispatching jobs from `slack-retro-webhook` on `message.im` events.

---

## Args (passed by the Worker)

```json
{
  "mode": "webhook",
  "channel_id": "D0XXXXXXXXX",
  "thread_ts": null,
  "reply_ts": "<ts of Tom's DM>",
  "user": "<Tom's Slack user ID>",
  "text": "<Tom's DM text>"
}
```

`thread_ts` is `null` for DMs (no parent context like alert-replies have). Reply by passing `reply_ts` as the `thread_ts` to `post_reply.sh` so your responses thread under Tom's command.

---

## Unattended execution rules

- NEVER ask questions in-session. Headless. If the request is genuinely ambiguous and would cause harm to guess, post a clarifying question in-thread (via `post_reply.sh`) and exit.
- NEVER fall back to other notification channels. Reply ONLY in the originating DM thread.
- On failure, log to `audit-log/YYYY-MM-DD.log` and post a brief failure note in-thread. Do not retry from this skill.

---

## Step 0. Ack the DM

The Worker no longer posts a synchronous ack — that confirmation now comes from this skill, so it only fires when claude has actually started working on the task.

Before doing anything else, post `Working on it...` threaded under Tom's command:

```bash
/Users/tomseo/.claude/skills/claude-dm-listener/post_reply.sh \
  "<channel_id from args>" \
  "Working on it..." \
  "<reply_ts from args>"
```

If the post fails, log to audit and continue — the result reply at Step 3 is still required.

---

## Step 1. Read the command

The args `text` field is Tom's command. It can be anything:

| Example DM | What to do |
|---|---|
| `draft a pass note for Acme` | Read `pass-note-drafter/SKILL.md` and execute its manual mode for Acme |
| `add https://linkedin.com/in/jane to -1 sourcing` | Read `neg1-scanner/SKILL.md` and run it on the URL |
| `what's the status of [opp]?` | Notion fetch + summarize, reply with a one-paragraph status |
| `run the diligence agent` | Read `diligence-agent/SKILL.md` and execute its sweep mode |
| `remember that I prefer X` | Save a memory entry per `auto memory` rules in CLAUDE.md |
| `what skills do I have for outreach?` | Grep skills directory, reply with a list |

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
- **Long tasks:** the processor's per-job timeout is 600s (10 min) by default. For longer work, post an interim reply early so Tom knows the task is in flight, then continue. If you genuinely need >10min, increase `timeout_sec` on the listener side or break the work into multiple commands.
