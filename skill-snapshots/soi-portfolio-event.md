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
3. **HOLDINGS** — the affected company's changed lines (`invested → `, `FMV → `, `MOIC → `, `OS% → `), plus
   a per-round block when round composition changes — new round row, markup, status flip. **NEVER as a
   column-aligned or code-block table** — those shatter on Slack mobile (Tom 2026-07-13; see send-alert's
   "NO column-aligned tables"). One line per round, ` · ` separated, label on every value:

   ```
   **<Company> per round:**
   **Pre-Seed (SAFE)** — inv $1,000,000 · OS 5.00% · FMV $1,000,000 · 1.00×
   **Seed (priced)** ← new — inv $500,000 · pending cap-table mark
   **Total** — inv $1,500,000 · FMV pending mark
   ```
4. **PACING** — total investable / first checks / follow-on.
5. **METHODOLOGY** — every draft WALKS THROUGH THE MATH for each changed figure, SAFEs included
   (Tom 2026-07-11). Per affected company: one derivation line per round — SAFE:
   `$inv ÷ $Xm cap = Y% · held at cost → FMV $inv`; priced/converted: `shares × $PPS = $fmv (cost → MOIC×)`
   — then the ownership method (`Σ (invested ÷ cap) across SAFEs` vs `FD% from pro-forma cap table`),
   FMV = Σ rounds, MOIC = FMV ÷ cost. Per changed fund-level line: the formula with the actual numbers
   (e.g. `invested $<total> ÷ $<fund size> = <pct>%`). `soi_notify.py --draft` emits this block
   deterministically; agent-composed drafts follow the same shapes.

Format values exactly as the portal renders them ($X,XXX,XXX; X.X%; X.Xx). The deterministic fallback
(`soi_notify.py --draft`, used by the daily sweep where no agent runs) emits the same sections in plain
old→new lines; agent-composed drafts (this skill) add the modal block and any context (e.g. which Notion
edit caused it).

## STEP 0 — Idempotence preflight (ALL webhook modes, before ANYTHING else)

A status flip is not necessarily an event — an unflip/reflip repair, a duplicate webhook, or a re-pick of
the same dropdown value delivers a flip the SOI has already captured. Whether to engage is decided IN CODE,
never by judgment (Tom 2026-07-13):

```
python3 ~/.claude/skills/soi-portfolio-event/soi_preflight.py
```

| exit | meaning | what you do |
|---|---|---|
| **0** | NO-OP — model matches the published snapshot | **Nothing.** No coinvestor check, no Notion writes, no Slack, no draft. Stop silently. |
| **4** | DATE-REPAIR — values captured, but the flip's automation clobbered Close Date on in-SOI round(s) | Perform ONLY the printed Notion Close Date restores, then stop. No draft, no Slack. |
| **3** | ENGAGE — tracked values changed, OR `TIER2-REQUIRED` lines printed (a priced round awaits its cap-table mark; the generator refuses to model it — see Guardrails) | Apply any printed `DATE-REPAIR first` restores; route each `TIER2-REQUIRED` company to Mode B1; otherwise proceed with Mode A. |
| **1** | GATE-FAIL — generator/validation errors other than PENDING MARK | Alert the per-company errors and stop (same as run.sh). |

The gate is deterministic end to end: it rebuilds the model from live Notion (no `--sync-os`, no writes),
diffs with the same `soi_notify.flatten/diff` the publish pipeline gates on, and sources original dates from
the newest `archive/soi_*.json` — the archive run.sh writes on every published run IS the record of what the
SOI has captured. Date restores are safe by construction: a round is flagged only if its invested AND fmv
match the archive, so a genuine edit (new amount + new date) can never be "repaired" back. Close Date isn't
a webhook-watched property, so the restore fires no jobs.

## Mode A — Tier 1 draft (webhook)

