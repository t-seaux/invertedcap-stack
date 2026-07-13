---
name: soi-portfolio-event
description: >-
  Handle a portfolio change on an Inverted Capital I Opportunity that affects the SOI, driven by the
  notion-webhook. Two tiers, BOTH draft-for-confirm. TIER 1 (field-driven — new investment, SAFE follow-on,
  status change, Inv @ Round / Round Details edit): build the draft model, post a before→after portal diff
  (Fund Returns / Summary Metrics / Holdings / Pacing) to Slack, publish via run.sh only on Tom's in-thread
  confirm. TIER 2 (document-driven
  — a PRICED follow-on round, or an EXIT with cash back): read the deal-doc cap table / wire, draft the
  proposed mark or distribution, and post a Slack alert to #claude-alerts that WALKS THROUGH THE MATH so Tom
  can reply in-thread to confirm or adjust; a claude-alerts-listener branch applies the confirmed value to
  fund_inputs.json and rebuilds. Marks/distributions live in fund_inputs.json (not Notion). Trigger: invoked
  by claude-job-queue from notion-webhook (Modes A/B); manual "draft the mark for <company>" also works.
---

# soi-portfolio-event

Real-time SOI maintenance when a portfolio Opp changes. Companion to `soi-refresh-inputs` (which handles
*fund-admin* re-anchoring on a quarterly cadence); this one handles *portfolio* changes event-driven.

Engine: `~/code/lp-portal/refresh_inputs.py` (subcommands `mark`, `distribution`) + `run.sh`.
**Marks/distributions live in `fund_inputs.json`** — `priced_round_marks[company] = {ownership, fmv}` and
`distributions[company]`. Never write investment/portfolio facts to Notion; those are read live by `soi_generate.py`.

## The two tiers

| Tier | Trigger (notion-webhook) | Handling |
|---|---|---|
| **1 — field-driven** | new Opp → Active Portfolio; SAFE follow-on → Portfolio: Follow-On; any Status change; Round Details / Inv @ Round edit (`opp-soi-field-edit`) | **Draft + confirm** (Tom 2026-07-11 — gate added; was rebuild-only). Build the draft model, show the before→after per the conveying convention, publish only on Tom's in-thread confirm. |
| **2 — document-driven** | a follow-on round that is **priced** (cap table, not a SAFE); or Status → **Exited** with cash returned | **Draft + confirm** — read the doc, walk through the math in Slack, apply on Tom's in-thread confirm. |

BOTH tiers gate on Tom's confirm before the live doc moves. The difference: Tier 1 drafts a Notion-derived
recompute (no judgment, no fund_inputs writes); Tier 2 drafts a valuation (cap-table read → mark). If
`Round Details`/`Inv @ Round` are blank the generator gate fails and the alert says so.

## Conveying changes — the before→after convention (Tom 2026-07-11)

Every draft (Tier 1 and Tier 2) shows the change as the ACTUAL portal elements with `old → new` arrows —
never a bare list of numbers. Sections, in portal order, showing ONLY lines that change:

1. **FUND RETURNS** — MOIC (Gross) / TVPI (Net) / DPI.
2. **SUMMARY METRICS** — companies, invested, fair value, **and the averages** (avg check, avg post, avg
   ownership, first-check %) when they move.
3. **HOLDINGS** — the affected company's table row (`invested → `, `FMV → `, `MOIC → `, `OS% → `), plus a
   company-modal block (`~/holdings/<co>` style: INVESTMENT SUMMARY lines + per-ROUND rows) when round
   composition changes — new round row, markup, status flip.
4. **PACING** — total investable / first checks / follow-on.
5. **METHODOLOGY** — every draft WALKS THROUGH THE MATH for each changed figure, SAFEs included
   (Tom 2026-07-11). Per affected company: one derivation line per round — SAFE:
   `$inv ÷ $Xm cap = Y% · held at cost → FMV $inv`; priced/converted: `shares × $PPS = $fmv (cost → MOIC×)`
   — then the ownership method (`Σ (invested ÷ cap) across SAFEs` vs `FD% from pro-forma cap table`),
   FMV = Σ rounds, MOIC = FMV ÷ cost. Per changed fund-level line: the formula with the actual numbers
   (e.g. `invested $6,596,697 ÷ $25m fund = 26.4%`). `soi_notify.py --draft` emits this block
   deterministically; agent-composed drafts follow the same shapes.

