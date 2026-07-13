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

### Special branch: Coop finances ingest / classify

Check this **before** First-Pass Diligence and the generic taxonomy. If the parent alert header contains `📒 Coop finances` (from the monthly prompt by `coop-finances-prompt` or a flag-summary from a prior run), route to one of two sub-flows based on the reply shape:

**Sub-flow A — files attached (`args.files` non-empty).**

This is Tom dropping the monthly Citi CSV and/or Reserve account screenshot/xls.

1. For each file in `args.files`:
   - Download: `curl -sSL -H "Authorization: Bearer $SLACK_USER_TOKEN" "<url_private>" -o /tmp/coop_<reply_ts>_<name>` (require `SLACK_USER_TOKEN` env; if missing, post `⚠️ SLACK_USER_TOKEN not set — can't download attachment` and exit).
   - Classify by name + mimetype:
     - `.csv` containing header `Status,Date,Description,Debit,Credit` → operating-account CSV → use as `--csv`
     - `.csv` containing header `date,type,amount,balance` → reserve CSV → use as `--reserve-csv`
     - `.xls` / `.xlsx` → reserve xls (parse first; if it has the Citi columns, treat as operating)
     - `.png` / `.jpg` → reserve screenshot. Read with vision, transcribe to a temp reserve CSV (`date,type,amount,balance` — one row per visible INTEREST line), then use as `--reserve-csv`.
2. Invoke the mutator:
   ```bash
   python3 ~/.claude/skills/coop-finances/update_pl.py \
     [--csv /tmp/coop_<ts>_citi.csv] \
     [--reserve-csv /tmp/coop_<ts>_reserve.csv]
   ```
   Capture stdout (JSON: `posted`, `flagged`, `summary`).
3. Format the close-loop reply:
   ```
   📒 Coop finances updated
   • <N> cells written
   • <M> items flagged for review:
     - <flag.reason> ($<amount> on <date>)
     - ...

   Reply `CHECK <NNNN> = <Category>` or `VENDOR <substring> = <Category>` to classify any flagged items.
   Workbook: 25_Garden_Place_PL.xlsx (iCloud)
   ```
   Post via `post_close_loop.sh`. Skip Step 3 (generic taxonomy). Audit intent: `coop-finances-ingest`.

**Sub-flow B — text classification (no files).**

Tom is replying to a prior flag summary with a classification. Parse patterns:

- `CHECK <N> = <Category>` (case-insensitive, optional spaces) → append a row to `~/.claude/skills/coop-finances/references/CHECK_LEDGER.md` in the existing table format: `| <N> | <today> | <amount-if-known-from-parent-alert> | <Category> | confirmed via Slack |`. If the same `<N>` is already in the ledger with a different category, REPLACE the row rather than append.
- `VENDOR <substring> = <Category>` → append a new rule row to `~/.claude/skills/coop-finances/references/VENDOR_CLASSIFICATIONS.md` under the appropriate section (EXPENSES — Recurring auto-classify, in most cases). Also append a matching entry to the `VENDOR_RULES` Python list in `~/.claude/skills/coop-finances/update_pl.py` — find the list and insert before the closing `]`, preserving formatting.
- Multiple classifications can appear in one reply (one per line). Process each.

After applying:
1. Re-invoke `update_pl.py` to apply the new classifications retroactively (no new CSV needed — `update_pl.py` reads the CSV last dropped, which... actually requires holding state). **Workaround for v1**: just skip the re-run. The next monthly drop will pick up the new classifications. Note in the close-loop: `(will apply on next CSV drop)`.
2. Post close-loop:
   ```
   ✅ Classified:
   • CHECK <N> → <Category>
   • VENDOR <substring> → <Category>
   ```
3. Audit intent: `coop-finances-classify`.

After handling either sub-flow, skip Step 3 (generic taxonomy) and proceed to Step 5 (audit log).

---

### Special branch: First-Pass Diligence feedback

Check this **before** the generic taxonomy. If the parent alert header starts with `🪏` AND
contains `First Pass Diligence:`, route the reply to `first-pass-diligence/FEEDBACK_PATTERNS.md`
instead of attempting a SKILL.md edit. First-pass feedback is soft taste correction — the
kind that compounds over time but rarely warrants an absolute rule edit.

**Routing rules:**

1. **Default route → FEEDBACK_PATTERNS.md.** Most first-pass feedback is texture: "the Tuor
   analog was forced", "Section 4 prose was too dense", "skip regulatory framing when sector
   isn't actually regulated". These append as new entries (newest on top) at the top of
   `/Users/tomseo/.claude/skills/first-pass-diligence/FEEDBACK_PATTERNS.md`, immediately
   after the header HTML comment block.

