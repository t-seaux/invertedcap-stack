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

A round is priced only if BOTH layers agree (Layer 1 — Round Details text via `soi_generate.py`; Layer 2 —
deal-doc materials, where a pro-forma cap table resolved via `find_cap_table.py` is the dead giveaway);
otherwise treat as SAFE. **Scope — any Notion write (including auto-correct) is allowed ONLY on in-SOI Opps
(Active Portfolio / Portfolio: Follow-On / Exited). NEVER modify a Committed Opp.** Auto-correct is
one-directional: SAFE docs but Round Details `post` → silently patch `post → cap` (cost-held, label-only, no
confirm); priced docs / a pro-forma cap table but Round Details `cap` → do NOT silently flip (that changes
valuation) → route to B1 and draft the mark for Tom's confirm. Full detail (both layers, doc signals, scope,
auto-correct) in `references/safe-vs-priced-detection.md` — read it now before proceeding.

## Mode B — Tier 2 draft (webhook): priced round or exit

### B1 — Priced follow-on round (re-mark)
A portfolio company raised a **priced** round; Inverted's SAFE(s) convert / the position re-marks off the new
pro-forma cap table. Ownership is NO LONGER invested ÷ cap — it is Inverted's fully-diluted % from the cap
table. Resolve the cap table deterministically via `find_cap_table.py` (**no cap table → send the "cap table
required" alert, carry NO numbers, STOP**), resolve OUR entity row from the Opp's `Fund` via
`find_entity_row.py` (**no row / ambiguous → alert Tom and STOP**), read the pro-forma FD% / total shares /
PPS / post-money, and compute the mark (MOIC always fmv ÷ cost, never hardcoded). **Cross-foot the aggregate
two independent ways — (A) FD% × post-money and (B) Inverted total shares × PPS — they MUST agree within
<0.5%; if not, do NOT propose a number: post both + the inputs and ask Tom to reconcile.** The generator's
**cost tie-out gate** (`shares × conversion_pps` must reproduce each round's cost, 1% tol) is the backstop;
never "fix" a tie-out by editing `Inv @ Round`. Full procedure (cap-table navigation, mark object shape,
`fund_inputs.json` write, the walk-through alert) in `references/mode-b-priced-and-exit.md` — read it now
before proceeding.

### B2 — Exit (distribution)
Status → Exited usually means cash returned to LPs (full or partial). Find the distribution amount, determine
residual NAV still held (0 if fully realized), and post a Slack alert walking through DPI / total value / MOIC
for Tom's in-thread confirm. Full procedure in `references/mode-b-priced-and-exit.md` — read it now before
proceeding.

## Mode C — Apply on confirm (claude-alerts-listener branch)

When Tom replies in the thread, the `claude-alerts-listener` "SOI mark confirm" branch dispatches here with
the parent draft + his reply. Parse his decision — **"confirm"** applies the proposed values verbatim; an
**adjustment** (e.g. "ownership 6.2%", "post-money 40m", "distribution 1.2m", "residual 0") recomputes with
the override (re-derive `fmv = ownership × post-money` if either changes) and applies. Apply with the engine
**always `--dry-run` first** (a GLOBAL flag — before the subcommand), echo the diff, then `bash run.sh`.
**The priced `mark` command cross-foots `sum(shares)×pps` vs `--fmv` and refuses >0.5% off; it takes
`--total-shares` plus the per-round `--shares-json`.** On an ADJUSTED confirm the drafted share counts no
longer hold — apply company-level (drop the per-round flags) and re-mark per-round off the corrected cap table
later. Full command forms + the close-loop reply in `references/mode-c-apply.md` — read it now before
proceeding.

## Listener wiring (one-time)

One-time setup of the notion-webhook SOI handler (fires on Inverted-1 Opp changes with a SOI-relevant
property change; debounces; enqueues Tier-1 rebuild or Tier-2 draft) and the claude-alerts-listener reply
branch that dispatches confirms/adjustments to Mode C. Full procedure in `references/listener-wiring.md` —
read it now before proceeding.

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
