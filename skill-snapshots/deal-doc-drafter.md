---
name: deal-doc-drafter
description: >-
  Generate execution-ready Inverted Capital deal docs — the YC Post-Money SAFE
  (Valuation Cap Only) and the Inverted SAFE Term Sheet — as PDFs from deal
  parameters. Tom gives the entity legal name, Inverted's purchase amount, and
  the post-money cap (plus, for a term sheet, the maximum raise and expiration
  date); the skill fills the pinned Google Doc template cross-run, exports PDF,
  runs a deterministic publish gate (zero unfilled tokens, all required values
  present), and saves to ~/Downloads as
  `<Short> x Inverted Capital - <Stage> <SAFE|Term Sheet> <MM.DD.YY>.pdf`.
  Fields deliberately left open render as [TBD] highlighted yellow. Also owns
  every deal-doc VERSION event — a new turn, vFinal awaiting execution, or the
  executed set, for the SAFE, term sheet, or side letter: same-name Drive
  replace into the round folder + swap the Notion Opp's Deal Docs chip (replace
  mints a new file ID, so the chip is refreshed every time). Manual-only.
  Trigger on "draft a SAFE for [company]", "generate the SAFE", "term sheet for
  [company]", "save down a PDF of the term sheet", "we agreed terms with
  [company]", "prep the docs for [company] — $X on $Y cap", "here's the
  signed/executed SAFE", "new turn of the [company] docs", "final docs for
  [company]", or any variant producing an Inverted SAFE or term sheet, or
  handing over an updated deal-doc version. NOT for drafting side letters
  (negotiated per deal, no template skill — but their version handling routes
  here) and NOT for Discount/MFN SAFE variants (cap-only form only — say so and
  stop if asked).
---

# deal-doc-drafter

Produce execution-ready Inverted deal-doc PDFs from deal parameters. Everything
deterministic lives in `scripts/fill_deal_doc.py` — this skill is: collect
parameters, run the script, deliver the PDF path.

## Inputs

Required for both doc types — if any is missing from Tom's message, ask before
running:

| Param | Flag | Example |
|---|---|---|
| Entity legal name | `--company` | `"Fair Appeal, Inc."` (exact legal name incl. suffix) |
| Inverted purchase amount | `--amount` | `1000000` (dollars, no commas) |
| Post-money cap | `--cap` | `10000000` |

Term sheet additionally requires:

| Param | Flag | Example |
|---|---|---|
| Maximum raise | `--round-size` | `2000000` |
| Expiration date | `--expiration` | `"August 21, 2026"` |

Optional — **leave open if Tom doesn't have them; never ask for these, never
invent them**:

| Param | Flag | Default |
|---|---|---|
| State of incorporation | `--state` | `Delaware` |
| Governing law | `--governing-law` | same as `--state` — NEVER left as the YC bracket |
| Company signatory name / title | `--signer-name` / `--signer-title` | `[TBD]` |
| Option pool % (term sheet) | `--option-pool` | `[TBD]` |
| Stage (filename + Drive folder) | `--stage` | `Pre-Seed` |
| Filename short name | `--short-name` | legal name minus Inc/LLC/Corp suffix |
| Output directory | `--out-dir` | `~/Downloads` |

Investor-side constants (Inverted Capital I, LP / Thomas Seo / Managing Member /
365 Bridge Street, 8PRO, Brooklyn, NY 11201 / tom@invertedcap.com) are already
typeset **in the templates** — not injected by the script. If the fund entity
ever changes, edit the Google Docs, not the code.

**Blanks that stay blank.** The SAFE's Date-of-Safe, company address, and
company email are bare underlines in the template with no token; they are filled
at signing. Both signature date lines on the term sheet are the same. Don't try
to fill them.

## Procedure

1. Extract parameters from Tom's message. Confirm nothing only if a **required**
   field is missing or ambiguous (e.g. "1m" → 1000000 is fine; a missing cap is
   not). When terms were "agreed in Notion", read the meeting note and
   cross-check every number against it before filling — flag any disagreement to
   Tom rather than silently preferring one source.
