---
name: decision-retro
description: >-
  Capture Tom's free-form retro on any invest/pass decision — including cold dismissals.
  Supersedes the old `founder-feedback` placeholder with broader scope: retros can touch
  founder signals, company stage, market structure, business model, positioning, valuation,
  anything Tom has texture on. Raw retro stays on the Opportunity page; extracted nuggets
  accumulate in a thematic master log at `neg1-enricher/DECISION_RETROS.md`. Strong
  patterns eventually get promoted from retros → `INVERTED_LENS.md` through manual curation.
  Three trigger paths. (1) Scheduled scan (daily 9am ET) detects Opportunity status transitions
  to Committed / Pass Note Pending / Pass (Met) / Pass (DNM) (including follow-on rounds) and posts a prompt to the
  `#decision-retros` Slack channel; Tom replies in thread (voice-to-text or typed);
  scheduled listener (daily 6pm ET) processes replies. (2) Capture mode (manual): Tom says
  "retro on [company]", "log retro on X", "decision retro on X", "log this retro",
  or any imperative variant indicating he's providing new retro text for an Opp.
  (3) Lookup mode (manual): Tom asks a past-tense question about a prior decision —
  "why did I pass on [X]", "why did I invest in [X]", "what did I think of [X]",
  "pull my retro on [X]", "show me my feedback on [X]", "my take on [X]". Lookup
  returns a structured summary (header / official pass note feedback / internal-feedback
  nuggets by dimension / framework connections / diligence context / Notion link) —
  never echoes the raw retro verbatim. Distinct from `first-pass-diligence` (which
  runs BEFORE the decision) and from `pass-note-drafter` (which produces outbound
  comms to founders). Capture mode writes INTERNAL reasoning that never leaves the
  system; Lookup mode surfaces it on demand. Free-form — never imposes a template.
---

# Decision Retro

Captures Tom's post-hoc reasoning on every invest/pass decision and routes the raw retro + extracted nuggets into the feedback corpus that drives framework evolution (per FOUNDER_EVAL_FRAMEWORK.md §6.6, §12.1).

## Scope

Every Opportunity transition into one of these terminal states triggers a retro prompt:
- **Invest side**: `Committed` (the moment of decision; `Active Portfolio` is post-close operational state — no separate retro)
- **Pass side**: `Pass Note Pending`, `Pass (Met)`, `Pass (DNM)` (includes cold dismissals — yes, one-sentence retros are fine)

Follow-on Opps (`(FO)` / `(Series X FO)` in title) are NOT skipped — each FO is a separate Notion row with its own ID, treated as a separate retro entry by the queue dedup. Each follow-on round is a real new decision; longitudinal retros across rounds are valuable calibration data.

The four trigger statuses act as a priority cascade: the queue dedups by Opp ID, so each Opp gets one retro at its earliest terminal-status transition (e.g., `Pass Note Pending` fires first; subsequent `Pass (Met)` is skipped because the Opp is already in the queue). The other statuses serve as backstops if an Opp ever bypasses an earlier state.

Manual trigger works the same way, independent of Status.

## Architecture (as of 2026-04-23)

**State is headless** — lives in `~/.claude/skills/decision-retro/queue.json`, not in Notion. No `Retro Captured` property or similar on the Opportunity page. This keeps the Opp schema clean and makes the retro flow easy to rip out or rewire.

Two scheduled agents:

- **`decision-retro-scan`** (daily 9am ET) — queries Notion for Opps in terminal status, dedups against `queue.json`, posts new prompts to `#decision-retros` as the `claude` Slack app with fingerprint `[opp:<short>]`, records each prompt's thread `ts` in the queue with `scope="opp"`.
- **`neg1-retro-scan`** (daily 9:05am ET) — mirror for the `-1 Scanner` DB. Queries rows where `Status ∈ {Passed, Outreach}`, posts prompts with fingerprint `[neg1:<short>]` and a person-shaped body (🧍 Current / Work History / Education / Status), records with `scope="neg1"`.
- **`decision-retro-listener`** (daily 6pm ET) — for each `prompted` item in the queue, reads the Slack thread, handles both `[opp:]` and `[neg1:]` fingerprints, treats concatenated Tom replies as the retro, runs the extractor (founder-signal-weighted for `scope="neg1"`), logs to source page + `DECISION_RETROS.md`, marks `completed`. Items prompted >7 days ago with no reply auto-close as `no_retro`. `skip`/`ignore`/etc. replies mark as `skipped` and are excluded from framework feed.
- **`retro-weekly-summary`** (Fri 4pm ET) — rolls up the week, groups by `scope` (Company retros / Founder-signal retros), posts synthesis to `#decision-retros`.

