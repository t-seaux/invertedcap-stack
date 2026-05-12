---
name: decision-retro-listener
description: "Processes thread replies in #decision-retros. Mode A (sweep): daily 6pm ET reconciliation pass over the full queue — catches what the webhook missed. Mode B (webhook): invoked via claude-job-queue when Slack fires a thread-reply event. Two B sub-flows: B0 detects parent thread type — retro prompts route to the existing nugget-extraction flow (B1+); weekly retro summary posts (from retro-weekly-summary) route to a feedback-handling flow (Mode B-summary) that mirrors claude-alerts-listener — ack, apply edits to retro-weekly-summary/SKILL.md or FOUNDER_EVAL_CASEBOOK.md, post close-loop. Framework writes (FOUNDER_EVAL_FRAMEWORK.md) are allowed only when Tom's reply explicitly authorizes them — his Slack reply IS the human gate. Not user-facing — invoked exclusively by scheduled-tasks/decision-retro-listener/run.sh (Mode A) or claude-job-queue-processor dispatching slack-retro-webhook jobs (Mode B)."
---

# Decision Retro Listener

Reads Tom's replies in `#decision-retros` threads, extracts framework nuggets, logs them to the source page (Opp or -1 Scanner row) and to `DECISION_RETROS.md`, and marks the queue entry done.

**Modes**
- **A (sweep):** daily 6 PM ET reconciliation pass. Iterates all `status=prompted` queue items. Invoked by `~/.claude/scheduled-tasks/decision-retro-listener/run.sh`.
- **B (webhook):** fires immediately on a Slack thread-reply event. Processes exactly one thread. Invoked by the claude-job-queue processor dispatching jobs from the `slack-retro-webhook` Cloudflare Worker.

**Mode B runs first in practice; Mode A picks up stragglers.**

---

## Unattended execution rules

- NEVER ask questions. NEVER prompt Tom in-session. Headless.
- On per-item failures, log to `audit-log/YYYY-MM-DD.log` and continue (Mode A) or exit non-zero so the job lands in `failed/` (Mode B).
- **No Slack DM alert in either mode.** Mode A's success signal is the per-Opp page Retro section + nugget appends to `DECISION_RETROS.md` + queue items flipped to `completed`. Mode B exits silently after writing. Tom does NOT want listener summaries in the `tom` self-DM channel — all retro chatter stays in `#decision-retros`.

## Shared paths

- Queue: `/Users/tomseo/.claude/skills/decision-retro/queue.json`
- Audit: `/Users/tomseo/.claude/scheduled-tasks/decision-retro-listener/audit-log/`
- Master nugget log: `/Users/tomseo/.claude/skills/neg1-enricher/DECISION_RETROS.md`

---

## Mode A — Scheduled sweep

### A1. Load the queue

If missing or empty, write `[<ts>] INFO: queue empty — no items to process` to the audit log and exit 0. **Do NOT send a Slack DM** (would land in the `tom` self-DM channel, which Tom explicitly does not want for listener output).

### A2. Find the #decision-retros channel ID

Use `mcp__claude_ai_Slack__slack_search_channels` (query: `decision-retros`).

If the Slack MCP tool is NOT attached, use `ToolSearch` with `query: "select:mcp__claude_ai_Slack__slack_search_channels"` to attach. If still unavailable after three attempts, write `[<ts>] ERROR: Slack MCP unavailable — aborting run` to the audit log and exit 0.

### A3. Iterate queue items with `status == "prompted"`

For each, run the single-item processor defined in the **Shared processing** section below. If processing succeeds, continue to the next item. If it fails, log to audit and continue — do not abort the run.

Before running the processor, also check age:
- Compute `age_days = now - prompted_at`
- If `age_days > 7`: set `status = "no_retro"`, `completed_at = now`, `nugget_count = 0`, and skip the processor

### A4. Summary alert

**No Slack DM summary.** The listener silently writes nuggets to source pages + `DECISION_RETROS.md` + flips queue items to `completed`. Tom does NOT want listener summaries in the `tom` self-DM channel — all retro chatter stays in `#decision-retros` (and the per-Opp page Retro sections are the durable record). Same rule as `decision-retro-scan` and `neg1-retro-scan`.

