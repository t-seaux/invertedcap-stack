---
name: nysif-payroll-report
description: >-
  Fill out the annual NYSIF Disability Benefits / Paid Family Leave payroll report (form DBL-205, policy
  DB #7975593) for Tom's household employee, directly as a completed PDF. Trigger when Tom forwards or
  screenshots a NYSIF "Payroll Report" / "Action Required - Insufficient Payroll Report" email (from
  dbpolicy@nysif.com or an @nysif.com underwriter), says "fill out the NYSIF form", "NYSIF payroll report",
  "do the DB/PFL report", or drops GTM payroll registers + the NYSIF form PDF with intent to complete it.
  Inputs: three GTM Payroll Register PDFs (full policy year 9/5→9/5, plus splits 9/5→1/1 and 1/1→9/5) and
  the NYSIF form PDF — usually all in ~/Downloads; if any are missing, ask Tom for exactly those docs,
  then complete the rest autonomously. Extracts GROSS wages from each register's "Employee
  Totals" row (never net / Direct Deposit / YTD), validates the splits sum to the full-year total, fills
  every field via fill_form.py, renders to verify, and saves the completed PDF to Downloads for Tom to
  sign and reply to NYSIF. Manual-only (Mode C).
---

# nysif-payroll-report

Tom employs a nanny (Dularie Persaud — reported in the **Female** column) through GTM Payroll Services
(client #189742). His NYSIF Disability Benefits & Paid Family Leave policy is **DB #7975593**, policy year
**9/5 → 9/5**. Every fall NYSIF requires a DB/PFL payroll report (form **DBL-205**); Part B splits the year
at January 1 because the PFL wage cap resets each calendar year.

History: Tom's original filing omitted Part B three years running (Sept 2024, Dec 2025, Sept 2026), each
time triggering an "insufficient payroll report" email with the form re-attached and Part B highlighted.
Resolution each time: complete the form and reply to **dbpolicy@nysif.com** with the amended PDF attached
("Please see attached."). NYSIF acks next morning; done. This skill produces that completed PDF — ideally
for the *original* filing so the bounce never happens.

## Inputs (usually in `~/Downloads`, which symlinks to iCloud Downloads)

1. **Three GTM Payroll Register PDFs** — filenames like `..._PayrollRegister_189742_<start>_<end>.pdf`:
   - full policy year: `<9/5/YY>` → `<9/5/YY+1>`
   - first split: `<9/5/YY>` → `<1/1/YY+1>`
   - second split: `<1/1/YY+1>` → `<9/5/YY+1>`
2. **The NYSIF form PDF** — either the blank/highlighted DBL-205 NYSIF emailed back, or Tom's copy.
   If Tom already hand-filled values, the fill script strips them and re-fills (see Step 3).

**Intake flow (Tom's standing instruction, 2026-09-16):** when invoked, first check `~/Downloads` for the
four PDFs. If any are missing, **ask Tom for exactly the ones needed** — name each explicitly (e.g. "GTM
Payroll Register 9/5/26 → 1/1/27", "the NYSIF form PDF from their email") so he can pull them from the GTM
portal / NYSIF email in one pass. Once all four are in hand, run Steps 1–4 end-to-end yourself with no
further questions or confirmations — the deliverable is the completed, filled PDF in Downloads.

## Step 1 — Extract gross wages from each register

For each register: `pdftotext -layout "<file>" -` and find the **`Employee Totals:`** row. The register's
column layout is `Earnings: Hours | Amount (Current) | Hours | Amount (YTD) | Deductions: Current | YTD |
Taxes: Current | YTD`. The figure you want is the **Current-period gross earnings Amount** — on the
Employee Totals row it is the first large dollar figure (e.g. `15,072.86` in
`Employee Totals: 0.00  0.00  15,072.86  0.00  24,216.13 | 12,851.55  21,478.92 | 1,785.65  2,737.21`).

⛔ **Traps — all three burned or nearly burned a real filing (2026-09-16):**
- **NOT the Deductions/Direct Deposit column** (net pay). Tom hand-filled the 2026 form with net figures
  ($12,852/$20,898/$33,750 instead of $15,073/$23,562/$38,635) and it had to be corrected.
- **NOT the YTD column** — it's calendar-year-to-date at report-generation time and repeats identically
  across all three registers. If two registers show the same figure, you grabbed YTD.
- **NOT `Total Payroll Debit`** at the bottom (that's net funding, not wages).

Also read the `N EMPLOYEES` line next to the check count — that's the employee count for every count field.

## Step 2 — Validate and compute the form values

1. **Sum gate:** split1 + split2 must equal the full-year register total **to the penny**. If not, the
   registers don't cover identical check ranges — stop and tell Tom which figure is off; do not fill.
2. **Round** each split half-up to whole dollars; **total = sum of the rounded splits** (guarantees the
   form's internal consistency even when the raw total rounds differently).
3. **Caps** — read the caps printed on THIS year's form (they change annually; never reuse a prior year's):
   - Part A.2 capped wages = `min(full-year gross, Part A cap)` — with one employee well above the DBL cap
     this is normally just the cap (e.g. `17680`).
   - Part B.1b = `min(split1, B1 cap)`, B.2b = `min(split2, B2 cap)` — historically the splits sit under
     the PFL caps, so actuals go in.
4. **Premium contribution (Part A.4):** check the register's Deductions block — if there is no DBL/PFL
   employee deduction line (only Direct Deposit), answer **No**. (2025 and 2026 filings: No.)
5. NYSIF's own consistency rule (their bounce email states it): DBL gross ≥ PFL gross, and gross ≥ capped
   sums. Using the same gross figure for Part A.3 and Part B.3a satisfies it by construction.

## Step 3 — Fill the PDF

Run the script in this skill's directory:

```bash
python3 <skill-dir>/fill_form.py \
  --form "<NYSIF form>.pdf" --out "<same-name> (filled).pdf" \
  --a-capped <partA2> --gross <total> --b1 <split1> --b2 <split2> \
  --date <today MM/DD/YY> --contribute no --employees 1
```

- Coordinate map targets form revision **DBL-205 (10-22)**. The script strips any pre-existing FreeText
  value annotations at target positions (so a half-filled form is safe input) and preserves everything
  else — including Tom's signature widget if present.
- **Never fabricate the signature.** If the source PDF has no signature, leave the line blank; Tom signs
  in Preview or via NYSIF's DocuSign envelope.
- Save the output to `~/Downloads`.

## Step 4 — Verify, then hand off

1. Render page 1 of the output (Read the PDF) and visually confirm: every field populated, no doubled
   text, counts in the Female column, splits sum to both gross fields. If the layout looks shifted, NYSIF
   revised the form — fix the coordinate map in `fill_form.py` from the new layout, refill, and note the
   revision here.
2. Tell Tom: the three extracted figures with which register each came from, the sum check, the caps
   applied, and the output path. He replies to the NYSIF email himself (to **dbpolicy@nysif.com**, or the
   underwriter who wrote) — historic body is one line: "Please see attached." Per standing rules, draft
   only if he asks; never send.