2. **Escalate to SKILL.md edit ONLY when Tom uses absolute-rule language.** Phrases like
   "never X", "always Y", "from now on", "stop including X" signal an absolute rule. In
   those cases, edit the appropriate prescriptive file (`first-pass-diligence/SKILL.md`,
   `FOUNDER_EVAL_FRAMEWORK.md`, or memory) instead of FEEDBACK_PATTERNS. When in doubt,
   route to FEEDBACK_PATTERNS — absolute rules will surface naturally through repetition
   (Step 3 promotion logic below).

3. **Ack-only replies** ("good catch", "perfect", "thanks") get a `✅ noted, no change needed`
   close-loop and no file write. Tom said he won't send filler replies, so this is rare.

**FEEDBACK_PATTERNS.md entry format** (append immediately below the header HTML comment
block — newest on top):

```markdown
## YYYY-MM-DD — <Company> — <one-line tag>

**Feedback:** "<Tom's reply, verbatim>"

**Pattern:** <generalized rule the listener extracted, 1-2 sentences>

**How to apply:** <when/where this should kick in on future runs, 1-2 sentences>

---
```

The `<Company>` value comes from the parent alert's first line (between `First Pass
Diligence:` and the opening `(`). The `<one-line tag>` is your 3-5 word characterization
of the pattern type (e.g., "forced-analog", "section-density", "regulatory-overreach").

**Promotion-on-repetition.** Before appending, grep `FEEDBACK_PATTERNS.md` for related
existing entries. If the same Pattern shows up in 3+ prior entries, include in the
close-loop reply: `📌 also: this pattern has appeared N times — consider promoting to
SKILL.md as a hard rule?` Don't promote unilaterally — Tom decides.

**Close-loop reply format for this branch:**

```
✅ logged to FEEDBACK_PATTERNS.md — <one-line summary of pattern>
```

Or if also flagging promotion candidacy:

```
✅ logged to FEEDBACK_PATTERNS.md — <one-line summary of pattern>
📌 this pattern has now appeared 3 times — consider promoting to SKILL.md as a hard rule?
```

After handling, skip Step 3 (generic taxonomy) and proceed to Step 4 (close-loop) and
Step 5 (audit log, intent tag: `first-pass-feedback`).

---

### Special branch: Three-way intro confirmation

Check this **before** the generic taxonomy and the NEW DEAL branch. If the parent alert starts with `❓ Three-way intro signal from` (`gmail-webhook/slack-alerts.js → sendUnclassifiedIntroAlert`) or `📩 Three-way intro from` (`sendUnmatchedIntroAlert`), AND Tom's reply confirms an intro context, flip the resolved Opp's Status → **Connected** without asking.

**Why this branch exists:** Tom flagged on 2026-05-28 that asking "what status would you like to move it to?" after a confirmed intro is unnecessary friction (memory: `feedback_intro_confirmed_status_connected.md`). Confirmed three-way intro = `Connected`. Period. **Never** post a "what status?" clarifying question on these alerts — even if the Opp is already at Outreach, Track, or any non-terminal status. The only acceptable clarifying question is on Opp resolution ("which Opp did you mean?"), never on status semantics.

**Confirmation triggers — Tom's reply matches any of these patterns (case-insensitive):**

- `added emails? to (this )?opp`
- `(just )?intro'?d us`
- `<name> intro'?d us`
- `connect(ed)?` (bare word)
- `confirmed?` + (intro context implied by parent alert)
- `this is <CompanyName>` / `it's <CompanyName>`
- `update status` + any of the above in the same reply or thread

If Tom's reply is genuinely ambiguous (e.g. "ignore this" or "wrong opp"), fall through to the generic taxonomy — don't force a Connected flip.

**Resolution path for the target Opp** (try in order, stop on first hit):

1. **By sender email in Contact alias list.** Parse `sourceEmail` from the alert: regex `from\s+([\w.+-]+@[\w.-]+)` against the first line. Query Notion Opportunities DB for an Opp whose `Contact` property contains `sourceEmail` (`mcp__claude_ai_Notion__notion-query-database-view` with a `contains` filter — Contact is a `; `-separated alias list per memory `feedback_upgrade_opp_contact_to_custom_domain_asap.md`).
2. **By founder email in Contact alias list.** If the alert includes a `Likely founder: <email>` line, repeat the search with that email.
3. **By company name from Tom's reply.** If Tom named a company (`this is Acme` / `Acme intro'd us`), search by Opp name.
4. **By prior thread context.** If Tom's earlier replies in the thread reference an Opp by name or URL, use that.

If exactly one Opp matches → that's the target. If multiple match → post a clarifying question naming the candidates and exit. If zero match → post `⚠️ couldn't resolve Opp from intro confirmation — <one-line reason>` and exit.

**Action once Opp is resolved:**

1. Flip Status → `Connected` via `mcp__claude_ai_Notion__notion-update-page` on the Status property only. Do not touch other fields.
2. Close-loop reply (Step 4 format):
   ```
   ✅ done — <OppName> Status → Connected
   ```
3. Audit intent tag: `intro-confirmation`.

After handling, skip Step 3 (generic taxonomy) and proceed to Step 5 (audit log).

---

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

### Special branch: SOI mark confirm

Check this **before** the generic taxonomy. If the parent alert header starts with `📈 Priced round —`
or `💸 Exit —` (drafted by the `soi-portfolio-event` skill), Tom is confirming or adjusting a proposed
SOI mark/distribution. Route here.

1. **Parse the parent alert** for the company and the proposed values:
   - Priced round: `ownership` (FD %, e.g. `6.2%`), `post-money` ($), and the derived `fair value` ($).
   - Exit: `cash distributed` ($) and `residual NAV` ($).
2. **Parse Tom's reply:**
   - `confirm` (or "approved", "approve", "yes", "lgtm", "ship it") → apply the proposed values verbatim.
   - An adjustment → override the named field(s), then RE-DERIVE: for a priced round
     `fair value = ownership × post-money`. Examples: `ownership 6.0%`, `post-money 38m`,
     `fair value 2.4m`, `distribution 1.2m`, `residual 0`. Parse `m`/`mm` = millions, `k` = thousands,
     `%` = decimal.
   - Anything ambiguous / a question → reply asking for clarification, do NOT write.
3. **Apply** with the engine (always `--dry-run` first, confirm the diff is what Tom asked), then rebuild:
   ```bash
   # priced mark
   python3 ~/code/lp-portal/refresh_inputs.py mark --company "<C>" --ownership <decimal> --fmv <int>
   # exit / distribution
   python3 ~/code/lp-portal/refresh_inputs.py distribution --company "<C>" --amount <int> --residual-fmv <int>
   cd ~/code/lp-portal && bash run.sh
   ```
4. **Close-loop reply** in-thread: the applied values, the company's new MOIC, and the new fund DPI/TVPI
   (from `refresh_inputs.py show`). Deliver the rebuilt `~/Inverted_Capital_I_SOI.html` path. Skip Step 3
   (generic taxonomy). Audit intent: `soi-mark-confirm`.

Never write portfolio facts to Notion here — only `fund_inputs.json` via the engine. Marks are valuation
judgments: if the cap-table read in the parent looks off, flag it rather than applying silently.

---

### Special branch: SOI rebuild confirm (Tier 1 gate)

Check this **before** the generic taxonomy. If the parent alert header starts with
`🧾 **SOI rebuild pending your confirm**` (posted by `soi_notify.py --draft` via the daily sweep, or by
`soi-portfolio-event` Tier 1), Tom is gating a plain rebuild — Notion-derived values only, no valuation
judgment. Route here.

1. **Parse Tom's reply:**
   - `confirm` (or "approved", "approve", "yes", "lgtm", "ship it", "publish") → publish.
   - **A value fix (e.g. "Factir should be 750k", "the cap is 15m not 12.5m") → he's correcting the
     NOTION data, and stating the value IS the instruction.** Apply it: patch the named field on the
     Opp (`Inv @ Round` for check size, `Round Details` for cap/post — SCOPE: only Opps in
     {Active Portfolio, Portfolio: Follow-On, Exited}, never Committed/other), reply in-thread with the
     field diff you wrote, and DON'T publish — the edit fires the webhook, which re-drafts with corrected
     numbers for his approval. If the reply is ambiguous about which field/Opp he means, ask in-thread
     instead of guessing.
   - A question → answer it in-thread from the draft's old→new lines + methodology, don't publish.
   - Anything else (feedback on format, skill behavior, "stop including X") → generic taxonomy applies:
     fix the skill/code/memory, close-loop.
2. **Publish** on confirm: `cd ~/code/lp-portal && bash run.sh` (plain — no `--draft`; gates must pass).
   The post-publish `📊 SOI updated` alert doubles as the applied record.
3. **Close-loop reply** in-thread: confirm published + deployed, restate the headline changes (invested /
   FMV / MOIC), link `~/Inverted_Capital_I_SOI.html`. Audit intent: `soi-rebuild-confirm`.

No `fund_inputs.json` writes on this branch — Tier 1 is pure Notion-derived recompute. If the draft's
numbers look wrong (e.g. a fat-fingered Inv @ Round produced a wild swing), say so in the close-loop
rather than publishing silently even on "confirm".

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