1. **Coinvestors check (every round entering the SOI)** — detection only; Tom writes the relation himself
   (Tom 2026-07-13). The workflow, in order:
   1. **Read THIS Opp's Coinvestors field first.** That relation is what the portal renders for this round.
   2. **Then sweep the round's docs** — the cap table (new-money section) first, else deck / investor
      update / call notes — for investors that SHOULD be on it:
      - **the round's LEAD(s)** — largest new-money check on the cap table or the named lead. A lead is
        ALWAYS expected on the round's own Opp; "already linked on a prior round's Opp" is not a reason
        to skip (miss caught live: Fika led the Signal7 Seed with $3.2m of $4m and was left off the Seed
        FO Opp because it was linked on the Pre-Seed).
      - beyond the lead(s): **only MAJOR institutional funds — never small funds or angels.**
   3. **If the sweep finds firms missing from the field, DO NOT link them yourself.** Put a line in the
      draft alert telling Tom what to add, e.g.:
      `⚠️ Coinvestors field is missing (add manually): **Fika Ventures** (led, $3.2m of $4m)` — then
      continue with the draft; the relation edit isn't webhook-watched, so Tom's fix re-renders on the
      confirm publish (or the next rebuild).

   **If neither the field nor the docs name anyone, leave it alone** — `soi_render_html.py:338`
   auto-renders "N/A" when the list is empty. Don't go hunting / fabricating: N/A is the right outcome
   when none are recorded.
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
- **Layer 2 — deal-doc materials** (corroboration, done here): inspect the Opp's **`Deal Docs`** property —
  and resolve it via `find_cap_table.py` (above), NOT by eyeballing `Diligence Materials`, which routinely
  does not hold the cap table. A cap table resolving at all is itself the strongest priced signal.
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

1. **Resolve the cap table DETERMINISTICALLY — always, first, before any other step.** Never hunt by hand,
   never eyeball the Notion `Diligence Materials` section (the cap table is normally NOT there — it lives in
   the `Deal Docs` property, which points at Drive). Run:

   ```
   python3 ~/.claude/skills/soi-portfolio-event/find_cap_table.py "<Company>" --round "<Round>"
   ```

   It resolves against the Drive mount (`My Drive/Deal Docs/<Company>/<Round> (<Mon YYYY>)/`), which is the
   store of record — Notion chips are just links into it, so the folder is complete where the property is
   lossy. Best match prints first.

   **On nonzero exit (no cap table): send this alert via send-alert, then STOP** — never stop silently
   (Tom 2026-07-13), and never derive a mark from Notion fields, the deck, the round's headline post-money,
   or any assumption. A priced mark comes off the cap table or it does not get made.

   ```
   📄 Cap table required — <Company> (<Round>)
   Status flipped to <Status> with Round Details "<Round Details>" (priced), but no pro-forma cap table
   resolves under Deal Docs/<Company>/. The SOI stays on hold at its last published state — no numbers
   until the cap table lands.
   → Drop it in Deal Docs/<Company>/<Round (Mon YYYY)>/ and the next event or daily sweep picks it up.
   ```