2. Run:
   ```bash
   python3 ~/.claude/skills/deal-doc-drafter/scripts/fill_deal_doc.py <safe|term-sheet> \
     --company "<Legal Name, Inc.>" --amount <n> --cap <n> [optional flags]
   ```
3. On `OK: <path>` + `DRIVE: <url>` — report both, plus the key terms you filled
   so Tom can eyeball them, and call out anything rendered `[TBD]`. The script
   auto-uploads the PDF to `Deal Docs/<Company>/<Stage> (<Mon YYYY>)/` per the
   deal-docs layout (idempotent createFolder; `--no-drive` to skip). If a Notion
   Opportunity exists for this deal, add the Drive URL as a chip on its
   `Deal Docs` property (`notion_files_property.py` — see the version section
   below); no Opp yet is fine, the chip lands with the first version event after
   the Opp exists.
4. On `GATE FAIL` — do NOT hand over any artifact. Fix or escalate:
   - `template drift` → the pinned Google Doc no longer carries the expected
     tokens; see Template provenance below.
   - anything else → report the failure verbatim to Tom.

## Open items render as [TBD], highlighted yellow

A field Tom deliberately leaves open (canonically the term sheet's option pool,
pending counsel) renders as `[TBD]` with a `#FFFF00` background, not as a raw
`{{OptionPool}}` token and not as a blank.

**Why:** a raw template token in a doc going to a founder and his counsel reads
as "we forgot to fill this in"; a highlighted `[TBD]` reads as "this is
deliberately open." Tom's call, 2026-08-07 (Fair). The script styles the range
via the Docs API and then **re-reads the doc to confirm every `[TBD]` run
actually carries the highlight** — a gate, not a best-effort.

## Why the Doc templates (do not go back to a raw .docx)

Both templates are Google Docs in `My Drive/Templates`, pinned by file ID:

- SAFE — `1dpSNuy6arL1PhEdTHybohh3ldZ1-SU9XpZZTEAD_tyw`
- Term sheet — `1NOAHgobYW2nHVeQvYRFdEAN0yQLHudtIkFoR3t2Wusw`

Tom's SAFE template is the YC cap-only form with `{{Placeholder}}` tokens
substituted for the fill-ins and the whole investor signature block already
typeset. **This is the canonical source. Never re-derive a SAFE from the raw YC
.docx.**

**Why:** the predecessor skill filled a raw YC `.docx` by replacing the bare
labels `Name:` / `Title:` / `Address:` / `Email:` in the signature block with
`Name: Thomas Seo` etc. Inserting text into a line whose blank is a tab-leader
underscore run pushes the underline out of alignment and overflows the address
onto the next line. The 2026-08-07 Fair SAFE shipped with `Name: Thomas Seo____`,
`Title: Managing Member____`, and `11201` orphaned at the left margin. Tom caught
it. Filling `{{tokens}}` in a Doc whose layout is already correct cannot produce
that class of bug.

The SAFE template still carries one un-tokenized YC bracket,
`[Governing Law Jurisdiction]`; the script fills it from `--governing-law`
(defaulting to the state of incorporation) and the gate rejects any bracketed
placeholder that survives. Origin: AgentBay 2026-07-27, where counsel's exec
copy shipped with it unfilled.

## Every new version — Drive replace + Notion chip swap

Applies to EVERY version event on ANY deal doc for the Opp — a new turn of the
SAFE, **term sheet, or side letter**, a vFinal awaiting execution, the executed
set (file drop, or a Gmail attachment — pull it via the Gmail Attachment Saver
per `shared-references/gmail-attachment-saver.md` if needed). Side-letter
drafting is out of scope below, but its version handling lands here.

1. Find the current file's exact filename: `listFolder` on the round folder via
   the Drive Manager Apps Script, or reuse the name from this conversation.