Helper: `send-alert/md_to_blocks.py` posts to the channel webhook at `~/.claude/skills/decision-retro/.webhook_url` (distinct from the DM webhook used by `send-alert` itself).

## Queue state (`queue.json`)

```json
{
  "items": [
    {
      "opp_id": "notion-page-uuid",
      "opp_name": "Acme Corp",
      "opp_url": "https://notion.so/...",
      "decision": "Pass",
      "scope": "opp",                 // "opp" (Opportunities DB) | "neg1" (-1 Scanner DB). Absent = "opp" for legacy entries.
      "short_id": "abc12345",
      "status": "prompted",           // "prompted" | "completed" | "skipped" | "no_retro"
      "thread_ts": "1745432123.456789",
      "prompted_at": "2026-04-23T09:00:00-07:00",
      "completed_at": null,
      "nugget_count": null
    }
  ]
}
```

Queue grows over time (acceptable — small file). No pruning required.

## Workflow

### Step 1 — Scan + prompt (9am ET)

1. Load `queue.json` (create if absent).
2. Query Notion Opportunities DB for rows where Status ∈ {`Committed`, `Pass Note Pending`, `Pass (Met)`, `Pass (DNM)`}. Include follow-on rows (titles with `(FO)` / `(Series X FO)`) — they are NOT skipped; each FO is a separate Opp ID treated as a separate retro entry. For each row capture `Website`, `Description`, and `🏁 Founder(s)` alongside `id/name/url/status`. Resolve each Founder relation → People page for name + `LI` URL.
3. For each Opp whose `id` is NOT in the queue:
   - Compose GFM markdown prompt (`[text](url)` for links, `**text**` for bold). `md_to_blocks.py` converts to Slack Block Kit — do NOT hand-write Slack mrkdwn (`<url|text>` / `*text*`), it ships as literal text / renders italic. Per memory `feedback_scheduled_alert_structure.md`:
     ```
     🏢 **{Opp Name}** ([{domain}]({Website})) | [Notion]({Notion url})

     - **Description:** {Description}
     - **Founder(s):** [{Founder Name}]({LI URL}), ...
     - **Status:** {Status}

     [opp:<short-opp-id>]
     ```
     Rules:
     - Opp Name is bold (`**{Name}**`); domain is wrapped in parens with the Website URL embedded as a GFM link
     - If Website is missing: drop the `([...]{...})` parens entirely
     - If Founder LI URL is missing: render founder as plain `{Founder Name}` (no link)
     - Multiple founders: comma-separated
     - Missing Founders → `N/A`; missing Description → `TBD`
     - Bullet labels (`**Description:**`, `**Founder(s):**`, `**Status:**`) are always bolded
     - **Never include a "Reply in this thread…" line** — the channel description already says this
   - POST via `send-alert/md_to_blocks.py` with `WEBHOOK_URL=$(cat ~/.claude/skills/decision-retro/.webhook_url)`.
   - Immediately call `mcp__claude_ai_Slack__slack_read_channel` on `#decision-retros` (limit 10) and find the message by grepping for the `[opp:<short-opp-id>]` fingerprint. Capture `ts`.
   - Append entry to `queue.json` with `status="prompted"`, `thread_ts=ts`, `prompted_at=now()`.

### Step 2 — Listen + extract + log (6pm ET)

For each `items[]` entry where `status == "prompted"`:

1. Check age: if `prompted_at > 7 days ago`, set `status="no_retro"`, continue (no processing).
2. Call `mcp__claude_ai_Slack__slack_read_thread` with the stored `thread_ts` on `#decision-retros`.
3. Filter replies to those from Tom (user ID check). If no replies, continue (leave in queue for next run).
4. Concatenate Tom's replies in thread order → raw retro text.
4a. **Skip check**: if the concatenated reply (trimmed, lowercased, punctuation stripped) matches `^(ignore|skip|pass|n/?a|no thanks|nope|nah|nvm|not today)$`, treat as an explicit skip. Set queue item `status="skipped"`, `completed_at=now()`, `nugget_count=0`. Do NOT run extraction, do NOT write to Opp page, do NOT append to `DECISION_RETROS.md`. Send no Slack alert for skips. Continue to next queue item.
5. **Extract**: Claude call with prompt:
   ```
   Given this raw retro on [Opp Name] ([Decision]) and the context below,
   extract nuggets keyed by framework dimension. Return ONLY valid JSON.

   Context: [...Opp page snippet, prior first-pass if any...]

   Raw retro: [...]

   Output shape:
   {
     "founder_signal": [...nuggets...],
     "market": [...],
     "biz_model": [...],
     "positioning": [...],
     "valuation": [...],
     "other": [...]
   }

   Rules: preserve Tom's exact phrasing where possible; empty arrays for dimensions
   Tom didn't touch; don't invent dimensions not in the schema.
   ```
