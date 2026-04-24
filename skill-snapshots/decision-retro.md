---
name: decision-retro
description: >-
  Capture Tom's free-form retro on any invest/pass decision — including cold dismissals.
  Supersedes the old `founder-feedback` placeholder with broader scope: retros can touch
  founder signals, company stage, market structure, business model, positioning, valuation,
  anything Tom has texture on. Raw retro stays on the Opportunity page; extracted nuggets
  accumulate in a thematic master log at `neg1-enricher/DECISION_RETROS.md`. Strong
  patterns eventually get promoted from retros → `FOUNDER_CALIBRATION.md` through manual curation.
  Two trigger paths. (1) Scheduled scan (daily 9am ET) detects Opportunity status transitions
  to Invested / Pass / Pass (Met) / Pass Note Pending and posts a prompt to the
  `#decision-retros` Slack channel; Tom replies in thread (voice-to-text or typed);
  scheduled listener (daily 6pm ET) processes replies. (2) Manual trigger: Tom says
  "retro on [company]", "log retro", "decision retro on X", "reflection on why I
  invested/passed on X", or any variant referencing post-hoc reasoning on a specific
  Opportunity. Trigger phrases include: "retro", "decision retro", "retro on [company]",
  "log retro", "reflection on [company]", "why did I pass on [X]", "why did I invest in [X]",
  "founder-feedback" (legacy name still routes here). Never asks confirmation — always
  captures. Distinct from `first-pass-diligence` (which runs BEFORE the decision) and
  from `pass-note-drafter` (which produces outbound comms to founders). This skill
  captures INTERNAL reasoning that never leaves the system. Free-form — never imposes a template.
---

# Decision Retro

Captures Tom's post-hoc reasoning on every invest/pass decision and routes the raw retro + extracted nuggets into the feedback corpus that drives framework evolution (per FRAMEWORK_PRD.md §6.6, §12.1).

## Scope

Every Opportunity transition into one of these terminal states triggers a retro prompt:
- `Invested` / `Active Portfolio`
- `Pass` / `Pass (Met)` / `Pass Note Pending` (includes cold dismissals — yes, one-sentence retros are fine)

Manual trigger works the same way, independent of Status.

## Architecture (as of 2026-04-23)

**State is headless** — lives in `~/.claude/skills/decision-retro/queue.json`, not in Notion. No `Retro Captured` property or similar on the Opportunity page. This keeps the Opp schema clean and makes the retro flow easy to rip out or rewire.

Two scheduled agents:

- **`decision-retro-scan`** (daily 9am ET) — queries Notion for Opps in terminal status, dedups against `queue.json`, posts new prompts to `#decision-retros` as the `claude` Slack app, records each prompt's thread `ts` in the queue.
- **`decision-retro-listener`** (daily 6pm ET) — for each `prompted` item in the queue, reads the Slack thread, treats concatenated Tom replies as the retro, runs the extractor, logs to Opp page + `DECISION_RETROS.md`, marks `completed`. Items prompted >7 days ago with no reply auto-close as `no_retro`.

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
      "status": "prompted",           // "prompted" | "completed" | "no_retro"
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
2. Query Notion Opportunities DB for rows where Status ∈ {`Invested`, `Active Portfolio`, `Pass`, `Pass (Met)`, `Pass Note Pending`}. For each row capture `Website`, `Description`, and `🏁 Founder(s)` alongside `id/name/url/status`. Resolve each Founder relation → People page for name + `LI` URL.
3. For each Opp whose `id` is NOT in the queue:
   - Compose GFM prompt:
     ```
     **[Opp Name]** | [Website] | [Notion](url)

     - Description: [Description]
     - Founder: [Founder Name](LI URL)
     - Decision: [Status]

     Reply in this thread — voice-to-text fine, one sentence is fine too.

     [opp:<short-opp-id>]
     ```
     Fallbacks: `N/A` for missing Website/Founder, `TBD` for missing Description. Website shows as `[domain](full_url)` (e.g. `[rengo.com](https://rengo.com)`). Founder hyperlink omitted if no LI URL. Multiple founders comma-separated.
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

### Manual trigger

When Tom says "retro on X" in an interactive session:
1. Locate the Opp in Notion.
2. Ask him for the retro text inline (or use whatever he just said as the retro).
3. Run Steps 2.5–2.9 directly (extract, log, report).
4. If the Opp was already in the queue (`prompted`), update to `completed` so the scheduled listener skips it.

## Important rules

- **Never impose a template** on the raw retro. The point is capturing what Tom wouldn't otherwise write down.
- **Extracted nuggets quote the raw retro where possible** — don't paraphrase away the texture.
- **Cold dismissal retros are fully valid**. A one-sentence "wrong GTM motion for this market" is a legitimate retro. Don't prompt for more.
- **Never prompt twice for the same Opportunity** — idempotency is enforced by queue.json `opp_id` dedup.
- **Promote to CALIBRATION manually.** This skill does NOT write to `FOUNDER_CALIBRATION.md` directly. Patterns solidify into calibration through Tom's explicit curation call.
- **Status re-transitions are one-shot.** If Tom flips an Opp Pass → Invested, the queue's existing `prompted` / `completed` entry stays — no re-prompt. Edge case; handle manually via Tom saying "retro on X" if he wants another pass.

## Consumption (downstream)

- `neg1-enricher` reads `DECISION_RETROS.md` alongside `FOUNDER_CALIBRATION.md` during Step 5 (rubric application). Retros surface as soft priors in the Eval Breakdown.
- Quarterly `--score-only` drift check (per FRAMEWORK_PRD.md §6.7 + Future State item 14) uses accumulated retros as evidence for whether signal anchors need retuning.

## Example

Tom flips `Acme Corp` → `Pass` on 2026-04-22. The 9am scan on 2026-04-23 posts to `#decision-retros`:

> **Acme Corp** | [acme.com](https://acme.com) | [Notion](https://notion.so/tom/acme-...)
>
> - Description: Consumer-subsidized B2B productivity suite.
> - Founder: [Jane Doe](https://linkedin.com/in/janedoe)
> - Decision: Pass
>
> Reply in this thread — voice-to-text fine, one sentence is fine too.
> `[opp:abc12345]`

Tom replies in thread (voice-to-text via Wispr Flow): "passed on Acme. founder was smart but the market is fundamentally consumer-subsidized B2B — anyone who's not paying for their employer's version will churn the day they change jobs. saw this exact pattern with X and Y."

6pm listener processes:
- Opp page gets `## Retro (2026-04-23)` block with the raw text.
- `DECISION_RETROS.md`:
  - `## Market` — `**2026-04-23 · Acme · Pass** — Consumer-subsidized B2B where churn is triggered by employer changes is a recurring trap pattern. Compare X and Y.`
  - `## Business model` — `**2026-04-23 · Acme · Pass** — Individual-pay-after-employer-pay GTM motion has hidden churn risk.`
- Slack DM: `🔁 Retro captured for Acme (2 nuggets across Market, Business model).`
