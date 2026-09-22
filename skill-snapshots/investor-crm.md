---
name: investor-crm
description: |-
  Build and maintain an Investor CRM (a.k.a. VC CRM) — the fundraise tracker Tom hands a founder (# / Name / Firm / Location / Status / Note(s), status color bands), one sheet per company in the Drive Portfolio folder. TWO modes. (A) CREATE from the company's Notion Opportunity intro relations — "create an investor CRM for [Company]", "spin up a VC CRM for [Company]", "build the investor CRM for [Company]", "make an investor tracker for [Company]", or any create/spin-up/build phrasing + investor/VC CRM + a company. (B) ADD / EDIT an existing CRM — "add [X] to [Company]'s investor CRM", "add [fund] to the investor CRM", "mark [X] as [status/passed/avoid] in [Company]'s CRM", "remove [X] from the CRM", "update [X]'s status", or any incremental edit. Manual-only, always inline — no confirmation before acting. NOT coinvestor-recommender (which suggests who to bring in); this renders/maintains the tracker sheet. Does NOT sweep Gmail/iMessage.

---

# Investor CRM

Build and maintain a company's fundraise tracker sheet in Tom's format. **Read
`references/spec.md` first** — it holds the status map, Notion sourcing, naming,
cache location, and the build command. Key rule: the JSON row-cache is the source
of truth; the sheet is a render of it (the uploaded `.xlsx` can't be read back).

Two modes — pick by the trigger.

## Mode A — Create

"create / spin up / build an investor (or VC) CRM for **[Company]**"

1. **Resolve the Opportunity** in Notion by company name (see spec §Notion sourcing).
   Disambiguate on Fund/Status if the name is shared. If no Opp exists, tell Tom and stop.
2. **Pull the four intro-relation buckets** off the Opp and fetch each related People
   page. Map `Name/Company/City → Name/Firm/Location`, bucket → Status, and write a
   short factual `note` (spec §Notion sourcing). Dedup cross-bucket people (spec §Dedup).
3. **Order** the rows by status: Intro made → Outreach → Qualified → Passed / NR.
   (No cohort separator on a fresh Create — all rows are Tom's intros.)
4. **Write the cache** `data/<slug>.json` with the row list.
5. **Build + render** (spec §Build + upload): run `scripts/build_crm.py`, read the
   emitted `b64`, `create_file` it (converts to a **native Sheet**), retrying on
   `invalid argument`. Retire the prior render via the meta sidecar, then save meta.
6. **Report** the Sheet URL, a one-line count by status, and flag anything low-confidence
   (blank locations, cross-bucket dups, missing People fields).

## Mode B — Add / Edit

"add **[X]** to [Company]'s investor CRM", "mark [X] as [status] / avoid / passed",
"remove [X]", "update [X]'s status/note"

1. **Load the cache** `data/<slug>.json`. If it's missing but the sheet exists, rebuild
   the cache from Notion (Mode A steps 1–4) first, then apply the edit. If neither
   exists, this is really a Create — run Mode A.
2. **Apply the edit** to the row list in memory:
   - *Add:* append the new entry. If it's a founder-sourced / inbound fund (not from the
     Notion relations), place it in the second cohort and set `sep_top: true` on the
     first such row (only one row carries the separator). Fill what's known; leave
     `location`/`note` blank if Tom didn't give them — he'll fill in.
   - *Mark/Update:* find the row by name/firm (case-insensitive), change `status`/`note`.
     Use the status vocabulary in spec (e.g. `Avoid`, `Passed / NR`).
   - *Remove:* drop the row.
   Keep ordering sane (spec §Ordering); renumbering is automatic (the script numbers rows).
3. **Save the cache**, then **re-render** (spec §Build + upload): `build_crm.py` →
   `create_file` (new native Sheet) → retire the prior sheet via the meta sidecar →
   save meta.
4. **Report** what changed and the URL.

## Notes

- One company → one native Sheet `<Company>: Investor CRM` in Portfolio. Each render
  is a fresh `create_file`; the meta sidecar tracks the live id so the previous render
  is retired automatically — re-running is always safe.
- Data edits go through this skill; **formatting** changes go in `scripts/build_crm.py`
  (then re-run for affected companies) — never hand-edit the sheet.
- No Gmail/iMessage sweep — the founder's own inbound funds get added manually via Mode B.