6. **Log to Opp page** — append a `## Retro (YYYY-MM-DD)` block with the raw reply text verbatim.
7. **Log to `DECISION_RETROS.md`** — append to each relevant thematic section:
   - `## Founder signals` — from `founder_signal` nuggets
   - `## Market` — from `market`
   - `## Company` — from anything not slotted elsewhere, plus `other`
   - `## Business model` — from `biz_model`
   - `## Positioning / moat` — from `positioning`
   - `## Valuation & terms` — from `valuation`

   Entry format:
   ```
   - **YYYY-MM-DD · [Company] · [Invested/Pass]** — [nugget verbatim or lightly paraphrased]
     - Source: [Opportunity page URL]
   ```
8. Mark queue item `status="completed"`, `completed_at=now()`, `nugget_count=<total>`.
9. Single-line alert via `send-alert` to DM: `🔁 Retro captured for [Company] ([N] nuggets across [sections]).`

### Capture mode (manual)

When Tom says "retro on X", "log retro on X", or any imperative indicating he's providing new retro text:
1. Locate the Opp in Notion.
2. Ask him for the retro text inline (or use whatever he just said as the retro).
3. Run Steps 2.5–2.9 directly (extract, log, report).
4. If the Opp was already in the queue (`prompted`), update to `completed` so the scheduled listener skips it.

### Webhook-prompt mode (single-page, immediate)

Triggered by the `claude-job-queue` processor with args `{ mode: "webhook-prompt", scope: "neg1"|"opp", neg1_page_id|opp_page_id, trigger_event }`. Fires when an event upstream (currently: Gmail draft-trash detected by `gmail-webhook`) flips a row to a terminal status and wants an immediate retro prompt — ahead of the next scheduled scan.

**Why it exists:** the 9am (`decision-retro-scan`) and 9:05am (`neg1-retro-scan`) scheduled agents are reconciliation — they catch anything that transitioned to terminal status since the last run. For event-driven terminal transitions (draft trash, future webhook sources), latency to prompt is the signal: the retro is most useful captured while the decision is still fresh. This mode posts the prompt immediately instead of waiting up to 24h.

**Behavior (per invocation):**

1. Load `queue.json`. If an entry with matching `{scope, neg1_page_id}` already exists with `status ∈ {"prompted", "completed", "skipped"}`, exit silently (dedup — the scheduled scan may have beaten us, or this is a retry). Log reason.
2. Fetch the target page via `mcp__claude_ai_Notion__notion-fetch`.
3. Compose the Slack prompt per scope. For `scope="neg1"` use this template exactly (GFM markdown — `[text](url)` for links, `**text**` for bold). `md_to_blocks.py` converts to Slack Block Kit; Slack mrkdwn (`<url|text>`, `*text*`) ships as literal text / renders italic:

   ```
   🧍 [{Name}]({LI URL}) | [Notion]({Notion page url})

   - **Current:** {Role} @ {Company Name}
   - **Work History:** {Company 1}, {Company 2}, {Company 3}
   - **Education:** {School 1}, {School 2}
   - **Status:** {Status}

   [neg1:<short-id>]
   ```

   Rules:
   - `<short-id>` = first 8 chars of the Notion page UUID
   - If LI URL is empty: render name as plain `**{Name}**` (no link)
   - Drop Work History / Education lines entirely if empty (do NOT emit the label with blank value)
   - Work History: up to 3 companies, comma-separated, pulled from Work History relation
   - Education: comma-separated, pulled from School(s) relation
   - Status bullet is just the Status value — no trailing "flipped {date}" clause
   - Header is just name + Notion link — do NOT repeat the status (e.g. `— -1: Passed`) since it's in the bullet

   For `scope="opp"` use the Opp format from Step 1 of the 9am scan (lines 82–96 above).
4. POST via `send-alert/md_to_blocks.py` using the `#decision-retros` webhook URL at `~/.claude/skills/decision-retro/.webhook_url`.
5. Immediately `slack_read_channel` on `#decision-retros` (limit 10) to find the message by fingerprint; capture `ts`.
6. Append a new entry to `queue.json` with `status="prompted"`, `thread_ts=ts`, `prompted_at=now()`, `scope`, page id, page name, page url, decision. Include a `trigger_source: "webhook:<trigger_event>"` field so the analytics layer can distinguish webhook-triggered from schedule-triggered prompts.
7. The existing `decision-retro-listener` (6pm ET) will process the reply when Tom threads a retro — no change to that path.

