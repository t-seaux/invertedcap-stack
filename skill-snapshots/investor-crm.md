---
name: investor-crm
description: |-
  Build and maintain an Investor CRM (a.k.a. VC CRM) – the fundraise tracker Tom hands a founder, one native Google Sheet per company in the Drive Portfolio folder, in Tom's locked house template (# / Name / Firm / Title / Location / Stage / Strength / Status / Notes (Claude) / Notes (TS)). THREE modes. (A) CREATE from the company's Notion Opportunity intro relations – "create an investor CRM for [Company]", "spin up a VC CRM for [Company]", "build the investor CRM for [Company]", "make an investor tracker for [Company]". (C) FROM A SCAN – turn a coinvestor-recommender Investor Scan into the sheet: "make an investor CRM for [Company] from the scan", "put the recs in a sheet", "turn the investor scan into a CRM", "add the weaker fits for review". (B) ADD / EDIT an existing CRM – "add [X] to [Company]'s investor CRM", "mark [X] as [status/passed/avoid]", "remove [X] from the CRM", "sub [X] in for [Y]", "update [X]'s status". Manual-only, always inline – no confirmation before acting. NOT coinvestor-recommender (which picks who to bring in); this renders/maintains the tracker sheet. Does NOT sweep Gmail/iMessage.
---

# Investor CRM

**Read `references/spec.md` first** – template, field rules, Notion sourcing, Drive path. ONE renderer for every
sheet: `scripts/crm_sheet.py` (template + firm/title rules + Drive v3 API as Tom). Never hand-build a workbook, never
paste base64 into a Drive tool, never create a second sheet for the same company.

**The live sheet beats the cache.** Tom edits CRMs directly (cuts = his human review; his Strength; his
Notes (TS)). Before ANY re-render: `crm_sheet.py readback <file_id> data/<slug>.json`, then edit, then
`crm_sheet.py push data/<slug>.json` – which replaces the sheet IN PLACE (same URL; his open tab keeps working).
Rows Tom deleted land in the cache's `cut` list and are never re-added unless he asks.

## Mode A – Create (from Notion)

1. Resolve the Opportunity (spec §Notion sourcing); no Opp → tell Tom and stop.
2. Pull the four intro-relation buckets → rows (spec maps bucket → Status). Dedup cross-bucket people.
3. Write `data/<slug>.json` (spec §Cache), sections = one main section ordered by status.
4. `crm_sheet.py push data/<slug>.json` → report the URL + count by status + low-confidence fields.

## Mode C – From a scan (coinvestor-recommender output)

1. Find the scan: the session-scan ledger file (`~/.claude/scripts/network_scans/<id>.json`) for that company.
2. `scripts/from_scan.py --scan <file> --company "<Co>" --round <Seed|Series A…> [--review] > data/<slug>.json`
   – main = strong fits (one row per lead, teammates in the note), `--review` adds "Weaker fits (for review)".
3. Blank LOCATION → look the person up (ContactOut LinkedIn enrich) and fill it; never guess.
4. `crm_sheet.py push` → report URL, counts, Claude's Stage calls, and what Tom fills (Strength, Notes (TS)).

## Mode B – Add / Edit

1. `readback` first (always). Missing cache but sheet exists → readback builds it.
2. Edit rows in the cache: add (fill what's known), status/notes changes, remove, "sub X in for Y" (replace the
   row; Tom's Strength/Notes (TS) for Y do NOT transfer to X). Title = functional role; firm via `firm_name()`.
3. `push` → report what changed + URL.

## Notes

- Formatting changes go in `crm_sheet.py` (and its test) – never per sheet. When Tom tweaks a sheet's look and
  says "log it", export the sheet, diff against the template, update `crm_sheet.py` constants + tests.
- After Tom's review, his calls are coinvestor-recommender Step 8 inputs (Qualified roster in Notion,
  finalization-log cuts, KB calibration from his Strength edits) – hand them over, don't keep them here.
- `scripts/build_crm.py` is the retired pre-2026-10-13 format (color bands) – don't use it.