The Mode A sweep audit log (`scheduled-tasks/decision-retro-listener/audit-log/YYYY-MM-DD.log`) is the operator-visible trail. Inspect there if Tom asks "did the listener pick up X?".

---

## Mode B — Webhook (per-reply)

Invoked with args from `slack-retro-webhook` Worker:

- `mode: "webhook"` (flag — always present)
- `channel_id` — Slack channel ID
- `thread_ts` — thread root ts (the retro prompt message)
- `reply_ts` — ts of the reply that triggered this job
- `user` — Slack user ID of the replier
- `text` — raw text of the reply

### B0. Detect parent thread type — retro vs. weekly summary

The `slack-retro-webhook` Worker dispatches every thread reply in `#decision-retros` to this skill, but the channel hosts two kinds of threads:

1. **Retro prompts** (per-Opp / per -1 candidate) — Tom's reply is a retro to extract; flow continues at B1.
2. **Weekly retro summary posts** (Friday 4pm ET, posted by `retro-weekly-summary`) — Tom's reply is *feedback on the summary alert itself* (formatting, theme classification, casebook entries, suggested refinements). Mirror the `claude-alerts-listener` pattern: ack, apply, close-loop.

Detect which kind by reading the parent message:

```
mcp__claude_ai_Slack__slack_read_thread (channel_id, thread_ts)
```

Pull the first non-bot-filtered message — actually, in this case we WANT the bot message since both retro prompts AND weekly summaries are posted by the `claude` app. Find the message whose `ts == thread_ts`. Inspect its text:

- If the parent text contains `Weekly Retro Summary` (case-insensitive) → run **Mode B-summary** below, then exit.
- Otherwise → continue to B1 (existing retro flow).

If `slack_read_thread` fails or returns no parent: log `[<ts>] WARN: B0 could not read parent thread, defaulting to retro flow` and continue to B1.

---

### Mode B-summary — feedback on the weekly retro summary

Tom replied to a `retro-weekly-summary` post. Treat his reply as a free-form change request on the summary skill, the Casebook, or the alert format. Mirror `claude-alerts-listener` Steps 0–5:

**B-summary.0. Ack the thread**

```bash
~/.claude/skills/claude-alerts-listener/post_close_loop.sh \
  "<channel_id from args>" \
  "<thread_ts from args>" \
  "Working on it..."
```

(Reuses the `claude-alerts-listener` helper + `.bot_token`. The `claude` app has post access to `#decision-retros`.)

**B-summary.1. Read full thread context**

`mcp__claude_ai_Slack__slack_read_thread` again to get full thread (parent + all replies). Filter to non-bot messages — Tom's text is what you act on. The parent (the summary alert text) is the context for what he's critiquing.

**B-summary.2. Understand the request**

Tom's reply is open-ended feedback on the weekly summary. Examples:

| Example reply | Action |
|---|---|
| "bold the decision labels" | Edit `retro-weekly-summary/SKILL.md` Step 7 format spec |
| "Spangler shouldn't be `Pass` — should be `Pass (Met)`" | Either (a) flip Notion Status if it's actually wrong, or (b) confirm the re-fetch in Step 2b is doing its job; do NOT hand-edit the queue |
| "drop the 'Coaching the founder' theme — duplicate of existing entry" | Edit `FOUNDER_EVAL_CASEBOOK.md` to remove or consolidate the entry |
| "promote 'Sub-scale seed' to a Framework dimension" | Tom's explicit reply IS the human gate — edit `FOUNDER_EVAL_FRAMEWORK.md`. Make the smallest change that satisfies intent (new dimension, threshold shift, anchored example). Echo the diff back in the close-loop reply so Tom can verify. |

**Scope constraints:**