2. Read it and extract, **how to navigate the cap table**:
   - **Resolve OUR entity row DETERMINISTICALLY from the Opp's `Fund` — never hardcode it, never eyeball the
     sheet** (Tom 2026-07-13). The Opp says which fund is investing; that is what tells you which row in the
     Excel is ours. The chain, all from data already on hand:

     ```
     Notion Opp `Fund`  ("Inverted 1️⃣")
       -> fund_inputs.json `notion_fund_prefix`  ("inverted 1")      # same filter soi_generate.py uses
       -> fund_inputs.json `fund_name`           ("Inverted Capital I, LP")  = the legal entity in the cap table
     ```

     ```
     python3 ~/.claude/skills/soi-portfolio-event/find_entity_row.py "<cap table.xlsx>" --sheet "<pro-forma sheet>"
     ```

     It folds punctuation and the Roman/Arabic ordinal (`Inverted Capital I, L.P.` == `Inverted Capital 1, LP`),
     requires **exactly one** row in the pro-forma sheet, and surfaces **near-misses** — an SPV, a placeholder,
     or a sibling fund (`Inverted Capital II`) — which it refuses to substitute. Several rows in the
     convertibles ledger are expected and fine (one per SAFE).

     **On nonzero exit (no row, or ambiguous): alert Tom and STOP.** Reading another entity's row is the
     highest-consequence misread in this flow — it marks our position off someone else's shares, and every
     downstream gate still passes because the arithmetic is internally consistent. The cost tie-out gate is
     the backstop (a foreign row won't reproduce our cost basis), but do not lean on it: resolve the row right.
   - **Aggregate (headline):** read Inverted's **`Fully Diluted Ownership %`** and **total share count** in the
     **PRO-FORMA section** (the post-round "cap table construction" columns that reflect the round once
     closed — NOT the pre-round / current columns). Read the round's **price-per-share (PPS)** and
     **post-money valuation**.
   - **Per round (for per-round MOIC):** get **how many shares each of Inverted's rounds holds post-round**.
     - **SAFEs convert into the new round** — for a SAFE round, find the shares it **converted into**, not a
       notional count. Find the **convertibles/SAFE conversion ledger** (the tab where SAFEs convert) and
       read Inverted's converted-share count. The fair value of that converted position = **converted shares
       × the NEW round Price-Per-Share** (NOT the conversion price) — the gap between the conversion price
       and the new PPS is the SAFE's markup, so per-round MOIC = `new PPS ÷ conversion price`.
     - **Do not call the markup a "discount".** The **conversion price is `min(valuation-cap price,
       discount price)`** and on a good round the **cap almost always binds** — the discount price (a % off
       the new PPS) is the *higher* of the two, so it buys *fewer* shares and never engages. Read the
       ledger's `Conversion Price` column; don't infer the mechanism from the presence of a discount term.
       The SAFE marks up simply because its conversion PPS is below the new round's PPS.
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
     `{ "pps": <NEW round PPS>, "post_money": <$>, "ownership": <FD% decimal>, "total_shares": <n>,
        "shares_by_round": { "<round_label>": { "shares": <converted shares>,
                                                "conversion_pps": <the price THIS round converted AT> }, ... } }`
     (round labels must match the SOI's, e.g. `Pre-Seed`, `Pre-Seed+`. A bare `<shares>` int is still accepted
     for legacy marks, but always write the object form.)

     **Always record BOTH prices per round — the old (conversion) PPS and the new round PPS** (Tom 2026-07-13).
     They are what make every multiple reconstructible and auditable later:
     `round multiple = pps ÷ conversion_pps`. `conversion_pps` is the ledger's **`Conversion Price`** column —
     `min(valuation-cap price, discount price)` — NOT the new PPS and NOT the discount price. For a
     **new-money** round, `conversion_pps` is simply the round's own PPS (so it marks at 1.00x, at cost).

     **Cost basis is INVARIANT and is never written here.** It lives in Notion (`Inv @ Round`, one Opp per
     round) and the generator reads it live. A round's cost is what we wired; it does **not** change when a
     later round re-marks the position. That invariance is load-bearing: the generator's **cost tie-out gate**
     uses it to validate the cap-table read — `shares × conversion_pps` must reproduce the round's cost
     (1% tol). If it fails, you misread the share column (pre- vs post-round) or the wrong entity row. Never
     "fix" a tie-out failure by editing `Inv @ Round` to match the cap table — the wire is the truth; the
     read is what's wrong.

     The generator then marks each round at shares × NEW pps, derives per-round + headline MOIC
     (always fmv ÷ cost, never hardcoded), and re-runs the cross-foot + cost tie-out gates.
4. Post a Slack alert (send-alert, GFM links) to #claude-alerts that **walks through the inputs and both
   cross-foot methods**:

   Structure with **bold section headers and bold headline values** (Tom 2026-07-13) — bold carries the
   eye down the message on mobile; italics don't. One line per round, ` · ` separated (see send-alert's
   "NO column-aligned tables"). Full math lives under METHODOLOGY at the bottom, not inline:

   ```
   📈 **<Company> — <Round> mark** (draft · reply **confirm** or adjust)

   **ROUND**
   $<raise> at $<post> post · PPS $<pps> · <cap table filename>

   **OUR POSITION** — <entity> · <total shares> FD sh = **<A>%**
   **<Round 1> (SAFE)** — $<cost> → $<fmv> · <X>× (conv $<conv_pps>, cap $<cap> binds)
   **<Round 2>** ← new — $<cost> → $<fmv> · 1.00×
   **Total** — $<cost total> → **$<fmv total> · <D>×**

   **CHECKS**
   FD% × post $<C1> ≈ sh × PPS $<C2> ✓ (<delta>%) · cost tie-outs ✓

   **FUND**
   Invested $<a>m → $<b>m · FMV $<a>m → $<c>m · MOIC <x>× → **<y>×**

   **METHODOLOGY**
   <one derivation line per round: shares × new PPS = FMV; tie-out shares × conv PPS = cost ✓>

   Reply **confirm** to apply, or adjust: e.g. "ownership 6.2%" / "post-money 40m".
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
# priced mark — per-round form (the engine cross-foots sum(shares)×pps vs --fmv, refuses >0.5% off)
python3 ~/code/lp-portal/refresh_inputs.py [--dry-run] mark --company <C> \
  --ownership <dec> --fmv <int> --pps <float> --post-money <int> --total-shares <int> \
  --shares-json '{"<round label>": {"shares": <n>, "conversion_pps": <float>}, ...}'
# exit / distribution
python3 ~/code/lp-portal/refresh_inputs.py [--dry-run] distribution --company <C> --amount <int> --residual-fmv <int>
cd ~/code/lp-portal && bash run.sh
```
(`--dry-run` is a GLOBAL flag — before the subcommand. On an ADJUSTED confirm the drafted share counts no
longer hold: apply company-level — drop the per-round flags — and re-mark per-round off the corrected cap
table later.)

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

