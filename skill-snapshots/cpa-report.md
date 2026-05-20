---
name: cpa-report
description: Generate a quarterly CPA report for Dash Fund entities (Dash Venture Fund and Dash Fund Management) from a Citi Transaction Dashboard CSV export. Produces a formatted Excel workbook with a Summary tab (income, expenses, distributions, and liquidation proceeds by period and entity), a Liquidation Events detail tab, per-entity transaction detail tabs (DVF and DFM), and a Raw Export tab. Trigger when Tom says "run CPA report", "generate CPA report", "quarterly CPA report", "CPA Excel", "build the CPA report", "run the Dash CPA report", or any variant asking to produce the quarterly fund report for his CPA. Also trigger when Tom uploads a Citi Transaction Dashboard CSV and asks to process it into a report. Always trigger inline — no confirmation needed before acting.
---

# CPA Report Skill

Generates the quarterly Dash Fund CPA report from a Citi Transaction Dashboard CSV export. The output is a single Excel workbook delivered to `/Users/tomseo/Downloads/`.

---

## Inputs

1. **Citi Transaction Dashboard CSV** — exported from Citi's "Transaction Dashboard All" view, covering the full year-to-date (or whatever period Tom provides). Tom will upload this file or reference it by path.
2. **Liquidation event mappings** (optional) — Tom may specify which inbound wires map to which portfolio company (e.g. "Acquiom wire on 1/9 = Teal Technologies"). If not provided, use the keyword mapping table below and flag unknowns.
3. **New category mappings** (optional) — Tom may specify how to classify previously `Uncategorized` transactions. Add these to the mapping table for the current run.

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

### MC Account Logic

For each row in the Citi CSV, determine whether the DVF MC (6880014247) or DFM MC (6880010107) is the `From Account Number` or `To Account Number`:

- **Incoming to MC** (`To Account Number` = MC acct) → **Income / Management Fees**
- **Outgoing from MC** (`From Account Number` = MC acct) → classify by payee using the table below

### Category Keyword Mapping

| Payee pattern (in `To Account Name`) | Section | Category |
|---|---|---|
| `Ryan Sells` | Distributions | Distribution - Ryan Sells |
| `J.P. Morgan` or `Thomas Seo` | Distributions | Distribution - Tom Seo |
| `NYC DEPT OF FINA` or `NYS DTF` or `Carta` | Expenses | Admin |
| `FRANCHISE TAX BO` | *(exclude — $0 filings)* | — |
| `Brex` | Expenses | T&E, Software |
| `Frank` + `Rimerman` | Expenses | Professional Fees |
| *(no match)* | Uncategorized | Uncategorized |

### NaN Account Name Resolution

Citi sometimes leaves `From Account Name` blank for internal transfers, recording only the account number in the `Description` field. Resolve as follows:
1. Check if `From Account Number` is in the account map above → use the label
2. If not, scan the `Description` string for a known account number (e.g. `006880018539`) → use the label
3. If still unresolved → use `(Unknown)`

### Zero-Dollar Transactions

Exclude all rows where `Amount == 0` (these are filing-only ACH entries with no cash movement).

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

File name: `Dash CPA Report - [PeriodRange] [YEAR].xlsx`

Where `[PeriodRange]` is derived from the active periods in the data:
- Single period: use the month range label — e.g. `Jan-Mar`, `Apr-May`, `Jun-Aug`, `Sep-Dec`
- Multiple periods: combine first and last — e.g. `Jan-Aug`

Period-to-label mapping:
| Period | File Label |
|---|---|
| 1/1–3/31 | Jan-Mar |
| 4/1–5/31 | Apr-May |
| 6/1–8/31 | Jun-Aug |
| 9/1–12/31 | Sep-Dec |

Examples: `Dash CPA Report - Jan-Mar 2026.xlsx`, `Dash CPA Report - Jan-Aug 2026.xlsx`
Deliver to: `/Users/tomseo/Downloads/`

### Tab order
1. **Summary** — main view
2. **Liquidation Events** — detail for all proceeds
3. **DVF – Detail** — all DVF MC transactions
4. **DFM – Detail** — all DFM MC transactions
5. **Raw Export** — unmodified Citi CSV

---

## Summary Tab Layout

### Column Structure (no FY Total columns)

```
A          B              C(gap)  D                  E(gap)  F
Label    | DVF [period] |       | DFM [period]      |       | DVF+DFM Total [period]
```

- Col A: 32 wide (labels)
- Data cols (B, D, F): 16 wide each
- Gap cols (C, E): 3 wide

Add one data column per entity group for each active period. As periods accumulate through the year, new columns are appended — the gap/total structure shifts right accordingly.

### Row Structure

**Row 1** — Title bar: "Dash Fund — [YEAR] Income, Expenses & Distributions" (dark navy, white text, spans full width)

**Row 2** — Entity headers (mid navy, white text, centered):
- B:B merged → "Dash Venture Fund" (expands to cover all DVF period cols)
- D:D merged → "Dash Fund Management" (expands to cover all DFM period cols)
- F:F merged → "DVF + DFM Total" (expands to cover all total period cols)

**Row 3** — Period column headers (dark navy, white text): period label under each entity group

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

- Sorted by date ascending
- Alternating white/gray rows
- Uncategorized rows: amber background, dark amber font, bold Category cell
- Amount column: right-aligned, currency format
- Column widths: Date=13, Period=12, Description=40, Amount=15, Method=16, Section=16, Category=28

---

## Raw Export Tab

Paste the full unmodified Citi CSV as a formatted table:
- Dark header row (row 1 = title, row 2 = column headers in mid navy)
- Alternating white/gray data rows, font size 9
- Auto-width columns capped at 30

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

## Style Rules

- Font: Arial throughout
- No gridlines on any sheet (`showGridLines = False`)
- Zero formula errors — all values are Python-computed and written as literals; no Excel formulas needed
- Negative numbers display with parentheses via the currency format
- Never show `nan` — resolve all blank account names before writing
