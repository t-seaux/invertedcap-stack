---
name: claude-dm-listener
description: "Tom's conversation surface on Slack: DMs to the `claude` bot, top-level posts in #claude-alerts / #personal-alerts, and follow-ups in threads rooted on Tom's own posts. Every message is the next line of a conversation (the thread / recent DMs are read first, both sides) — a question gets an answer, a command gets executed (often by invoking another skill), a correction adjusts the last action; replies in-thread. Webhook-only — invoked by claude-job-queue dispatching jobs from the slack-retro-webhook Cloudflare Worker on `message.im` events."
---

# Claude DM Listener

> **Headless Gmail:** every Gmail read/write in this skill follows `shared-references/headless-gmail.md` — reads via `admin_run.py` when the Gmail MCP isn't attached (its absence ≠ Gmail down); a run that can't finish ends with a `JOB_FAILED:` line.

When Tom sends a direct message to the `claude` Slack bot, posts a top-level message in `#claude-alerts` / `#personal-alerts`, or replies in a thread rooted on one of his own posts there, treat it as **the next line of a conversation** with you (Tom 2026-10-08: *"want to be able to have conversations with you over slack"*). This is the headless equivalent of talking to Claude Code — full skill access, broad authority. Sometimes that means executing a command; often it means answering a question, thinking something through with him, or adjusting what you just did.

Claim the job with a 👀 reaction as your very first action (Step 0 below), so Tom sees that this skill — not just the Worker — has picked it up. Then do the work and post the result.