Format values exactly as the portal renders them ($X,XXX,XXX; X.X%; X.Xx). The deterministic fallback
(`soi_notify.py --draft`, used by the daily sweep where no agent runs) emits the same sections in plain
old→new lines; agent-composed drafts (this skill) add the modal block and any context (e.g. which Notion
edit caused it).

## Mode A — Tier 1 draft (webhook)

1. **Coinvestors check (new portfolio entries only)** — if this is a **new** SOI entry (Status flipped to
   Active Portfolio for the first time, or a Follow-On Opp with no prior round in the SOI), inspect the
   Opp's **Coinvestors** relation. If empty, look in the Opp body / diligence materials (deck / cap table /
   investor update / call notes) for explicitly-named coinvestors and link the matching Companies DB rows
   (create new ones for any coinvestor not yet tracked). **Selectivity bar: only MAJOR institutional funds
   — never small funds or angels** (e.g. Signal7 Seed cap table lists Fika, Recall, TMV, an angel → link
   Fika only; when unsure, leave it off and ask). **If no coinvestors are listed anywhere on the Opp, leave
   the relation empty** — `soi_render_html.py:338` auto-renders "N/A" when the list is empty. Don't go
   hunting / fabricating: N/A is the right outcome when none are recorded.
2. **Build the draft model without publishing:**
   `cd ~/code/lp-portal && python3 soi_generate.py --strict --out /tmp/soi_model_draft.json`
   (no `--sync-os` — no Notion writes before Tom confirms). Generator gates must pass; on gate failure
   alert the per-company errors and stop, exactly like run.sh does.
3. **Compose the draft** per the conveying convention above: diff the draft model against
   `.last_snapshot.json` (labels/kinds mirror `soi_notify.flatten`), render the changed sections with
   `old → new` arrows, and include the company-modal block for the affected Opp. Header line MUST start
   with `🧾 **SOI rebuild pending your confirm**` — claude-alerts-listener routes replies on it. End with:
   `Reply **confirm** to publish, or fix Notion and this re-drafts.`
4. **Post via send-alert and STOP.** Nothing is published, no snapshot is written. Tom's in-thread
   "confirm" → claude-alerts-listener "SOI rebuild confirm" branch runs `bash run.sh` (plain), which
   publishes + deploys + updates the snapshot; its `📊 SOI updated` alert is the applied record.
   Note: confirm publishes the LATEST Notion state (run.sh regenerates), not the drafted snapshot — if
   Notion moved again in between, the post-publish diff shows the final values.

## Detecting SAFE vs priced (two layers)

A round is a priced round only if BOTH layers agree; otherwise treat as SAFE.

- **Layer 1 — Round Details text** (deterministic, done by `soi_generate.py`): `cap` ⇒ SAFE,
  `post` ⇒ priced, an explicit `SAFE` anywhere ⇒ SAFE, `priced`/`equity round` ⇒ priced.
- **Layer 2 — deal-doc materials** (corroboration, done here): inspect the Opp's Diligence Materials.
  - **SAFE** signals: a SAFE agreement / post-money SAFE doc (often + a side letter only).
  - **Priced** signals: **a pro-forma cap table** (dead giveaway), Stock Purchase Agreement (SPA),
    Investor Rights Agreement (IRA), Right of First Refusal & Co-Sale (ROFR), Voting Agreement.