2. Re-upload under that SAME name — the Apps Script upload action
   trashes-and-replaces on exact-name match, so the prior version is replaced
   in place:
   ```bash
   python3 ~/.claude/skills/deal-doc-drafter/scripts/upload_deal_doc.py <new_version.pdf> \
     --company <Short Name> --stage <Stage> --round-month "<Mon YYYY of the existing round folder>" \
     --name "<exact existing filename>.pdf"
   ```
   Re-running `fill_deal_doc.py --overwrite` does this for you: same filename in,
   same filename up.
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

- Every token replacement asserts its expected occurrence count against the
  Docs API's `occurrencesChanged` (template-drift tripwire).
- Every `[TBD]` must carry the yellow highlight, verified by re-reading the doc.
- Post-render, the output PDF text must contain **zero** `{{Tokens}}` and zero
  `[Bracketed]` placeholders other than `[TBD]`, exactly as many `[TBD]`s as
  were intended, and every required filled value (amount, cap, company, state,
  governing law, expiration, signatory, investor entity).
- Refuses to overwrite an existing PDF at the target path without `--overwrite`
   — a counsel-drafted doc can carry the exact same conventional name
  (AgentBay 2026-07-27 incident).
- The scratch Doc copy is created and deleted by the script seconds apart —
  cleanup of its own artifact, not a Tom-data deletion.

## The two harnesses (Tom, 2026-08-07)

These answer "did anything change that I didn't ask to change?" — one for
content, one for formatting. Neither is optional and neither needs a flag on a
normal run.

**Harness A — nothing changed but the fields we filled.** Every run snapshots
the template's text *before* replacing anything, replays the same substitutions
in Python, and requires the result to equal the filled document
character-for-character. A mismatch prints a unified diff and hard-fails. This
catches a token that also matched somewhere unintended, a stray edit, a template
that drifted between copy and fill, and any Docs-API behaviour that isn't a
literal substitution. Verified 2026-08-07 by injecting a rogue
`Investor`→`Purchaser` replacement: the gate fired and named the exact clauses.

**Harness B — formatting hasn't changed.** Two layers:

1. *Per run:* the filled PDF's page count must equal the template's. A field
   that overflows its line pushes pagination off. Cheap, catches gross overflow.
2. *`--self-test`:* renders three fixed synthetic deals (`safe`,
   `term-sheet-tbd`, `term-sheet-filled` — deliberately wide values so a
   regression has room to show) and diffs each one's `pdftotext -layout` output
   against a committed golden in `tests/golden/`. This is the layer that pins
   what the output is *supposed* to look like, rather than merely that it's
   self-consistent.

```bash
python3 ~/.claude/skills/deal-doc-drafter/scripts/fill_deal_doc.py --self-test
```

**Run `--self-test` after ANY change to a template Doc or to this engine, before
drafting a real doc.** If a template changed on purpose, re-run with `--bless`
and eyeball the diff before accepting it — `--bless` overwrites the goldens, so
a blind bless silently launders a regression into the baseline.

Harness B is the one that would have caught the 2026-08-07 SAFE. The mangled
signature block had the *same page count* as the template, so the per-run check
alone would have missed it; the golden diff surfaces it line by line. Verified
by perturbing the golden to the mangled shape and confirming the diff named the
exact `Name: Thomas Seo____` / orphaned-`11201` lines.

## Template provenance

Both templates are Tom-maintained Google Docs, pinned by file ID above so a
retitle can't silently repoint the script. The SAFE template is YC "Postmoney
Safe - Valuation Cap Only" v1.2 with Inverted's tokens and signature block
applied. If YC ships a new version (or the drift tripwire fires): re-download
from ycombinator.com/documents, word-diff against the current template, show Tom
the diff before he updates the Doc, then re-run.

## Out of scope

- **Side letters** — negotiated per deal; draft from the most recent executed
  side letter in `Deal Docs/` (template:
  `Inverted Capital I - Side Letter Template`, `1XErrVnq98S4BydfOgXopvC2fG8zwh_nplgOlAm6Vwos`)
  as a starting point, in conversation, not here.
- **Discount / MFN SAFE variants, non-US SAFEs** — not templated. Tell Tom and
  stop rather than improvising on the cap-only form.
