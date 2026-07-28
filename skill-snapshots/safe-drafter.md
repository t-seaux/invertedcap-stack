---
name: safe-drafter
description: >-
  Generate an execution-ready YC Post-Money SAFE (Valuation Cap Only) PDF from
  deal parameters. Tom gives the entity legal name, investment amount, and
  post-money cap (plus optionally state of incorporation, company signatory,
  address, email, date, stage); the skill fills the pinned YC template
  cross-run, converts docx→PDF via a Google Drive round-trip (headless, no
  Word), runs a deterministic publish gate (zero leftover bracketed
  placeholders, all required values present), and saves the PDF to ~/Downloads
  with the deal naming convention. Missing optional fields are left blank in
  line; the signing date stays a blank underline unless given. Also owns every
  deal-doc VERSION event — a new turn, vFinal awaiting execution, or the
  executed set, for the SAFE or the side letter: same-name Drive replace into
  the round folder + swap the Notion Opp's Deal Docs chip (replace mints a new
  file ID, so the chip is refreshed every time). Manual-only.
  Trigger on "draft a SAFE for [company]", "generate the SAFE", "populate a
  SAFE", "prep the SAFE for [company] — $X on $Y cap", "SAFE for [company]",
  "here's the signed/executed SAFE", "new turn of the [company] docs", "final
  docs for [company]", or any variant producing a YC SAFE or handing over an
  updated deal-doc version. NOT for drafting side letters (negotiated per deal,
  no template skill — but their version handling routes here) and NOT for
  Discount/MFN SAFE variants (cap-only form only — say so and stop if asked).
---

# safe-drafter

Produce an execution-ready YC Post-Money SAFE (Valuation Cap Only) PDF from deal
parameters. Everything deterministic lives in `scripts/fill_safe.py` — this
skill is: collect parameters, run the script, deliver the PDF path.

## Inputs

Required — if any is missing from Tom's message, ask before running:

| Param | Flag | Example |
|---|---|---|
| Entity legal name | `--company` | `"AgentBay, Inc."` (exact legal name incl. suffix) |
| Investment amount | `--amount` | `1000000` (dollars, no commas) |
| Post-money cap | `--cap` | `12500000` |

Optional — **leave blank in line if Tom doesn't have them; never ask for these,
never invent them**:

| Param | Flag | Default |
|---|---|---|
| State of incorporation | `--state` | `Delaware` |
| Governing law | `--governing-law` | same as `--state` — NEVER left as the YC bracket |
| Company signatory name / title | `--signer-name` / `--signer-title` | blank line |
| Company address / email | `--company-address` / `--company-email` | blank line |
| Date of Safe | `--date` | blank underline (filled at signing) |
| Stage (filename only) | `--stage` | `Pre-Seed` |
| Filename short name | `--short-name` | legal name minus Inc/LLC/Corp suffix |
| Output directory | `--out-dir` | `~/Downloads` |

Investor-side constants (Inverted Capital I, LP / Thomas Seo / Managing Member /
365 Bridge Street, 8PRO, Brooklyn, NY 11201 / tom@invertedcap.com) are hardcoded
in the script — edit the constants there if the fund entity ever changes.

## Procedure

1. Extract parameters from Tom's message. Confirm nothing only if a **required**
   field is missing or ambiguous (e.g. "1m" → 1000000 is fine; a missing cap is
   not).
2. Run:
   ```bash
   python3 ~/.claude/skills/safe-drafter/scripts/fill_safe.py \
     --company "<Legal Name, Inc.>" --amount <n> --cap <n> [optional flags]
   ```
3. On `OK: <path>` + `DRIVE: <url>` — report both, plus the key terms you filled
   (amount, cap, state, governing law) so Tom can eyeball them. The script
   auto-uploads the PDF to `Deal Docs/<Company>/<Stage> (<Mon YYYY>)/` per the
   deal-docs layout (idempotent createFolder; `--no-drive` to skip). If a Notion
   Opportunity exists for this deal, add the Drive URL as a chip on its
   `Deal Docs` property (`notion_files_property.py` — see the version section
   below); no Opp yet is fine, the chip lands with the first version event
   after the Opp exists.
