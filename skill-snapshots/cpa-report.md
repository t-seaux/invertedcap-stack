---
name: cpa-report
description: Generate the quarterly CPA report for Dash Fund entities (Dash Venture Fund and Dash Fund Management) from per-account Citi CSV exports. Produces a formula-linked Excel workbook (Raw → Detail → Summary, with SUMIFS aggregations) — Summary tab shows income, expenses, distributions, and (when LP CSVs are supplied) liquidation proceeds, by period and entity, with a YTD column under each entity group. A 5-check deterministic verification pass runs before write; any failed check aborts. Output lands at the canonical `~/.../Dash Tax & Audit/` iCloud location and prior versions auto-archive. Trigger when Tom says "run CPA report", "generate CPA report", "quarterly CPA report", "CPA Excel", "build the CPA report", "run the Dash CPA report", or any variant asking to produce the quarterly fund report for his CPA. Also trigger when Tom uploads Citi per-account CSVs and asks to process them into a report. Always trigger inline — no confirmation needed before acting.
---

# CPA Report Skill

Generates the quarterly Dash Fund CPA report from per-account Citi CSV exports. The output is a single Excel workbook with **live formula links** between Raw → Detail → Summary, delivered to the canonical location at `~/Library/Mobile Documents/com~apple~CloudDocs/Desktop/Dash Tax & Audit/`. Prior versions are auto-archived to the `Archive/` subfolder on every run. A deterministic verification pass runs before the workbook is written; any failed check aborts the write.

---

## Inputs

1. **Per-account Citi CSVs** (canonical as of 2026-05-27) — one CSV per Citi account, each named `CCB_CHECKING_<account>_<DDMMYYYY>.csv` with columns `DATE, TRANSACTION TYPE, DESCRIPTION, AMOUNT (USD), BALANCE (USD)`. `AMOUNT > 0` = inbound, `AMOUNT < 0` = outbound. The account number is derived from the filename. Tom drops these into iCloud `~/Library/Mobile Documents/com~apple~CloudDocs/Downloads/` (mirrored at `/Users/tomseo/Downloads/`).
   - For a full report: 5 CSVs (DVF MC, DFM MC, Fund I LP, Fund II LP, Fund II-A LP).
   - For income/expense/distributions only (no liquidation events): the 2 MC CSVs suffice — pass `--skip-liquidation`.
2. **Liquidation event mappings** (optional) — Tom may specify which inbound LP wires map to which portfolio company (e.g. "Acquiom wire on 1/9 = Teal Technologies"). If not provided, use the keyword mapping table below and flag unknowns.
3. **New category mappings** (optional) — Tom may specify how to classify previously `Uncategorized` transactions. Add these to the mapping table for the current run.

### Builder script

`build_report.py` in this skill folder handles ingest + Excel generation. Invocation:

```bash
python3 build_report.py \
  /path/to/CCB_CHECKING_6880014247_*.csv \
  /path/to/CCB_CHECKING_6880010107_*.csv \
  [/path/to/CCB_CHECKING_6880018539_*.csv ...] \
  --year 2026 --through 2026-05-31 \
  [--skip-liquidation]
  # --out is optional; default → ~/.../Dash Tax & Audit/Dash CPA Report - <range> <year>.xlsx
```

The script:
1. Parses each per-account CSV and classifies every in-scope txn.
2. Runs the verification pass (5 deterministic checks against in-memory data). If any check fails, the script prints diagnostics and exits non-zero **without** writing the workbook.
3. Builds the workbook with formula-linked Raw → Detail → Summary tabs.
4. Archives any existing `Dash CPA Report - *.xlsx` in the canonical folder to `Archive/`.
5. Writes the new workbook to the canonical folder.

---

## Entity and Account Mapping

Two entities are tracked. Each maps to a Citi management company (MC) account:

| Entity | Account Number | Description |
|---|---|---|
| Dash Venture Fund (DVF) | 6880014247 | Fund I MC |
| Dash Fund Management (DFM) | 6880010107 | Fund II MC |

