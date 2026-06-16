---
name: soi-refresh-inputs
description: >-
  Re-anchor the Fund 1 SOI's fund-level inputs when fresh administrator docs arrive from Vector / Valence.
  The SOI's returns run on a NAV roll-forward (live NAV = last audited Total Partners' Capital, rolled
  forward daily by documented events + LPA management-fee accrual). This skill updates the ANCHOR and fee
  facts in ~/.claude/scripts/soi/fund_inputs.json — it does NOT touch investment/portfolio data (that is
  live from Notion). Three fund-admin sources: (1) management fee = latest Vector email .xlsx; (2) financial
  statements (NAV / paid-in) = Valence portal; (3) capital calls = Valence portal. Docs lag quarter-end by
  weeks; between anchors the daily run.sh keeps rolling forward, so this is NOT time-gated — it re-anchors
  only when a doc newer than the current anchor appears. Trigger: "refresh SOI inputs", "re-anchor SOI",
  "new financials from Vector", "Vector sent the fee calc", "update SOI cap calls", or a scheduled poll.
---

# soi-refresh-inputs

Keeps the SOI's **fund-level** inputs current. The fund-returns model (`soi_generate.py`) uses a **NAV
roll-forward**: the live NAV is the last *audited* Total Partners' Capital, rolled forward each day by
documented movements (new capital calls, priced-round markups, distributions) plus the LPA management-fee
accrual, and it re-anchors to each new financial statement. The daily `run.sh` (launchd) does the rolling
forward automatically — **this skill only updates the anchor + fee facts when fresh source docs arrive.**

## Source-of-truth boundary (do not cross)

| Data | Source | Touched by this skill? |
|---|---|---|
| Companies, invested cost, ownership, round details, marks | **Notion** (live via `soi_generate.py`) | **NO — never** |
| Fund NAV anchor (Total Partners' Capital), paid-in (Contributed Capital) | **Valence** financial statements | yes — `financials` |
| Management fee base / rate / ITD | **Vector email** `.xlsx` | yes — `fee` |
| Capital call schedule (status / dates / %) | **Valence** portal | yes — `calls` |

`fund_inputs.json` blocks this skill may edit: `financial_statements`, `management_fee`,
`capital_call_schedule`. Leave everything else alone.

## Timing model

Administrator docs arrive **weeks after quarter-end** (e.g. Q1 ending 3/31 may not be final until late in
the next quarter). That is expected and fine: the roll-forward covers the gap. So:

- **Never** assume a new statement exists just because a quarter ended.
- Re-anchor **only** when a doc's period-end is **newer than the current anchor** (`refresh_inputs.py show`
  prints the current anchor `as_of`). The engine refuses stale re-anchors unless `--force`.
- Until then, do nothing — the daily roll-forward is already correct.

## Engine

`~/.claude/scripts/soi/refresh_inputs.py` — deterministic; every mutation prints a field diff and writes
atomically. Always `--dry-run` first, show the diff, then apply.

```
refresh_inputs.py show                         # current anchor + fee facts + live roll-forward today
refresh_inputs.py fee --xlsx PATH              # parse Vector fee workbook -> management_fee
refresh_inputs.py financials --nav N --paid-in N --as-of YYYY-MM-DD [--distributions N] [--force]
refresh_inputs.py calls --json '[{...}]'       # replace capital_call_schedule
```

## Modes

### Mode A — Manual ("refresh SOI inputs", "Vector sent the fee calc", etc.)

Run whichever sources Tom points at. Default: check all three.

### Mode B — Scheduled poll

Run on a cadence (e.g. weekly). For each source, check whether something newer than the current anchor
exists; re-anchor if so, otherwise report "still rolling forward from <anchor date>".

## Steps

Run `refresh_inputs.py show` first to see the current anchor `as_of` and fee facts.

### 1. Management fee (Vector email `.xlsx`)

1. Gmail search: `from:vectorais.com subject:"management fee" has:attachment` — take the most recent thread
   (sender is typically `mpyzdrowski@vectorais.com`; subject like `Q3 '26 Management Fee`).
2. Download the `.xlsx` attachment to `/tmp/` (Gmail MCP `get_thread` FULL_CONTENT → attachment).
3. `refresh_inputs.py --dry-run fee --xlsx /tmp/<file>.xlsx` → show the diff → apply if it changed.
   - The workbook's `Calc` sheet carries: `Management Fee Base` (LP commitments only — GP + side-letter
     LPs excluded, per LPA 6.1(b)/(f)), `Initial Closing` (fee start), `Total fees (inception-to-date)`,
     and a step-down table (2.5%→1.5% after 20 quarters). The parser reads these by label.

### 2. Financial statements (Valence → NAV anchor)

1. Read the latest financial statements from the **Valence portal**
   (`valence.vectorais.com`, authenticated) via the Claude-in-Chrome MCP — locate the most recent
   statement package and open the **Statement of Assets, Liabilities and Partners' Capital**. (If Tom
   instead hands you the PDF, `Read` it directly.)
2. Extract three figures from that statement:
   - **NAV** = `Total partners' capital`
   - **Paid-in** = `Contributed capital`
   - **Distributions** = cumulative distributions to date (0 if none)
   - and the statement **period-end date**.
3. If period-end is newer than the current anchor:
   `refresh_inputs.py --dry-run financials --nav <N> --paid-in <N> --as-of <YYYY-MM-DD> --distributions <N>`
   → show diff → apply. Otherwise report it's not newer and stop.

   ⚠️ These are dollars-and-cents authoritative numbers. Read them off the statement; never estimate or
   round. NAV (`Total partners' capital`) already nets all fees, expenses, and syndication costs — do not
   adjust it.

### 3. Capital calls (Valence)

1. Read `valence.vectorais.com/investor-contact-portal/capital-calls` via Chrome.
2. Build the schedule rows (`n`, `month` e.g. `"Sep 2026"`, `status` `"Completed"`/`""`, `pct` as a
   decimal). Mark a tranche `Completed` only when Valence shows it called/funded.
3. `refresh_inputs.py --dry-run calls --json '[...]'` → show diff → apply.

### 4. Rebuild + verify + alert

1. `cd ~/.claude/scripts/soi && bash run.sh` (omit `--no-send` so change alerts fire) — gates must pass.
2. Show Tom: the field diffs applied, the new anchor, and the resulting live NAV / would-be TVPI from
   `refresh_inputs.py show`.
3. Deliver the rebuilt `~/Inverted_Capital_I_SOI.html` (SendUserFile).

## Guardrails

- **Confirm before applying** financials/calls changes — these drive LP-facing returns. Show the diff,
  then write. (Fee parse is low-risk; still show the diff.)
- Never write investment/portfolio fields — those come from Notion.
- If a source is unreachable (Valence login, missing attachment), skip that source, report it, and leave
  the existing anchor in place. The roll-forward stays valid.
- TVPI/RVPI remain gated to N/A until 60% called (Tom's setting in `fund_inputs.json`); re-anchoring does
  not change the gate.