4. On `GATE FAIL` — do NOT hand over any artifact. Fix or escalate:
   - `template drift` → the pinned template no longer matches the expected
     placeholders; see Template provenance below.
   - anything else → report the failure verbatim to Tom.

## Every new version — Drive replace + Notion chip swap

Applies to EVERY version event on ANY deal doc for the Opp — a new turn of the
SAFE **or side letter**, a vFinal awaiting execution, the executed set (file
drop, or a Gmail attachment — pull it via the Gmail Attachment Saver per
`shared-references/gmail-attachment-saver.md` if needed). Side-letter drafting
is out of scope below, but its version handling lands here.

1. Find the current file's exact filename: `listFolder` on the round folder via
   the Drive Manager Apps Script, or reuse the name from this conversation.
2. Re-upload under that SAME name — the Apps Script upload action
   trashes-and-replaces on exact-name match, so the prior version is replaced
   in place:
   ```bash
   python3 ~/.claude/skills/safe-drafter/scripts/upload_deal_doc.py <new_version.pdf> \
     --company <Short Name> --stage <Stage> --round-month "<Mon YYYY of the existing round folder>" \
     --name "<exact existing filename>.pdf"
   ```
3. **Swap the Notion chip — every time, not just executed.** The replace mints
   a NEW Drive file ID, so any existing chip on the Opp's `Deal Docs` property
   is now a dead link. Resolve the Opp (semantic search, then SQL `Name LIKE`
   fallback per CRM conventions), read its `Deal Docs` property to find the
   stale chip for this doc, then:
   ```bash
   python3 ~/.claude/scripts/notion_files_property.py --page-id <opp> --prop "Deal Docs" --url "<stale chip url>" --remove
   python3 ~/.claude/scripts/notion_files_property.py --page-id <opp> --prop "Deal Docs" --url "<new fileUrl>" --label "<filename>"
   ```
   First version with no chip yet → skip the remove, just add.
4. **Superseded drafting turns relocate, they don't vanish.** When an executed
   version supersedes interim turns (redlines, revised-clean drafts), move each
   superseded file into the company's Diligence Materials Drive folder (the
   `[G DRIVE] <Company> Diligence Materials` chip on the Opp points at it) via
   the Apps Script `moveFile` action — file ID survives a move — then add a
   chip for it on the Opp's `Diligence Materials` property and remove its
   `Deal Docs` chip. End state: `Deal Docs` = current version of each doc only;
   the drafting history lives under Diligence Materials.
5. Confirm to Tom with the Drive URL and note the Opp chips are current.

## Built-in gates (in the script — do not re-verify manually, do not bypass)

- Every placeholder replacement asserts its expected occurrence count
  (template-drift tripwire).
- Post-render, the output PDF text must contain **zero** `[Bracketed]`
  placeholders and every required filled value (amount, cap, company, state,
  governing law, investor entity). Origin: AgentBay 2026-07-27, where counsel's
  exec copy shipped with `[Governing Law Jurisdiction]` unfilled.
- docx→PDF is a Drive round-trip (upload-with-convert → export PDF → delete the
  script's own scratch Doc) via the gmail-reconciler SA with DWD as
  tom@invertedcap.com. The scratch-Doc delete is cleanup of a file the script
  itself created seconds earlier — not a Tom-data deletion.

## Template provenance

`assets/postmoney_safe_cap_only.docx` — YC "Postmoney Safe - Valuation Cap Only"
v1.2, downloaded 2026-07-27 from ycombinator.com/documents
(sha256 `185d24f5…3649c4`). Verified word-identical to the live YC form on that
date. If YC ships a new version (or the drift tripwire fires): re-download from
ycombinator.com/documents, word-diff old vs new, show Tom the diff before
swapping the asset, then update this section's date/sha.

## Out of scope

- **Side letters** — negotiated per deal; draft from the most recent executed
  side letter in `Deal Docs/` as a starting point, in conversation, not here.
- **Discount / MFN SAFE variants, non-US SAFEs** — not templated. Tell Tom and
  stop rather than improvising on the cap-only form.