Supporting LP accounts (used for liquidation proceeds detection only):

| Account | Label |
|---|---|
| 6880018539 | Fund I LP |
| 6880016664 | Fund II LP |
| 6880018299 | Fund II-A LP |

---

## Period Definitions

Transactions are bucketed into four periods matching the estimated tax payment calendar. Use `1/1–3/31` style labels — never quarter designations (Q1, Q2, etc.).

| Period Label | Date Range |
|---|---|
| 1/1–3/31 | Jan 1 – Mar 31 |
| 4/1–5/31 | Apr 1 – May 31 |
| 6/1–8/31 | Jun 1 – Aug 31 |
| 9/1–12/31 | Sep 1 – Dec 31 |

Only render columns for periods that have actual transaction data. If the CSV covers only Jan–Mar, the workbook will have one data column per entity group.

---

## Transaction Classification

### MC Account Logic (per-account CSV)

The CSV's account identity comes from the filename. Each row's sign tells you direction:

- **Inbound** (`AMOUNT > 0`):
  - `DESCRIPTION` contains an **LP account number** (`6880018539`, `6880016664`, `6880018299`) → **Income / Management Fees**
  - `DESCRIPTION` contains the **other MC account number** (or `Transfer to/from Ready Credit`) → **Internal Transfer** (excluded from Summary; still rendered in detail)
  - `DESCRIPTION` is `RETURNED CHECK` → pair with the prior same-account outgoing of equal absolute amount; **both legs** are reclassified as **Internal Transfer / Wash – Returned Check** and excluded from Summary (per Tom 2026-05-27: "leave blank cause they net to 0")
  - Else → Uncategorized
- **Outbound** (`AMOUNT < 0`): apply the keyword mapping table below to `DESCRIPTION`.

Citi prefixes account numbers with `00` in descriptions (e.g. `006880018539 VIA CBusOL Re # 017477`). Match by substring, not `startswith`, so the prefix doesn't matter.

### Category Keyword Mapping (matched against DESCRIPTION, case-insensitive)

| Pattern | Section | Category |
|---|---|---|
| `Ryan Sells` | Distributions | Distribution - Ryan Sells |
| `J.P. Morgan` / `JP Morgan` / `Thomas Seo` | Distributions | Distribution - Tom Seo |
| `NYC DEPT OF FINA` / `NYS DTF` / `Carta` | Expenses | Admin |
| `FRANCHISE TAX BO` (with non-zero amount) | Expenses | Admin (CA franchise tax) |
| `Discern` | Expenses | Admin |
| `ALTAIR` | Expenses | Admin |
| `Brex` | Expenses | T&E, Software |
| `Frank` + `Rimerman` | Expenses | Professional Fees |
| `Transfer to Ready Credit` / `Transfer from Ready Credit` | Internal Transfer | Internal Transfer (MC↔MC) |
| *(no match)* | Uncategorized | Uncategorized |

### Zero-Dollar Transactions

Exclude all rows where `AMOUNT == 0` (filing-only ACH entries with no cash movement).

### Internal Transfer Handling

MC↔MC transfers (e.g. Ready Credit pushes) appear as an outbound in one account and an inbound in the other. Both legs are classified as `Internal Transfer` and **excluded from Summary subtotals** so they don't double-count. They still render in the per-entity detail tabs.

---

## Liquidation Proceeds Detection

Scan the LP accounts (Fund I LP, Fund II LP, Fund II-A LP) for inbound wires that are **not** MMMF/internal transfers. Exclude rows where `Description` contains `MMMF` or `VIA CBusOL`.

Also exclude rows where `Description` contains `PARTNER:=` — these are wire returns from inactive LP accounts (e.g. Randheep Fernando, Dominic Sood) and are not liquidation proceeds.

### Known Liquidation Mappings

Maintain this mapping table. Update it each quarter as Tom confirms new events:

