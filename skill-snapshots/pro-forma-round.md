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
| Fund NAV | $Xm | $Ym |
| Gross MOIC | X.Xx | Y.Yx |
| Net TVPI (est) | X.Xx | Y.Yx |
| DPI | X.Xx | X.Xx |
| [Company] % of NAV | X% | Y% |
| [Company] × Fund Size | X.Xx | Y.Yx |

Methodology behind the rows:
- **Fund NAV** = the fund's total portfolio fair value (sum of holdings' FMV). For these Dash funds — ~fully called with reserves deployed — cash is negligible, so portfolio FMV is the NAV; use "NAV" as the label. New NAV = old total − company's old FMV + modeled value (+ new check on both sides if applicable).
- Gross MOIC = fund NAV ÷ invested. DPI doesn't move in a markup scenario (all paper) — same value in both columns, no annotation.
- Net TVPI: **Dash funds** — estimate as `(V − 20% × max(0, V − called)) / called`, which reproduces the SOI's current net figure; label as an estimate. **Inverted** — use the lp-portal NAV-bridge methodology and respect the 60%-called TVPI gate; don't invent a net number the portal wouldn't show (if gated, the TVPI row shows `N/A (gated)`).
- Concentration rows:
  - `% of NAV` = position FMV ÷ fund NAV, as a **percentage** (e.g. 41%) — how much of the fund's value this one name is.
  - `× Fund Size` = position FMV ÷ fund size (committed capital), as a **multiple** (e.g. 0.70x, not 70%) — what this position alone returns against the whole fund, in the same units fund returns are quoted. This is the fund-returner lens: `1.0x` means the position is worth the entire fund; `>1.0x` means it returns the fund on its own.

Close with the read, not just the math (be opinionated): step-up vs last round and elapsed time, whether the implied multiple is heat or trajectory,

> **Step-up is pre-money ÷ prior post-money, not post-to-post.** The valuation step-up = the new round's **pre-money** ÷ the prior round's **post-money** (e.g. Series B pre $301.5m ÷ Series A post $83.5m = 3.6x). Do NOT quote the post-to-post ratio ($335m ÷ $83.5m = 4.0x) as the step-up — that overstates it by counting the new money as growth. (The FMV math still uses the new post-money for `ownership × post`; this note is only about the headline step-up figure in the read.) pro-rata math — state it objectively as three ownership states: no-participation (current % × dilution factor), with the modeled check (+ check ÷ post), and full pro-rata (cost to hold flat, with the check as a % of pro-rata); no editorializing labels like "top-off" or "signal check" in the written record, and remaining-reserves reality check.

**Terminal value analysis (fixed closing block).** After the read, a table of exit scenarios — default ladder $1b / $2b / $5b / $10b unless Tom names other marks:

| Exit Value | Position Value | MOIC | × Fund Size |

Position Value = post-round blended ownership × exit value; MOIC = position ÷ all-in cost; × Fund Size = position ÷ fund committed capital (multiple, one decimal). Header states the assumptions inline: post-round ownership held constant at exit (no further dilution modeled — each future round would dilute further), all-in cost, gross of carry. Do not model secondary sales here — that's separate analysis Tom asks for explicitly.

## Recording the PF to the Opp card body (optional output mode)

Default output is a chat answer that writes nothing. **When Tom asks to record the pro forma into the Opportunity card** (e.g. "run the PF in the body", "log this PF on the card", or a follow-on-card flow that wants the analysis attached), append the analysis to the Opp page body via `notion-update-page` (`insert_content`, `position: end`) using this fixed structure:

1. **Dated header** — `## <Company> <Round> PF Draft — <Mon DD, YYYY>` (today's date, when the analysis was run). "PF Draft" signals it's indicative, not a booked mark.
2. **Assumptions block** — list every input and tag each `✅ confirmed` or `⚠️ NOT confirmed`, with the source. Section and label names use Title Case (Capitalize First Letters): `Round Size`, `Option Pool Refresh`, `Tom's Check`, `Company Level`, `Fund Level`, `Terminal Value`. Always cover, at minimum:
   - **Round Size** (raised $) — confirmed vs. rumored.
   - **Valuation** (post-money / cap) — confirmed vs. rumored.
   - **Option Pool Refresh** — whether a pool top-up is modeled and whether that provision is confirmed. If the term sheet's pool isn't in hand, model none and mark it `⚠️ NOT confirmed` (existing holders diluted by the new-money fraction only), and say what a refresh would cost.
   - **Tom's Check** (if a follow-on check is modeled) and whether it's committed.
   - **Anchor** — the SOI/source snapshot + date the ownership/FMV came from, and the tie-out (blended ownership × last-round post = FMV).
3. **Company level** and **Fund level** tables — same fixed contracts as the Report section above.
4. **Read** — the objective close, as bullets (per the read contract above — three-state pro-rata math, no editorializing labels).
5. **Terminal value table** — the fixed closing block from the Report section (exit ladder with Position Value / MOIC / × Fund Size, assumptions stated inline).
6. **Indicative-scenario footer** — one line stating it's anchored to SOI ownership (not the unsigned cap table) and what to replace when the real pro forma lands.

Confirmed/not-confirmed tagging is the point of the written record: a PF Draft on the card must make explicit which inputs are hard (signed terms) vs. assumed (unconfirmed pool, uncommitted check), so a future reader never mistakes an assumption for a fact.

**Exactly one PF analysis lives on a card at a time — replace, never append.** Re-running (e.g. the real cap table lands, terms change) REPLACES the existing draft in place: locate the prior `## … PF Draft — <date>` block (header through its indicative footer) and swap in the new one, updating the header to the new run date. Never leave two PF drafts on a card — this mirrors how `finalize-diligence` swaps its assessment in place. The refreshed draft reflects whatever inputs have since been confirmed (e.g. once SignalFire's pro forma lands, the option-pool line flips from `⚠️ NOT confirmed` to `✅ confirmed` and the `×0.90` assumption is replaced with the actual dilution).

The canonical example lives on the `Outmarket (Series B FO)` card (Series B PF Draft — Aug 16, 2026).

## Known traps

- The 2%→10% pool intuition ("that's 8 points") is off under the standard convention: the old pool dilutes too, so the incremental pool is `p_t − u·d` points of post (8.4, not 8.0, in the 2→10 case) and the pre-money share add is larger still (10.25). Explain the denominators if Tom pushes back.
- Don't restate SOI FMV methodology — SAFEs at cost / LRP marks are the source sheet's job; this skill only layers a hypothetical round on top.
- Sheet read: Drive MCP `read_file_content` on the SOI sheet overflows; it lands in a tool-results file — parse that with python.
