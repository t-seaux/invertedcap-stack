---
name: decision-retro
description: >-
  Capture Tom's free-form retro on any invest/pass decision — including cold dismissals.
  Supersedes the old `founder-feedback` placeholder with broader scope: retros can touch
  founder signals, company stage, market structure, business model, positioning, valuation,
  anything Tom has texture on. Raw retro stays on the Opportunity page; extracted nuggets
  accumulate in a thematic master log at `neg1-enricher/DECISION_RETROS.md`. Strong
  patterns eventually get promoted from retros → `FOUNDER_CALIBRATION.md` through manual curation.
  Two trigger paths. (1) Scheduled scan (daily) detects Opportunity status transitions to
  Invested / Pass / Pass (Met) / Pass Note Pending and pings Tom via Slack with a "retro
  on [company]?" prompt. (2) Manual trigger: Tom says "retro on [company]", "log retro",
  "decision retro on X", "reflection on why I invested/passed on X", or any variant
  referencing post-hoc reasoning on a specific Opportunity. Trigger phrases include:
  "retro", "decision retro", "retro on [company]", "log retro", "reflection on [company]",
  "why did I pass on [X]", "why did I invest in [X]", "founder-feedback" (legacy name
  still routes here). Never asks confirmation — always captures. Distinct from
  `first-pass-diligence` (which runs BEFORE the decision) and from `pass-note-drafter`
  (which produces outbound comms to founders). This skill captures INTERNAL reasoning
  that never leaves the system. Free-form — never imposes a template.
---

# Decision Retro

Captures Tom's post-hoc reasoning on every invest/pass decision and routes the raw retro + extracted nuggets into the feedback corpus that drives framework evolution (per FRAMEWORK_PRD.md §6.6, §12.1).

## Scope

Every Opportunity transition into one of these terminal states triggers a retro prompt:
- `Invested` / `Active Portfolio`
- `Pass` / `Pass (Met)` / `Pass Note Pending` (includes cold dismissals — yes, one-sentence retros are fine)

Manual trigger works the same way, independent of Status.

## Workflow

### Step 1: Detect (scheduled mode)

Runs once daily (morning). Query Opportunities DB for rows where:
- Status recently changed to any terminal state (last 24h), AND
- No `Retro Captured` flag on the page yet.

For each match, send a Slack DM to Tom with:
- Company name + new Status
- A short prompt ("Quick retro? What drove the call? What texture didn't make it into the pass note?")
- Deep link to the Opportunity page

Tom can respond in thread or click through and write on the Opportunity.

### Step 2: Capture (both modes)

Tom's retro is **strictly free-form**. No template, no required fields. Length scales to stakes. Capture verbatim — don't "clean up" or restructure.

Write raw retro as a callout block on the Opportunity page:
```
## Retro (2026-04-22)

[verbatim retro text]
```

### Step 3: Extract and route (LLM post-pass)

Read the raw retro. For each thematic nugget, append to the appropriate section of `/Users/tomseo/.claude/skills/neg1-enricher/DECISION_RETROS.md`:

- `## Founder signals` — insights about the founder archetype, motivations, track record, trajectory
- `## Market` — market structure, regulatory dynamics, TAM questions, competitive landscape
- `## Company` — stage, team, ops, execution quality
- `## Business model` — unit economics, GTM motion, pricing, take rate, moat dynamics
- `## Other` — anything that doesn't fit above (valuation gripes, process issues, timing)

Each entry format:
```
- **2026-04-22 · [Company] · [Invested/Pass]** — [one-line nugget extracted from retro]
  - Source retro: [link to Opportunity page]
```

A single retro can contribute nuggets to multiple sections. That's expected and fine.

### Step 4: Flag on Opportunity

Set `Retro Captured = true` on the Opportunity page so the scheduled scanner skips it next run.

### Step 5: Report

Single-line Slack confirmation: "Retro captured for [Company] ([N] nuggets across [sections])."

## Important rules

- **Never impose a template** on the raw retro. The point is capturing what Tom wouldn't otherwise write down.
- **Extracted nuggets quote the raw retro where possible** — don't paraphrase away the texture.
- **Cold dismissal retros are fully valid**. A one-sentence "wrong GTM motion for this market" is a legitimate retro. Don't prompt for more.
- **Never prompt twice for the same Opportunity**. Idempotency via the `Retro Captured` flag.
- **Promote to CALIBRATION manually.** This skill does NOT write to `FOUNDER_CALIBRATION.md` directly. Patterns solidify into calibration through Tom's explicit curation call.

## Consumption (downstream)

- `neg1-enricher` reads `DECISION_RETROS.md` alongside `FOUNDER_CALIBRATION.md` during Step 5 (rubric application). Retros surface as soft priors in the Eval Breakdown.
- Quarterly `--score-only` drift check (per FRAMEWORK_PRD.md §6.7 + Future State item 14) uses accumulated retros as evidence for whether signal anchors need retuning.

## Example

Tom pastes into Slack DM response: "Passed on Acme. Founder was smart but the market is fundamentally consumer-subsidized B2B — anyone who's not paying for their employer's version will churn the day they change jobs. Saw this exact pattern with X and Y."

Output:
- Opportunity page `## Retro (2026-04-22)` block populated verbatim.
- `DECISION_RETROS.md`:
  - `## Market` — new entry: `**2026-04-22 · Acme · Pass** — Consumer-subsidized B2B where churn is triggered by employer changes is a recurring trap pattern. Compare X and Y.`
  - `## Business model` — new entry: `**2026-04-22 · Acme · Pass** — Individual-pay-after-employer-pay GTM motion has hidden churn risk.`
- Slack: "Retro captured for Acme (2 nuggets across Market, Business model)."