| Inbound source keyword | Portfolio Company |
|---|---|
| `Acquiom` | *(check description for company name — e.g. "Teal Technologies acquired by Mercury")* |
| `Burgiss` or `MSCI PAYMENT` | Vantager |
| `Toucan Technologies` | Toucan Technologies |
| `SRS Acquiom` | *(check description)* |

If an inbound LP wire doesn't match any known pattern, include it with company name `⚠ UNKNOWN — confirm with Tom` in amber highlight.

Fund II LP and Fund II-A LP proceeds are **combined** into a single `Fund II + II-A` figure throughout.

---

## Output Workbook Structure

### Canonical output location

Default: `~/Library/Mobile Documents/com~apple~CloudDocs/Desktop/Dash Tax & Audit/Dash CPA Report - <range> <year>.xlsx`

This is iCloud-synced and is the single source of truth for the latest Dash CPA report. Any future run of the skill can refer to this file to see what was last produced.

### Auto-archival

On every run, **before** writing the new workbook, the script moves every existing `Dash CPA Report - *.xlsx` in the canonical folder into the `Archive/` subfolder. If a same-named archive file already exists, the moved file is suffixed with a `_YYYYMMDD_HHMMSS` timestamp so nothing is overwritten.

### File name

`Dash CPA Report - [PeriodRange] [YEAR].xlsx`

Where `[PeriodRange]` is derived from the active periods in the data:
- Single period: use the month range label — e.g. `Jan-Mar`, `Apr-May`, `Jun-Aug`, `Sep-Dec`
- Multiple periods: combine first and last — e.g. `Jan-May`, `Jan-Aug`

Period-to-label mapping:
| Period | File Label |
|---|---|
| 1/1–3/31 | Jan-Mar |
| 4/1–5/31 | Apr-May |
| 6/1–8/31 | Jun-Aug |
| 9/1–12/31 | Sep-Dec |

Examples: `Dash CPA Report - Jan-Mar 2026.xlsx`, `Dash CPA Report - Jan-May 2026.xlsx`

### Tab order
1. **Summary** — main view, SUMIFS-driven from Detail tabs
2. **DVF – Detail** — all in-scope DVF MC transactions, live-linked to DVF Raw Export
3. **DFM – Detail** — all in-scope DFM MC transactions, live-linked to DFM Raw Export
4. **Liquidation Events** — *(only when LP CSVs are supplied — omitted under `--skip-liquidation`)*
5. **DVF – Raw Export** — unmodified Citi DVF CSV (Date as Excel date, Amount/Balance as numbers)
6. **DFM – Raw Export** — unmodified Citi DFM CSV (same)

### Dynamic linking architecture

```
Raw Export  →  Detail  →  Summary
 (data)        (refs)    (SUMIFS)
```

- **Raw Export** tabs hold the typed source data. `DATE` is parsed to a real Excel date so `MONTH(...)` works; `AMOUNT (USD)` and `BALANCE (USD)` are real numbers.
- **Detail** tabs reference Raw Export cell-by-cell for Date / Description / Amount / Payment Method. The `Period` column is an Excel formula derived from the row's Date. `Section` and `Category` are static (Python-classified at build time, since Excel can't reasonably express keyword classification).
- **Summary** tab uses `SUMIFS` against the Detail tabs filtered by Section, Category, and Period. YTD = `SUM(period cells)`. The Total (DVF+DFM) columns = `=DVFcell + DFMcell`. Section subtotals = `SUM(data rows above)`.

Editing an amount or description in a Raw Export tab flows through to Detail and Summary automatically. Structural changes (adding or removing rows) still require re-running the script.

---

## Summary Tab Layout

### Column Structure (YTD column under each entity group)

```
A      | DVF P1..Pn | DVF YTD | gap | DFM P1..Pn | DFM YTD | gap | Total P1..Pn | Total YTD
```

- Col A: 32 wide (labels)
- Data + YTD cols: 16 wide each
- Gap cols: 3 wide