**Scope — only ever edit in-SOI Opps.** Any Notion write (incl. the auto-correct below) is allowed ONLY on
Opps whose Status is **Active Portfolio, Portfolio: Follow-On, or Exited**. NEVER modify a **Committed** Opp
(or anything else) — Committed is out of SOI scope and may legitimately be a future priced round written as
`post` (e.g. Signal7's Committed Seed FO at `$42m post`). Leave it alone.

**Auto-correct rule (no confirm needed):** if an **in-scope** Opp shows **SAFE docs** in Layer 2 but Round
Details says `post`, silently fix the label — patch Round Details `post → cap` in Notion (a SAFE is held at
cost, so this is a label fix, not a valuation change). Conversely, if Layer 2 shows **priced docs / a pro-forma cap table**
but Round Details says `cap`, do NOT silently flip it — that changes valuation, so go to B1 and
**draft the mark for Tom's confirm**, noting the cap-vs-post conflict in the alert.

## Mode B — Tier 2 draft (webhook): priced round or exit

### B1 — Priced follow-on round (re-mark)
A portfolio company raised a **priced** round; Inverted's SAFE(s) convert / the position re-marks off the
new pro-forma cap table. Ownership is NO LONGER invested ÷ cap — it is Inverted's fully-diluted % from the
cap table.

1. Find the deal-doc cap table for the round (Opp's Diligence Materials / attached deck / data room). If
   not present, alert Tom that the cap table is needed and stop.
2. Read it and extract, **how to navigate the cap table**:
   - **Find Inverted's entity row(s)** — the fund holds as **`Inverted Capital 1, LP`** (cap tables may also
     write the official Roman form `Inverted Capital I, LP`; match either). Do NOT use a placeholder/SPV row.
   - **Aggregate (headline):** read Inverted's **`Fully Diluted Ownership %`** and **total share count** in the
     **PRO-FORMA section** (the post-round "cap table construction" columns that reflect the round once
     closed — NOT the pre-round / current columns). Read the round's **price-per-share (PPS)** and
     **post-money valuation**.
   - **Per round (for per-round MOIC):** get **how many shares each of Inverted's rounds holds post-round**.
     - **SAFEs convert into the new round** — for a SAFE round, find the shares it **converted into**, not a
       notional count. Find the **convertibles/SAFE conversion ledger** (the tab where SAFEs convert) and
       read Inverted's converted-share count. The fair value of that converted position = **converted shares
       × the NEW round Price-Per-Share** (NOT the discounted conversion price) — the gap between conversion
       price and new PPS is the SAFE's markup.
     - A direct new-money purchase = the shares bought in that round.
     - Aggregate check: the entity's `Fully Diluted Shares` in the post-round cap table should equal the sum
       of its rounds' (converted) shares.
   - **Navigate by MEANING — cap-table formats vary widely** (sheet & column names differ by vendor/law
     firm). Synonyms you'll see:
     - SAFE conversion tab: `SAFE Conversion`, `Convertibles Ledger`, `Convertible Ledger`, `SAFE Schedule`.
     - Converted-shares column: `Conversion Shares`, `Number of Conversion Shares`, `Shares (As Converted)`.
     - Ownership: `Fully Diluted Ownership`, `FD %`, `As-Converted %`; shares: `Fully Diluted Shares`.
     - The post-round view: `Pro Forma`, `Summary/Detailed Pro Forma`, or a pro-forma column block in the
       cap table. If unsure which column is post-round, show Tom the candidates in the draft.
   - Two real models to calibrate against (different layouts, same mechanics):
     `~/Downloads/Outmarket Series Seed Pro Forma (Final).xlsx` and `~/Downloads/Suppli - Series A - Pro
     Forma. (4923-2426-8178.4).xlsx`.
3. Compute the proposed mark. **MOIC is always fair value ÷ cost basis — never hardcode it.**
   - **Per round:** `round_fmv` = round's (converted) shares × PPS; `round_moic` = round_fmv ÷ that round's
     invested cost.
   - **Headline (company):** cost basis and fair value are the **sum of the round rows** —
     `fmv` = Σ round_fmv = total shares × PPS; `cost` = Σ round invested; `MOIC` = fmv ÷ cost.
   - **Cross-foot the aggregate two independent ways — they MUST match** (algebraically identical, so a
     mismatch means a bad read — wrong shares, PPS, %, or post-money):
     - **(A)** FD ownership % × post-money
     - **(B)** Inverted total shares × PPS  (= Σ round_fmv)
     - Agree within <0.5% → proceed. If not, do NOT propose a number — post both + the inputs and ask Tom
       to reconcile.
   - Write the mark to `fund_inputs.json` `priced_round_marks[Company]` as:
     `{ "pps": <PPS>, "post_money": <$>, "ownership": <FD% decimal>, "total_shares": <n>,
        "shares_by_round": { "<round_label>": <converted shares>, ... } }`
     (round labels must match the SOI's, e.g. `Pre-Seed`, `Pre-Seed+`). The generator then marks each round
     at shares×PPS, derives per-round + headline MOIC, and re-runs the cross-foot gate.
4. Post a Slack alert (send-alert, GFM links) to #claude-alerts that **walks through the inputs and both
   cross-foot methods**:

   ```
   📈 Priced round — <Company> needs a mark (reply in thread to confirm/adjust)
   Cap table (<source>): Inverted Capital 1, LP holds <shares> FD shares of <total> = <A>% post-round
   Round PPS: $<pps>   Post-money: $<B>
   Fair value (A) = <A>% × $<B> = $<C>
   Fair value (B) = <shares> × $<pps> = $<C>   ✓ cross-foot match
   Proposed mark → ownership <A>%, fair value $<C>; MOIC = $<C> ÷ $<cost> = <D>×
   Reply "confirm" to apply, or e.g. "ownership 6.2%" / "post-money 40m" to adjust.
   ```

   Do NOT write anything yet. The parent message must carry a recognizable origin so the listener routes
   the reply here (see "Listener wiring").

### B2 — Exit (distribution)
Status → Exited usually means cash returned to LPs (full or partial).

1. Find the distribution amount (closing statement / wire confirmation email / Tom's note). If unknown,
   alert and ask.
2. Determine residual NAV still held after the distribution (0 if fully realized).
3. Post a Slack alert walking through it:

   ```
   💸 Exit — <Company> distribution (reply in thread to confirm/adjust)
   Cash distributed to LPs: $<amount>  (source: <wire/closing stmt>)
   Residual NAV still held: $<residual>
   Effect: DPI += <amount>/paid-in; total value = residual + distribution; MOIC = (residual+dist)/cost = <D>×
   Reply "confirm" to apply, or give the correct amount / residual.
   ```

## Mode C — Apply on confirm (claude-alerts-listener branch)

When Tom replies in the thread, the `claude-alerts-listener` "SOI mark confirm" branch dispatches here with
the parent draft + his reply. Parse his decision:

- **"confirm"** → apply the proposed values verbatim.
- **adjustment** (e.g. "ownership 6.2%", "post-money 40m", "distribution 1.2m", "residual 0") → recompute
  with the override (re-derive `fmv = ownership × post-money` if either changes) and apply.

Apply with the engine (always `--dry-run` first, echo the diff), then rebuild:

```
# priced mark
python3 ~/code/lp-portal/refresh_inputs.py mark --company <C> --ownership <dec> --fmv <int>
# exit / distribution
python3 ~/code/lp-portal/refresh_inputs.py distribution --company <C> --amount <int> --residual-fmv <int>
cd ~/code/lp-portal && bash run.sh
```

Then post a close-loop reply IN THE SAME THREAD: the applied values, the new company MOIC, and (if shown)
the new fund DPI / TVPI. Deliver the rebuilt `~/Inverted_Capital_I_SOI.html`.

## Listener wiring (one-time)

- **notion-webhook** (`~/code/notion-webhook/Code.js`): an SOI handler fires on Opportunity changes where
  Fund = Inverted 1 AND Status ∈ {Active Portfolio, Portfolio: Follow-On, Committed, Exited} AND a
  SOI-relevant property changed (Status, Round Details, Inv @ Round, 🕰️ Funding History). It **debounces**
  (coalesce rapid edits) and enqueues a claude-job-queue job: Tier-1 rebuild by default; Tier-2 draft when
  the round reads priced/equity or Status → Exited.
- **claude-alerts-listener**: add a branch that recognizes this skill's draft parent messages (priced-round
  / exit drafts) and dispatches the reply to `soi-portfolio-event` Mode C.

## Guardrails

- **Both tiers are draft-for-confirm — nothing publishes without Tom's in-thread reply** (Tier 1 gate
  added 2026-07-11). Tier 2 additionally walks through the cap-table inputs and both cross-foot methods
  every time so Tom can sanity-check the read.
- Every draft follows the before→after conveying convention (portal sections, `old → new` arrows).
- Never write portfolio facts to Notion; only `fund_inputs.json` (priced_round_marks / distributions).
- A bad cap-table read is the main risk — show the share counts and post-money you used, not just the %.
- TVPI/RVPI stay gated to N/A until 60% called; a mark doesn't change the gate (but updates the underlying).
