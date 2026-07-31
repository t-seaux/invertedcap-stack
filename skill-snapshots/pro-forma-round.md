---
name: pro-forma-round
description: Model a hypothetical (pro forma) financing round for an existing portfolio position. Fixed output contract — company level: ownership, value, and MOIC both blended and by round; fund level: gross MOIC and TVPI. Handles option-pool top-ups, follow-on checks in the modeled round, and dilution decomposition. Trigger when Tom says "calculate pro forma position/ownership/returns for [company]", "contemplate a pro forma round [terms] for [company]", "what's my MOIC if [company] raises [X] on [Y]", "model a round for [company]", "pro forma [company] at [valuation]", "how does a [X] on [Y] round move my fund returns", "what if we put another [amount] into [company]'s round", or any variant asking how a hypothetical round for a portfolio company changes his position or fund returns profile. Manual-only — conversational analysis, writes nothing anywhere unless Tom separately asks.
---

# Pro Forma Round Modeling

Model a hypothetical round for a portfolio company: position value, MOIC (blended + by tranche), and fund returns impact. This is scenario analysis — it never writes to Notion, the SOI, or fund_inputs.json. Output is a chat answer.

## Step 1: Identify the fund and load the position

Determine which fund holds the position (Notion Opp `Fund` property if unsure), then pull current ownership from the canonical source:

**Dash I & II → SOI Google Sheet** `13w1uO3qE04yFDCtnrtA78jPrJ2Z31JJoQiCrbx4uc8k` ("Dash Funds I & II: SOI"). Per company: Current Ownership, Initial/Follow-On/Total Inv Capital, FMV (LRP), MOIC. Fund-level: Total FMV, invested, fund size, called %, gross MOIC / TVPI / DPI (LRP and DVA variants). Use the LRP set for scenario math. The sheet is large — read it via the saved-tool-result file and chunk with python, not raw Read.

**Inverted 1 → lp-portal live model** (`~/code/lp-portal/`). Ownership/FMV/rounds: latest `archive/soi_<date>.json` (or regenerate via `soi_generate.py`). Fund-level (size, called, NAV, financial-statement anchors): `fund_inputs.json`. Do NOT use any static Google Sheet for Inverted — the Notion-generated model is the live view. Respect its conventions: SAFE-only positions are held at cost; priced marks come from `priced_round_marks`; TVPI is gated N/A until 60% called.

**Round-by-round history (both funds) → Notion Opp cards.** Each round is its own Opp linked via the `🕰️ Funding History` self-relation. Per card: `Inv @ Round`, `Round Details` (e.g. `$2.5m on $22.5m post` / `$2.4m on $10m cap`), `OS% @ Round`, Close Date.

**Validation gate (mandatory):** blended ownership × last-round post-money must reproduce the source's FMV for the company (within rounding). If it doesn't tie out, stop and reconcile before modeling — one of the inputs is stale.

**Cap-table hard rule:** the SOI ownership figures are themselves derived from executed pro-forma cap tables, so they are a valid anchor for *hypothetical* scenario math. But if the round being modeled is real and closing (term sheet in hand), the final numbers must come from the actual pro-forma cap table per the priced-round hard rule — label scenario output as indicative and say what it's anchored to.

## Step 2: Decompose ownership by tranche

Blended ownership is exact from the source; the per-tranche split is reconstructed:

1. SAFE tranches: conversion % = invested ÷ post-money cap.
2. Dilute each earlier tranche through every subsequent round by that round's new-money fraction (new $ ÷ post), plus pool top-ups **if the round's pro forma is available**.
3. Priced-round tranches: purchase % = invested ÷ post.
4. Force the sum to equal the source's blended ownership by solving the final priced round's dilution factor as the residual: `f = (blended − last_round_buy%) ÷ pre_round_holdings%`. This absorbs any unrecorded pool top-ups into the last round.

Always state the caveat: the blended total is exact; the tranche *split* is approximate wherever an intermediate round's pool change isn't in hand (shifts a few tenths between adjacent tranches, never the total).

## Step 3: Model the round