Each entity group contains one column per active period plus a trailing `YTD` column. The `DVF + DFM Total` group mirrors that layout (period cells + YTD). As periods accumulate through the year, new period columns are appended within each group; the YTD and gap structure shifts right accordingly.

Formula references for value cells:
- DVF period cell: `=SUMIFS('DVF – Detail'!D:D, 'DVF – Detail'!F:F, "<Section>", 'DVF – Detail'!G:G, "<Category>", 'DVF – Detail'!B:B, "<Period>")`
- DVF YTD cell: `=SUM(<DVF period range>)`
- DFM cells: same shape against `'DFM – Detail'`
- Total period cell: `=<DVF period cell>+<DFM period cell>`
- Total YTD cell: `=<DVF YTD cell>+<DFM YTD cell>`
- Subtotal row cell: `=SUM(<data rows above>)` for that column

### Row Structure

**Row 1** — Title bar: "Dash Fund — [YEAR] Income, Expenses & Distributions" (dark navy, white text, spans full width)

**Row 2** — Entity headers (mid navy, white text, centered):
- Merged across the full DVF group (period cols + YTD) → "Dash Venture Fund"
- Merged across the full DFM group (period cols + YTD) → "Dash Fund Management"
- Merged across the full Total group (period cols + YTD) → "DVF + DFM Total"

**Row 3** — Period column headers (dark navy, white text): each entity group has period labels followed by `YTD` as the final column

**Rows 4+** — Sections in this order:

#### INCOME (light blue section header)
- One row per income category
- **Total Income** subtotal row (medium blue, bold, top+bottom border)
- Blank spacer row

#### EXPENSES (light blue section header)
- One row per expense category
- **Total Expenses** subtotal row
- Blank spacer row

#### DISTRIBUTIONS (light blue section header)
- Distribution - Tom Seo
- Distribution - Ryan Sells
- **Total Distributions** subtotal row
- Blank spacer row

#### ⚠ UNCATEGORIZED — REVIEW REQUIRED (amber section header)
- Only rendered if uncategorized transactions exist
- Amber background, dark amber text throughout
- **Total Uncategorized** subtotal row
- Blank spacer row

#### LIQUIDATION PROCEEDS (soft green section header, spans A:F)
- **Sub-header row** (mid navy): Company | Fund I LP | [gap] | Fund II + II-A | [gap] | DVF + DFM Total
  - Columns snap exactly to the grid: Company=A, Fund I LP=B, gap=C, Fund II+II-A=D, gap=E, Total=F
- One data row per portfolio company (alternating white/light gray)
- **Total Proceeds** row (soft green subtotal, top+bottom border on value cells)

### Color Reference

| Element | Hex |
|---|---|
| Title / dark header bg | `1F2D3D` |
| Entity header bg | `2C4A6E` |
| Section header bg | `EAF2FB` |
| Subtotal row bg | `C8DCF0` |
| Liquidation section bg | `EEF4E8` |
| Liquidation subtotal bg | `C5DFB8` |
| Uncategorized bg | `FFF2CC` |
| Uncategorized font | `7B4F00` |
| Alternating row | `F5F5F5` |

### Number Format
`_($* #,##0.00_);_($* (#,##0.00);_($* "-"??_);_(@_)`

Show `None` (blank) instead of `$0.00` for zero values.

---

## Liquidation Events Tab

Columns: Date | Period | Portfolio Company | Received From | Fund | Amount

- One row per fund per event (e.g. Teal Technologies has two rows: Fund II LP and Fund II-A LP)
- Sorted by date, then company
- Total row at bottom (soft green subtotal)
- Column widths: Date=13, Period=12, Company=26, Source=28, Fund=14, Amount=16

---

## DVF – Detail and DFM – Detail Tabs

Columns: Date | Period | Description | Amount | Payment Method | Section | Category