**Webhook-only.** No sweep mode, no manual mode. Invoked by the claude-job-queue processor dispatching jobs from `slack-retro-webhook` on two paths:
- `message.im` events (Tom DM'd the bot) — `channel_id` starts with `D`; `thread_ts` set when he replied inside a DM thread
- Top-level posts in `#claude-alerts` (`C0B06385BP1`) or `#personal-alerts` (`C0BKZ2L0BDK`) — no `thread_ts`
- Thread replies in those channels whose thread root is **Tom's own post** — `thread_ts` = that root (replies under a *bot alert* go to `claude-alerts-listener` instead)
- Any other **Claude space** — a channel whose only members are Tom and the bot, including channels created later (detected by membership in the Worker; no registration) — same rules as above
- **`@claude` anywhere else** — shared channels, group DMs: Tom tagged the bot, `args.shared` is `true`, the tag is already stripped from `text`
- Hand-offs from `decision-retro-listener` / `neg1-sourcing-listener` when a message there isn't a retro / card decision / channel command (`handoff.sh`)

Always post replies in the thread: `thread_ts` when set (keep the conversation in its thread), else `reply_ts` (start a thread under Tom's message).

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

`thread_ts` is the thread root when Tom wrote inside a thread, `null` for a top-level message. Reply thread = `thread_ts ?? reply_ts` (pass it as `post_reply.sh`'s third arg).

`shared` is `true` when other people can read the reply (shared channel, group DM). **In a shared space:** answer what was asked and nothing more. Never put fund / LP / portfolio-confidential material (marks, SOI, LP names or commitments, deal terms, retros, pass reasons, pipeline status) in the reply; if the ask needs it, reply `I'll DM you this` and post it to Tom's DM with the bot instead (`post_reply.sh D0B0R3NPTV2 "<text>"` — Tom's DM channel with the bot). Absent/`false` = Tom-only space.

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
| 👀 `eyes` | working | this skill, Step 0 (`claim.sh`) |
| 🏁 `checkered_flag` | done | this skill, Step 3 (`post_reply.sh … done`) |

Before doing anything else, claim the job. `claim.sh` reads the bot's OWN reactions on the trigger message FIRST, then claims it (adds 👀, clears the Worker's ⏳) — so a re-run can tell itself apart from a fresh run:

```bash
/Users/tomseo/.claude/skills/claude-dm-listener/claim.sh "<channel_id from args>" "<reply_ts from args>"
```

| exit | verdict | do |
|---|---|---|
| 0 | `fresh` | proceed normally |
| 10 | `done` — a prior run already posted its final reply (🏁) | exit 0 with an audit note; no work, no reply |
| 11 | `resume` — a prior run claimed (👀) and died before finishing | side effects may be partially applied: **verify before every non-idempotent step** (e.g. search for an existing draft and reuse it), finish only the remainder, and say so in the reply |
| 1 | `error` — couldn't read reactions | treat as `resume` |

Only the bot's reactions count — Tom's ✅ means "confirm", never "done". (Code-enforced 2026-10-04: the old "add 👀, later check for 👀" read always saw its own 👀.)

---

## Step 1. Read the conversation, then the message

**Conversation first** (same rule as the text lane — `feedback_text_agent_conversation_first`; one brain across surfaces). Before interpreting `args.text`, load what came before it:

```bash
# Tom wrote inside a thread (thread_ts set): the whole thread, both sides
python3 /Users/tomseo/.claude/skills/claude-dm-listener/read_thread.py "<channel_id>" "<thread_ts>" --exclude "<reply_ts>"
# top-level DM (channel_id starts with D, thread_ts null): the last few DMs from the past 6h + their threads
python3 /Users/tomseo/.claude/skills/claude-dm-listener/read_thread.py "<channel_id>" --recent "<reply_ts>"
```

(Top-level channel post with no thread: there is no prior context — skip.) Lines are `tom:` / `claude:`. Exit 1 = Slack read failed → proceed on `args.text` alone and say so if it matters. Empty output = no recent conversation.

Then ask what `args.text` IS relative to that transcript:
- **an answer** to a question you asked → carry on with what you were doing
- **a correction / tweak** of your last action ("no, the other Acme", "make it shorter", `word*` = typo fix) → adjust that action, don't start over
- **a follow-up question or discussion** ("why that one?", "what do you think about X?") → answer it, grounded in the thread + live sources (Notion, Gmail, files, memory). Thinking out loud with Tom is a first-class outcome, not a fallback
- **a new command** → execute it (table below)

If a discussion surfaces a durable preference or rule, save it to its real home (skill / corpus / memory) and say where — a rule stated in Slack binds everywhere.

Commands can be anything:

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
- Do exactly what Tom asked. Don't expand scope. A question is answered, not turned into an edit.
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
  "<thread_ts from args, or reply_ts when thread_ts is null>" \
  done "<reply_ts from args>"
```

`done` adds 🏁 to Tom's message once the post lands (the completion half of `claim.sh`). Pass it on the FINAL reply only — never on an interim progress update. The 5th arg (`reply_ts`) puts 🏁 on Tom's message, not the thread root — always pass it.

The third argument keeps the conversation in one thread: inside an existing thread, reply there; a top-level message starts a thread under itself.

Format:
- **Conversation** (answers, discussion, pushback): plain prose, like a reply in a chat — no `✅ done` prefix. As long as the answer needs, no longer; Slack mrkdwn (`*bold*`, `<url|label>`).
- `✅ done — <one-line summary>` for completed actions, with file paths / Notion links / etc. so Tom can verify
- `❓ <question>` for clarifying questions
- `⚠️ couldn't do this — <reason>` for failures

For multi-action commands, list as a tight bullet list. Reference paths so Tom can audit.

---

## Step 4. Audit log

Append to `~/.claude/skills/claude-dm-listener/audit-log/YYYY-MM-DD.log`:

```
[<ISO timestamp>] reply_ts=<ts> intent=<short tag> outcome=<applied|answered|clarification|failed> notes=<what got done>
```

---

## Notes

- **Bot identity for posting back:** the reply posts as the `claude` Slack app via the bot token at `~/.claude/skills/claude-dm-listener/.bot_token` (mode 600). Do NOT use the Slack MCP for replies — that posts as `tom`, defeating the bot identity split (Tom would be talking to himself).
- **Idempotency:** `claim.sh` (Step 0) decides — `done` → exit quietly, `resume` → verify partial side effects before redoing any (code-enforced 2026-10-04; the old "has the bot replied in-thread" check was blind to a run that died after its side effect but before replying).
- **Long tasks:** the per-job timeout is 900s (15 min) — `timeout_sec` set by `slack-retro-webhook` when it enqueues, matching the processor's own default (raised from 600s on 2026-08-03). For longer work, post an interim reply early so Tom knows the task is in flight, then continue. If you genuinely need >15min, raise `timeout_sec` in the Worker or break the work into multiple commands.