- ALLOWED writes (always): `~/.claude/scheduled-tasks/retro-weekly-summary/SKILL.md`, `~/.claude/skills/neg1-enricher/FOUNDER_EVAL_CASEBOOK.md`, `~/.claude/skills/neg1-enricher/DECISION_RETROS.md`, `send-alert/md_to_blocks.py` (only for format-spec changes), memory under `~/.claude/projects/-Users-tomseo/memory/`.
- ALLOWED with explicit request: `~/.claude/skills/founder-outreach/FOUNDER_EVAL_FRAMEWORK.md`. Tom's reply must clearly authorize it — e.g., "promote X to the framework", "add Y as a new dimension", "update the framework rubric to ...", "add this anchor to Signal 5". A passing mention of the Framework in a broader request does NOT count; the action verb has to target the Framework. When in doubt, post a clarifying reply and exit.
- Notion writes are allowed only if Tom is explicitly asking to fix a stale Notion field (e.g., "flip Spangler to Pass (Met)").

**Framework-edit guardrails** (only relevant when the explicit-request path fires):
- Echo the diff in the close-loop reply: a fenced code block with the changed section before/after, or a tight bullet of "added section §X.Y named Z, anchored to <case>". Tom needs to be able to verify without leaving Slack.
- Smallest change that satisfies intent. Don't refactor surrounding sections, don't renumber, don't introduce abstractions Tom didn't ask for.
- If the request would conflict with existing Framework content (e.g., promoting a pattern that's already a dimension under a different name), surface that in the close-loop instead of writing — `❓ existing Signal 4 already covers this — fold in or rename?` etc.
- If Tom's request mentions the Casebook entry to promote, also annotate the Casebook entry that it was promoted (one-line `**Promoted to Framework**: <date> — see §X.Y`) so the cases→doctrine flow stays traceable.

**B-summary.3. Apply the change**

Use the Edit tool directly — this job runs under `--dangerously-skip-permissions` and Tom's Slack reply IS the human gate. Do not report "blocked" or post the diff for approval; just apply it.

Smallest edit that satisfies intent. Prefer per-skill SKILL.md edits over `send-alert/`-wide format changes unless Tom explicitly asks for a global format change.

If genuinely ambiguous (the request could mean two different things and the diff would be non-trivially different), post a clarifying reply via `post_close_loop.sh` and exit — don't guess.

**B-summary.4. Post close-loop**

```bash
~/.claude/skills/claude-alerts-listener/post_close_loop.sh \
  "<channel_id>" \
  "<thread_ts>" \
  "✅ done — <one-line summary with file path>"
```

Format conventions: `✅ done — <summary>` / `❓ need more info — <q>` / `⚠️ couldn't apply — <reason>`.

**B-summary.5. Audit log**

Append to `~/.claude/skills/decision-retro-listener/audit-log/YYYY-MM-DD.log`:

```
[<ISO ts>] thread=<thread_ts> reply=<reply_ts> intent=summary-feedback outcome=<applied|clarification|refused|failed> files=<comma-separated paths>
```

Use `intent=summary-feedback` so these entries are distinguishable from retro-flow entries.

Exit 0. Do NOT continue to B1.

---

### B1. Find the matching queue item

Load the queue. Find the item whose Slack prompt in `#decision-retros` has `ts == thread_ts`.

Because older queue entries may not have `thread_ts` cached, do a Slack lookup to resolve: use `mcp__claude_ai_Slack__slack_read_channel` on `channel_id`, read ~30 messages around `thread_ts`, find the one with that ts, extract the fingerprint `[opp:<SHORT_ID>]` or `[neg1:<SHORT_ID>]`, then match the queue entry by `short_id`.

If no queue match: log `[<ts>] WARN: webhook reply to unknown thread_ts=<ts>` to audit and exit 0 (not an error — could be a reply to a non-retro prompt).

### B2. Check idempotency

If the queue item has `status == "completed"` or `status == "skipped"`, this is a re-delivery or a late reply after processing. Exit 0 with audit note `[<ts>] INFO: thread_ts=<ts> already processed (status=<x>), skipping`.

### B3. Run the single-item processor

Run the **Shared processing** section below against this one item.

### B4. Exit

No alert. The Slack post on the source page (created by step 3f) is the confirmation that matters.

---

## Shared processing (single item)

Given one queue item, run these steps in order:

### 3a. Read the thread

Use `mcp__claude_ai_Slack__slack_read_thread` with the channel ID and the prompt message `ts` (`thread_ts` in Mode B; discover via fingerprint grep in Mode A per the original listener logic).

Filter replies to those authored by a real user (not the `claude` bot). Filter out any message with `bot_id` or `subtype == "bot_message"`.

### 3b. Handle "no reply yet" (Mode A only)

- Zero user replies: leave `status = "prompted"`, continue to next queue item.
- Mode B should never see zero user replies (the webhook fires on a reply), but if it does, log and exit 0 defensively.

### 3c. Concatenate raw retro text

Join all user messages in thread order with `\n\n`. This is the raw retro text.

### 3d. Skip keyword check

If the concatenated text — trimmed, lowercased, punctuation stripped — matches `^(ignore|skip|pass|n/?a|no thanks|nope|nah|nvm|not today)$`, treat as an explicit skip:
- Set queue entry `status = "skipped"`, `completed_at = now()`, `nugget_count = 0`
- Do NOT extract nuggets, do NOT write to the source page, do NOT append to `DECISION_RETROS.md`
- Return success

### 3e. Extract nuggets

Fetch the source page via `notion-fetch` with `opp_url`:
- `scope == "opp"`: Opportunity page. Context snippet = stage, fund, brief description.
- `scope == "neg1"`: -1 Scanner page. Context snippet = Current role, Work History headline, Eval Summary, Signals (cap ~500 chars). Extraction should be founder-signal-weighted.

Apply the taxonomy:

```
{
  "founder_signal": [...],
  "market": [...],
  "biz_model": [...],
  "positioning": [...],
  "valuation": [...],
  "other": [...]
}
```

Rules:
- Empty arrays for dimensions Tom didn't touch.
- One nugget = one self-contained sentence or short phrase.
- Preserve Tom's wording (lightly trim "um", "like"). Don't sanitize away specifics.
- `scope == "neg1"` retros typically lean on `founder_signal`. Don't force content into other dimensions.

### 3f. Log raw retro to the source page

Append to the source page via `notion-update-page`:

```
## Retro (YYYY-MM-DD)

<raw retro text verbatim>
```

Page ID is `opp_id` in the queue entry (historical name — for `scope == "neg1"` this is a -1 Scanner page ID).

### 3g. Log nuggets to DECISION_RETROS.md

Path: `/Users/tomseo/.claude/skills/neg1-enricher/DECISION_RETROS.md`

For each non-empty taxonomy array, append entries under the matching H2:

| Taxonomy key | H2 section |
|---|---|
| founder_signal | `## Founder signals` |
| market | `## Market` |
| biz_model | `## Business model` |
| positioning | `## Positioning / moat` |
| valuation | `## Valuation & terms` |
| other | `## Other` |

Entry format (one per nugget):
```
- **YYYY-MM-DD · <Name> · <Decision>** — <nugget text>
  - Source: <opp_url>
```

For `scope == "neg1"`, prepend `-1:` to the Decision label:
```
- **YYYY-MM-DD · Elsie Kenyon · -1: Outreach** — <nugget text>
  - Source: <-1 Scanner page URL>
```

If a section header doesn't exist yet, add it alphabetically before appending.

### 3h. Mark queue entry done

Update the queue entry in place:

```json
{
  ...,
  "status": "completed",
  "completed_at": "<ISO-8601 now>",
  "nugget_count": <total across all taxonomy arrays>
}
```

Save `queue.json` atomically (write tmp, rename).

---

## Notes

- **Bot identity filter:** the `claude` Slack app (incoming webhook) appears as a bot. Filter by `bot_id` / `subtype`. Only real user messages count.
- **Re-editing replies:** if Tom edits or adds to a thread after Mode B already flipped it to `completed`, the update is NOT reprocessed. To re-extract, manually revert `status` to `prompted` in the queue and let Mode A pick it up.
- **Extraction happens in-session** (Claude doing the work directly) — no separate API call.