- **Live-linked**: Date, Description, Amount, Payment Method cells are formula refs to the corresponding Raw Export row. Period is an Excel `IF/MONTH(...)` formula derived from the Date cell. Section and Category are static (Python-classified).
- Sorted by date ascending
- Alternating white/gray rows
- Uncategorized rows: amber background, dark amber font, bold Category cell
- Amount column: right-aligned, currency format
- Date column: `yyyy-mm-dd` format
- Column widths: Date=13, Period=12, Description=40, Amount=15, Method=16, Section=16, Category=28

The Period formula written into each Detail row:
```
=IF(MONTH(A<r>)<=3,"1/1–3/31",IF(MONTH(A<r>)<=5,"4/1–5/31",IF(MONTH(A<r>)<=8,"6/1–8/31","9/1–12/31")))
```

---

## DVF – Raw Export and DFM – Raw Export Tabs

The unmodified per-account Citi CSV as a formatted table — these are the **source of truth** that Detail and Summary reference.

- Dark header row (row 1 = title, row 2 = column headers in mid navy)
- Alternating white/gray data rows, font size 9
- Auto-width columns capped at 30
- **Typed cells**: `DATE` is written as an Excel date (formatted `yyyy-mm-dd`); `AMOUNT (USD)` and `BALANCE (USD)` are written as numbers (formatted with the standard currency format). This makes the downstream `MONTH(...)` formula and `SUMIFS(amount)` aggregation work natively.

---

## Uncategorized Handling

After generating the workbook, print to chat a summary of any uncategorized transactions:

```
⚠ UNCATEGORIZED ITEMS — please review:

Dash Venture Fund:
  [date] | [description] | $[amount]

Dash Fund Management:
  [date] | [description] | $[amount]
```

Ask Tom to confirm the correct category for each. Once confirmed, note the new mappings so they are applied on future runs (store them in this skill's keyword mapping table via the skill-creator update workflow).

---

## Liquidation Event Confirmation

After generating the workbook, if any inbound LP wires were flagged as `⚠ UNKNOWN`, list them and ask Tom to confirm the portfolio company name before finalizing.

---

## Verification

Before writing the Excel, `build_report.py` runs 5 deterministic checks against in-memory data. **Any failed check aborts the write** (exit code 2). The checks are validated against the Python data, not the workbook — openpyxl can't evaluate formulas, so workbook-level recalc is delegated to Excel on open. Since the formulas in Detail and Summary are mechanically derived from the verified Python data, accuracy carries through.

| # | Check | What it catches |
|---|---|---|
| 1 | **Citi running-balance tie-out** (full CSV) — newest balance − opening balance == Σ amounts | Any misread / dropped / duplicated row, since Citi's own running balance is the ground truth |
| 2 | **Per-account in-scope reconciliation** — Σ raw txns in [year, through] == Σ classified txns | Any row lost between parsing and classification |
| 3 | **No txn lost during aggregation** — Σ agg values + Σ Internal Transfer txns == Σ all classified | A classification bug that orphans a txn |
| 4 | **Per-section/period sum** — agg cell == sum of matching txns | Aggregation arithmetic bug |
| 5 | **Internal Transfers net to zero** — DVF + DFM combined Internal Transfer balance == $0.00 | Orphaned wash legs (RETURNED CHECK without a match; MC↔MC transfer without a counter-leg) |

Tolerance: half-cent (`$0.005`) on every comparison to absorb floating-point noise.

If a check fails, the script prints which check failed, the conflicting values, and exits without writing. The prior workbook in the canonical folder remains untouched until a clean run completes.

---

## Style Rules

- Font: Arial throughout
- No gridlines on any sheet (`showGridLines = False`)
- Summary uses Excel formulas (SUMIFS / SUM / cell math) so the workbook stays live-linked to Raw Export edits. Detail tab Date/Description/Amount/Method are also formula refs to Raw Export; Period is an Excel formula; Section/Category are static. Numbers are guaranteed accurate by the verification pass before write.
- Negative numbers display with parentheses via the currency format
- Show `None` (blank) instead of `$0.00` in Python-written cells; Excel-formula cells render their own zero behavior per the number format
- Never show `nan` — resolve all blank account names before writing