- **No cap table, no priced math — PERIOD** (Tom 2026-07-13). Priced-round numbers (ownership, FMV, MOIC,
  converted shares) may ONLY ever be read off an actual cap-table document resolved via `find_cap_table.py`.
  Never derive them from Round Details, a deck, a press post-money, or arithmetic on prior rounds — naive
  `invested ÷ post` ignores dilution and SAFE conversion and WILL be wrong. This bans estimates everywhere,
  including draft/alert text: if the cap table isn't in hand, the alert says "cap table needed" and carries
  NO numbers. The ONLY document-free math permitted is SAFE math, which is simple by construction:
  `ownership = invested ÷ cap`, held at cost (`FMV = invested`, MOIC 1.0×).
- **Both tiers are draft-for-confirm — nothing publishes without Tom's in-thread reply** (Tier 1 gate
  added 2026-07-11). Tier 2 additionally walks through the cap-table inputs and both cross-foot methods
  every time so Tom can sanity-check the read.
- Every draft follows the before→after conveying convention (portal sections, `old → new` arrows).
- Never write portfolio facts to Notion; only `fund_inputs.json` (priced_round_marks / distributions).
- **`Inv @ Round` is FROZEN once a round is in the SOI** (SHARED_SAFETY #7, Tom 2026-07-13). Never write
  it on an in-SOI Opp — not to fix a tie-out, not to reconcile a cap table. The preflight prints a
  `COST-EDIT` line whenever an in-SOI round's cost differs from the last published archive; every draft
  alert must surface that line FIRST so an unintended overwrite can't hide in a routine diff.
- A bad cap-table read is the main risk — show the share counts and post-money you used, not just the %.
- TVPI/RVPI stay gated to N/A until 60% called; a mark doesn't change the gate (but updates the underlying).