**Idempotency:** the upstream webhook must pass `idempotencyKey = "draft-trash-retro-{messageId}"` (or analogous) so `claude-job-queue` dedupes re-triggers. Plus the step-1 queue check inside this skill is a second layer against the scheduled scan firing first.

**Failure:** if the page is no longer in terminal status by the time the job runs (Tom manually flipped it back), exit silently — log reason. Never post a prompt for a non-terminal row.

### Lookup mode (manual)

When Tom asks a past-tense question about a prior decision — "why did I pass on X", "why did I invest in X", "what did I think of X", "pull my retro on X", "show me my feedback on X", "my take on X" — return a structured summary, never the raw retro verbatim.

**Target resolution:**
- If the question names a **company** ("why did I pass on Dorsal"): search the Opportunities DB (`collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Use the earliest-fund original investment if multiple matches (e.g. `Rengo` + `Rengo (Seed FO)`) — follow-on rows (`(FO)`, `(Seed FO)`, `(A FO)`, `(SPV)`) deprioritize. Use `scope="opp"`.
- If the question names a **person** ("why did I pass on Elsie Kenyon", "what did I think of [person]", especially when paired with `-1`, `scanner`, or a LinkedIn URL): search the -1 Scanner DB (`collection://32c00bef-f4aa-80a5-923b-000b83921fa3`) by Name. Use `scope="neg1"`. For neg1 targets, the output structure changes slightly — see "-1 Lookup variant" below.
- If no match, say so and stop.

**Data sources (in priority order):**
1. **Official pass note** — the outbound email archived by `pass-note-drafter`. Locate via the Opp's `✍️ Notes` relation → filter to entries with `Category = Diligence` whose body contains the pass note text. If not in Notes, fall back to Gmail search for sent mail to the founder around the Close Date. Summarize (do not paste verbatim).
2. **Internal feedback** — the Opp page's `## Retro (YYYY-MM-DD)` block(s), plus any matching entries in `~/.claude/skills/neg1-enricher/DECISION_RETROS.md` sourced to this Opp URL. Distill into dimension-keyed summaries (no raw text quoted).
3. **Diligence context** — the Opp page's first-pass diligence block and/or pre-mortem block, or the linked first-pass page in `✍️ Notes`. One-line summary only.
4. **Framework** — `~/.claude/skills/neg1-enricher/INVERTED_LENS.md` + `FOUNDER_EVAL_FRAMEWORK.md`. Match nuggets to named patterns.

**Output structure** (inline chat response, not Slack):

```
**[Company Name]** — [Decision] · Close Date [YYYY-MM-DD] · Retro logged [YYYY-MM-DD]

**Official pass note feedback** (if applicable)
• [1–3 bullets summarizing what Tom told the founder — tone + substance]

**Internal feedback**
• *Founder signal:* [summary, no verbatim]
• *Market:* [summary]
• *Business model:* [summary]
• *Positioning:* [summary]
• *Valuation:* [summary]
(Omit dimensions with no nuggets. Use bare `~$3-4B`, en dashes only.)

**Framework connections**
• Reinforces: [pattern name from INVERTED_LENS.md] — [one-line why]
• Challenges: [pattern] — [why]
• Expands: [new pattern hint] — [why]

**Diligence context**
• [One-line summary of pre-decision thinking — first-pass priors or pre-mortem headline]

[Notion](url)
```

**Edge cases:**
- **Queue status `prompted` (reply not yet processed)**: don't wait for the 6pm listener. Do an **inline one-off pass** for this single Opp right now:
  1. Read the Slack thread via `mcp__claude_ai_Slack__slack_read_thread` using the stored `thread_ts`.
  2. Filter to Tom's replies, concatenate → raw retro text.
  3. Run Step 4a skip-check (from the scheduled listener logic above). If it matches skip, mark queue item `status="skipped"` and treat as the skipped edge case below.
  4. Otherwise run Steps 5–8 from the listener flow: extract nuggets via Claude, append `## Retro (YYYY-MM-DD)` to the Opp page, append to `DECISION_RETROS.md`, mark queue item `status="completed"` / `completed_at=now()` / `nugget_count=<total>`.
  5. Use the freshly extracted nuggets in the Lookup output. Tell Tom inline: "Processed your Slack reply just now — logged and summarized below."
  6. If the thread has zero replies from Tom, emit the "No reply yet" fallback: header + "No reply logged in Slack yet — thread is here: <slack link to thread>" + whatever Official pass note + Diligence context is available.
- **No retro logged** (`queue.json` has no entry for this Opp at all): emit the header + "No retro logged" in place of Internal feedback, but still surface Official pass note feedback + Diligence context if available.
- **Queue status `no_retro`** (7-day window closed with no reply): same as "No retro logged" — note that no reply came in before the window closed.
- **Queue status `skipped`**: emit the header + "No internal feedback (marked skipped)" in place of Internal feedback. Still surface Official pass note + Diligence context.
- **Invest decisions**: omit "Official pass note feedback" section entirely. Internal feedback + Framework connections + Diligence context still apply.

Lookup is normally read-only. The only write path is the **inline one-off pass** above, which is functionally equivalent to the 6pm listener running early on that single Opp — not a new write path, just timing acceleration.

**-1 Lookup variant** (for `scope="neg1"` targets):

Output structure is adjusted for a person, not a company:

```
🧍 **[Name](LI URL)** — -1: [Passed|Outreach] · Flipped [YYYY-MM-DD] · Retro logged [YYYY-MM-DD]

**Current**
• [Role] @ [Company]

**Work History** (one-line summary, 2–4 bullets max — lean on Experience Summary on the row)
• [Company] ([years], key signal)
• ...

**Internal feedback**
• *Founder signal:* [summary, no verbatim] — this dimension dominates for -1 retros
• *Market / biz model / positioning:* [only if Tom's retro touched these]

**Framework connections**
• Reinforces: [signal pattern] — [one-line why]
• Challenges / Expands: [...]

**Eval context** (replaces "Diligence context" for -1s)
• Eval Score (Spike): [0–1]
• Signals flagged at enrichment: [list from the Signals multi-select]
• Claude Rec at enrichment: [Reach Out / Pass]

[Notion](url)
```

Omit "Official pass note feedback" — -1 rows don't have outbound pass notes. Everything else (skip handling, no-retro handling, inline one-off pass, prompted-but-no-reply) works the same way.

## Important rules

- **Never impose a template** on the raw retro. The point is capturing what Tom wouldn't otherwise write down.
- **Extracted nuggets quote the raw retro where possible** — don't paraphrase away the texture.
- **Cold dismissal retros are fully valid**. A one-sentence "wrong GTM motion for this market" is a legitimate retro. Don't prompt for more.
- **Never prompt twice for the same Opportunity** — idempotency is enforced by queue.json `opp_id` dedup.
- **Promote to CALIBRATION manually.** This skill does NOT write to `INVERTED_LENS.md` directly. Patterns solidify into calibration through Tom's explicit curation call.
- **Status re-transitions are one-shot.** If Tom flips an Opp's status across terminal states (e.g., `Pass Note Pending` → `Pass (Met)`, or `Pass (Met)` → `Committed` — rare), the queue's existing `prompted` / `completed` entry stays — no re-prompt. The cascade fires once per Opp ID at the earliest terminal transition. Edge case; handle manually via Tom saying "retro on X" if he wants another pass.

## Consumption (downstream)

- `neg1-enricher` reads `DECISION_RETROS.md` alongside `INVERTED_LENS.md` during Step 5 (rubric application). Retros surface as soft priors in the Eval Breakdown.
- Quarterly `--score-only` drift check (per FOUNDER_EVAL_FRAMEWORK.md §6.7 + Future State item 14) uses accumulated retros as evidence for whether signal anchors need retuning.

## Example

Tom flips `Acme Corp` → `Pass` on 2026-04-22. The 9am scan on 2026-04-23 posts to `#decision-retros`:

> 🏢 **Acme Corp (acme.com)** | [Notion](https://notion.so/tom/acme-...)
>
> - Description: Consumer-subsidized B2B productivity suite.
> - Founder(s): [Jane Doe](https://linkedin.com/in/janedoe)
> - Status: Pass
>
> `[opp:abc12345]`

Tom replies in thread (voice-to-text via Wispr Flow): "passed on Acme. founder was smart but the market is fundamentally consumer-subsidized B2B — anyone who's not paying for their employer's version will churn the day they change jobs. saw this exact pattern with X and Y."

6pm listener processes:
- Opp page gets `## Retro (2026-04-23)` block with the raw text.
- `DECISION_RETROS.md`:
  - `## Market` — `**2026-04-23 · Acme · Pass** — Consumer-subsidized B2B where churn is triggered by employer changes is a recurring trap pattern. Compare X and Y.`
  - `## Business model` — `**2026-04-23 · Acme · Pass** — Individual-pay-after-employer-pay GTM motion has hidden churn risk.`
- Slack DM: `🔁 Retro captured for Acme (2 nuggets across Market, Business model).`
