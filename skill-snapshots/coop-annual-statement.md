---
name: coop-annual-statement
description: >-
  Year-end package Tom sends as treasurer of 25 Garden Place Corp: the "<YEAR> Financial Statement" xlsx
  (monthly operating income/expenses + reserve fund, prior-treasurer layout) and the shareholder memo stating
  NYC Property Taxes the co-op paid that calendar year (total and per apartment) for their personal returns.
  Also covers the late-November year-end projection / assessment email that precedes it. Built from the
  coop-finances P&L workbook via build_statement.py. Trigger on "annual coop statement", "coop financial
  statement", "coop tax memo", "send the coop year-end", "property tax memo for shareholders", "coop year-end
  assessment", or the January tax-checklist item for the co-op. Manual; Tom sends the email.
---

# coop-annual-statement

Annual obligation Tom inherited from Hattie (prior treasurer). Upstream data comes from `coop-finances`, which maintains the canonical P&L workbook. This skill only reads that workbook, packages the year, and drafts the emails.

## What gets sent (reference: Hattie, 2026-02-08, "2025 Financial Statement")

- **To:** all shareholders at their personal addresses. Pull the list fresh from the last co-op-wide thread each year and do not hardcode it. As of 2026-09 that list is Greg + Sandy Maltzman, James Frankel, Lucy Boswell, Elsie Kenyon, Tom (thomas.seo@outlook.com), and RJ + Hattie. Known address drift: Greg uses both `gregorymaltzman@` and `gregory.maltzman@`, and RJ's address is `expjtg@earthlink.net`. Hattie's Feb email used `.com`, which is probably a typo.
- **Also send to:** any shareholder who sold during the year. They need the tax figure too (see Jeri Boylan, the prior Apt 2 owner, who asked in 2026-02).
- **Attachment:** `<YEAR> Financial Statement.xlsx`, one sheet `<YEAR> FS`. Columns are months JAN–DEC + TOTAL. Sections: OPERATING ACCOUNT → INCOME (Balance Forward, Maintenance, Assessments, Repair Settlement) → EXPENSES (taxes, utilities, cleaning, insurance, bank fees, repairs, accounting, adjustments) → End-of-Year BALANCE → RESERVE FUND. Footnotes go under the balance, such as "Some expenses from <prior year> could not be posted until <year>".
- **Body** (reuse Hattie's wording; the co-op expects it):

  > Dear 25 Garden Place Corp Shareholders,
  >
  > Our 25 Garden Place Corp <YEAR> Financial Statement is attached. It shows NYC Property Taxes paid in <YEAR>.
  >
  > Please note that we paid a total of $<TOTAL> in NYC Property Taxes in <YEAR> ($<PER_APT> per apartment). Please save this memo for your tax purposes.
  >
  > [Assessment true-up line if any, e.g. "You will each receive a partial refund of $380 from our December <YEAR> assessment."]
  >
  > Best regards,
  > Tom

- **Timing:** Hattie's cadence was a year-end update + assessment call in late November, then the final statement once the December Citi statement is reconciled (sent Feb 8 for 2025). Target the **end of January** because shareholders need the number to file.

## Steps

1. **Close the year in coop-finances.** Ingest the December Citi CHK-7926 statement and the Reserve statement. Clear every `_Raw` row with `Status=flagged`. The builder refuses to write until the ledger reaches December.
2. **Build:** `python3 ~/.claude/skills/coop-annual-statement/build_statement.py --year <YEAR>`. It writes to `~/Library/Mobile Documents/com~apple~CloudDocs/Desktop/Garden/Annual Statements/<YEAR> Financial Statement.xlsx` and prints JSON (totals, EOY balance, property-tax payments by txn date, per-apartment figure, gates). Use `--dry-run` for a preview and `--allow-partial` for a mid-year draft that is never circulated.
   - **Footnotes** are generated automatically under the End-of-Year BALANCE, with `*` / `**` markers on the line they explain, the way Hattie used "ѫ":
     - (a) the Balance Forward explanation, whenever `Prior-Year Accrual` rows exist (checks that cleared this year but were expensed last year);
     - (b) every NYC Property Tax payment with its date and amount, plus the total and per-apartment figure, so buyers and sellers can split by date.

     Add others with `--note "Insurance=Two-year policy prepaid in April"` (repeatable). The JSON output includes `footnotes`, so the email can reference them.
3. **Gates. Every one must be clean before drafting:**
   - `INCOMPLETE` / `flagged` → go back to step 1.
   - `TAX MISMATCH` → a property-tax payment's Month column crosses the year line. The memo number is the one keyed off the **payment date**, because shareholders deduct taxes paid in the calendar year.
   - List each property-tax payment (date, amount) back to Tom and check it against NYC DOF's account history for the BBL. An equal split ÷4 assumes equal shares per apartment, so confirm it against the stock certificates or proprietary leases once, then note the result here.
   - **Balance Forward continuity:** this year's Balance Forward = last year's **circulated** End-of-Year BALANCE, never the bank balance (Tom's rule, 2026-09-27). Anything already expensed or accrued in the circulated statement that clears the bank the next year (for example, assessment refund checks) gets posted in `_Raw` as `Prior-Year Accrual`, which is excluded from the P&L so it isn't counted twice. Precedent: 2026 opens at $493 (Hattie's 2025 EOY), and the four $380 refund checks cashed in Feb 2026 are `Prior-Year Accrual`.
4. **Assessment true-up.** If a year-end assessment ran a surplus, compute the refund per apartment and include the line in the email. Hattie's version: $500/apt December assessment → $380 refunded, $1,520 total. Record the refund checks under Adjustments.
5. **Ownership changes.** If an apartment changed hands during the year, the per-apartment figure covers the whole year. List the payment dates in the email to the buyer and seller so each can take the payments made while they held the shares, and tell them to confirm with their CPA. Precedent: Tom and Elsie closed on Apt 2 (from Jeri Boylan) on **2025-07-31**. Of the 2025 payments in Hattie's statement (Apr $13,163, Jun $11,222, Oct $11,209), only Oct falls after closing: $2,802.25 per apartment for Tom and Elsie versus $6,096.25 for Jeri. Verify exact payment dates against the bank before giving these to the CPA. This skill never gives tax advice beyond the numbers.
6. **Draft the email** (Gmail draft from Tom's personal account; Tom sends) with the xlsx attached. Save the xlsx and a PDF copy in `Garden/Annual Statements/`.
7. **Capital improvements.** Append any new board-approved capital improvements to `references/CAPITAL_IMPROVEMENTS.md`. Former shareholders ask for these for cost basis.

## Late-November year-end update (precedes the statement)

Hattie met with the President and the incoming Treasurer around Thanksgiving, then emailed a projection. The template is her 2025-11-29 email, "25 Garden Place Corp <YEAR> Financial Update and Assessments". It covers:
- a projected Nov/Dec shortfall (unbilled bills estimated from the prior year's actuals) → a per-apartment operating assessment, refunded if in surplus;
- a per-apartment assessment to fund the **January property-tax installment** (due Jan 15, or interest accrues) so it is paid in early January. The timing of that payment moves the next year's memo number: a payment made in December counts in the earlier year.

Run `build_statement.py --year <YEAR> --allow-partial --dry-run` for the YTD base, then project Nov/Dec from the prior year's same months.

## Files

- `build_statement.py`: builder plus gates (reads `coop-finances` workbook tabs / `_Raw`).
- `references/CAPITAL_IMPROVEMENTS.md`: cost-basis ledger for shareholders.
- Reference artifact: Hattie's 2025 statement, archived at `Garden/Annual Statements/2025 Financial Statement (Hattie).xlsx`.