Inputs: new money `N`, post-money `POST`, target available pool `p_t` (as stated), current available pool `u`, optional additional check from the fund.

- New-investor fraction: `n = N / POST`.
- **Pool top-up — default convention is post-money measured, pre-money created** (the standard "option pool shuffle": target measured as % of post-round FD cap table, shares created before new money so existing holders bear it). With existing FD normalized to `E = 1`:
  - Pool shares to add: `ΔP = (p_t/(1−n) − u) / (1 − p_t/(1−n))`
  - Existing-holder dilution factor: `d = (1−n) / (1 + ΔP)`
- Alternate (founder-friendly, only if the terms say so): pool measured on pre-money FD → `ΔP = (p_t − u)/(1 − p_t)`, same `d` formula.
- Every existing tranche: `own_post = own_now × d`. New check in this round: `own = check / POST`, enters at 1.0x.
- Sanity print: new investors = `n` of post, pool = `p_t` of post, all holdings sum to 100%.

Do the arithmetic in a python block (Bash), never freehand — and show the decomposition: total existing dilution = new money points + incremental pool points, plus what the pool refresh alone costs Tom in dollars vs a no-top-up round.

## Step 4: Report — fixed output contract

Two sections, always both, in this order:

**1. Company level — blended and by round.** Table with one row per tranche + a blended row. Column names and order are fixed:

| Tranche | Cost Basis | FMV | Ownership | MOIC |

`Cost Basis` = dollars invested (never "Invested"), `FMV` = value at the modeled POST (never "Value"), `Ownership` = today → post in one cell, `MOIC` = today → pro forma in one cell (same arrow convention as Ownership, no parens). If an additional check is modeled, show it as its own tranche row and note the blended-MOIC drag is mechanical (new dollars at 1.0x).

Tranche labels capitalize every word of the stage/round name: `Pre-Seed SAFE`, `Seed FO`, `Series A FO`, `This Round` — never `Pre-seed SAFE` or `This round`.

**2. Fund level — MOIC and TVPI.** Current and Pro Forma as separate columns (no arrows here), fixed rows:

| Fund Metric | Current | Pro Forma |
|---|---|---|
| Fund FMV | $Xm | $Ym |
| Gross MOIC | X.Xx | Y.Yx |
| Net TVPI (est) | X.Xx | Y.Yx |
| DPI | X.Xx | X.Xx |
| [Company] %FMV | X% | Y% |
| [Company] %Fund Size | X% | Y% |

Methodology behind the rows:
- New fund FMV = old total − company's old FMV + modeled value (+ new check on both sides if applicable).
- Gross MOIC = fund FMV ÷ invested. DPI doesn't move in a markup scenario (all paper) — same value in both columns, no annotation.
- Net TVPI: **Dash funds** — estimate as `(V − 20% × max(0, V − called)) / called`, which reproduces the SOI's current net figure; label as an estimate. **Inverted** — use the lp-portal NAV-bridge methodology and respect the 60%-called TVPI gate; don't invent a net number the portal wouldn't show (if gated, the TVPI row shows `N/A (gated)`).
- Concentration rows: `%FMV` = position FMV ÷ fund FMV; `%Fund Size` = position FMV ÷ fund size — both as percentages (e.g. 64%, not 0.64x).

Close with the read, not just the math (be opinionated): step-up vs last round and elapsed time, whether the implied multiple is heat or trajectory, pro-rata math (what full maintenance would cost vs what the check recovers), remaining-reserves reality check, and secondary/liquidity angle if the mark is rich.

## Known traps

- The 2%→10% pool intuition ("that's 8 points") is off under the standard convention: the old pool dilutes too, so the incremental pool is `p_t − u·d` points of post (8.4, not 8.0, in the 2→10 case) and the pre-money share add is larger still (10.25). Explain the denominators if Tom pushes back.
- Don't restate SOI FMV methodology — SAFEs at cost / LRP marks are the source sheet's job; this skill only layers a hypothetical round on top.
- Sheet read: Drive MCP `read_file_content` on the SOI sheet overflows; it lands in a tool-results file — parse that with python.
