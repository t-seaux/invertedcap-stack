---
name: diligence-qa
description: Draft a forward-looking Diligence Q&A Google Doc for a pipeline opportunity. Reads every diligence material, call note (with transcript), and prior Claude artifact for the named Opportunity, segregates what's known from what's open across four buckets (Product / Distribution / Market / Team), then produces a publishable Google Doc — copied from a canonical template carrying Tom's visual chrome (logo, justified body, header hierarchy, bullet spacing) and populated via Docs API batchUpdate. Lands in `Diligence/[Company]/` (per-Opp subfolder created idempotently). Links back into the Opp's `Diligence Materials` field and posts a Slack alert. Trigger when Tom says "create a diligence Q&A for [X]", "draft Q&A on [X]", "diligence questions for [X]", "draft diligence Q&A for [X]", "Q&A doc for [X]", or any variant requesting a forward-looking question artifact for a named pipeline company. Manual-only — no scheduled or webhook entry point.
---

# Diligence Q&A

Draft the initial Google Doc Diligence Q&A for a pipeline opportunity. The
artifact is a forward-looking question set — what Tom needs to dig into next,
not a synthesis of what's already known. Reading the materials produces the
question set; the question set is the publishable artifact.

Architectural cousin: `draft-investment-memo` — same Drive-template-copy +
Docs API pipeline, same auth path, same Notion linkage shape. Key
differences:
- **No `research-artifact-audit` gate.** Questions are not factual claims;
  the audit's claim-tracing apparatus doesn't apply. Format correctness is
  enforced by `format_guard.py` instead.
- **Per-Opp Drive subfolder**, not a shared parent folder.
- **One-time template scaffold** has to run before the first per-Opp run
  (see Step T).

Sibling files in this skill:
- `canonical_spec.py` — typed accessor over the **measured reference profile**.
  Formatting values (font sizes, bold flags, indents, spacing, named styles,
  alignment) and the doc IDs are NOT hand-typed here — they are derived from
  `~/.claude/templates/diligence-qa/reference_profile.json` +
  `reference.json`, which `~/.claude/templates/measure_reference.py` produces
  from Tom's live reference doc. See "Formatting source of truth" below.
- `build_qa_doc.py` — deterministic builder: copies the logo-only template,
  inserts the body as plain text once, then styles every paragraph by computed
  character offset. No markdown markers, no `replaceAllText`, no delete passes —
  nothing shifts indices, so nothing can inherit a stray bullet.
- `format_guard.py` — deterministic post-build verification (G1-G9)
- `scaffold_template.py` — one-time `[TEMPLATE] Diligence Q&A` scaffold
- `QA_CONTENT_SCHEMA.md` — typed shape of the question dict the drafter emits

**Why the builder, not `replaceAllText` + a formatting harness:** the prior
pipeline inserted markdown into a bulleted placeholder and fixed it up with
delete passes. The deletes shifted indices and every inserted line inherited a
bullet — the doc rendered with a bullet on every paragraph. The builder is
index-shift free and the guard's G7 explicitly forbids bullets on non-question
paragraphs.

---

## Formatting source of truth (the templates library)

The repeated formatting iteration had ONE root cause: the guard validated
build output against hand-typed constants, so a transcription error made the
guard's *expectation* wrong in the same way and the bad output passed green.
That can no longer happen. The formatting canon is now MEASURED, not typed:

- `~/.claude/templates/diligence-qa/reference.json` — doc IDs (reference doc,
  logo-only template, Drive root). The single place to update an ID.
- `~/.claude/templates/diligence-qa/reference_profile.json` — per-paragraph-role
  style profile (namedStyleType, bold, italic, fontSize, bullet, alignment,
  indent, spacing) measured from the live reference doc.
- `~/.claude/templates/measure_reference.py` — produces the profile.
- `~/.claude/templates/golden_test.py` — diffs a built doc's measured profile
  against the reference profile (catches a builder that applies a value wrong).

`canonical_spec.py` loads both files and derives every formatting number from
them, so builder AND guard trace each number to the doc Tom approved. To
**regenerate the profile** (deliberate — only after Tom re-approves a changed
reference doc):

```bash
python3 ~/.claude/templates/measure_reference.py \
    --doc-id <reference_doc_id> \
    --out ~/.claude/templates/diligence-qa/reference_profile.json
```

---

## Step T: One-time template scaffold (DO ONCE, BEFORE FIRST RUN)

Before Step 0 can succeed, the `[TEMPLATE] Diligence Q&A` Doc must exist in
the Diligence root folder (`canonical_spec.DILIGENCE_ROOT_FOLDER_ID`).

```bash
python3 ~/.claude/skills/diligence-qa/scaffold_template.py
```

The script copies Tom's reference doc (`canonical_spec.REFERENCE_DOC_ID`),
wipes its body, and writes a placeholder skeleton. It prints the new Drive
ID — record it as `template_doc_id` in
`~/.claude/templates/diligence-qa/reference.json` (NOT in canonical_spec —
that constant is now derived from the manifest). Until `template_doc_id` is
set, the per-run steps below will refuse.

If the visual canon evolves (new logo, different header weights, additional
section), update `reference_doc_id` in the manifest to point at the new
canonical doc, re-run `measure_reference.py` to refresh the profile, trash the
prior `[TEMPLATE]`, re-run the scaffold, and record the new `template_doc_id`.

---

## Step 0: Idempotency check — refuse if a Q&A already exists

Before any other work, check whether a Diligence Q&A doc already exists for
this Opportunity. Two-stage lookup:

1. Resolve the per-Opp Drive subfolder ID by calling the Apps Script
   `createFolder` action (idempotent — returns existing folder if present).
2. `mcp__claude_ai_Google_Drive__search_files` with query
   `"<Company> - Diligence Q&A" parent:<opp_subfolder_id>`.

If found, STOP. Reply to Tom with:

> A Q&A doc already exists at `<Drive URL>`. This skill only drafts initial
> Q&As. Future edits to the existing doc will be handled by a dedicated
> `update-diligence-qa` skill (not yet built). Want me to scaffold that
> instead, or open the existing doc?

Do NOT auto-version (`v2`), do NOT create a side-by-side draft, do NOT proceed.

Also surface and refuse if `canonical_spec.TEMPLATE_ID is None` — direct Tom
to Step T.

---

## Step 1: Resolve the Opportunity & gather sources

### 1a. Resolve the Opp

`notion-search` the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`)
for the company name. If more than one row matches, surface candidates and
stop. Never auto-pick.

**FO-suffix gate** — if the Opp name matches `/\([^)]*\bFO\b[^)]*\)/i`, STOP
and route Tom to the underlying non-FO Opp. Per memory
`feedback_first_pass_fo_gate_in_webhook` — diligence artifacts are written
against the original Opp, not the follow-on row.

### 1b. Fetch every linked Note WITH transcripts

For each page in the `✍️ Notes` relation, `notion-fetch` with
`include_transcript: true` (mandatory — per memory
`feedback_summarize_call_use_full_transcript`). Categorize each note in
working memory:

- First-pass diligence note (if present)
- Pre-mortem (if present)
- Founder meetings / call notes
- Backchannel / expert feedback
- Other Claude analysis (market research, build teardown, etc.)

A first-pass diligence note is **NOT required** for Q&A — Q&A can run before
first-pass. Note its presence/absence in Step 2 grounding.

### 1c. Fetch every Diligence Material

Iterate `Diligence Materials` using the same access-method matrix as
`draft-investment-memo` Step 1c (Google Docs → MCP `read_file_content`;
Drive PDFs → `read_pdf_bytes`; Notion-hosted → Chrome pipeline; DocSend →
`docsend-to-pdf`; video → Whisper / Loom transcript path).

---

## Step 2: First-pass "known vs. open" pass

Read every source in full. For each of the four buckets — **Product**,
**Distribution**, **Market**, **Team** — build a working two-column table:

- **Known** — what the materials answer cleanly. Cite the source in a
  scratch note (this is grounding for Step 4, not output).
- **Open** — what's missing, ambiguous, contradicted, or only partially
  addressed. These are candidate questions.

The Open column for each bucket becomes a flat list of candidate questions —
there are NO subsections in the Factir canon. Each question gets a short
bold lead-in label at draft time (Step 4).

---

## Step 3: Read the voice guide

Read `~/.claude/skills/writing-style/letters-and-memos/STYLE.md` in full.
Apply Tom's declarative, no-hedge register to question phrasing. Patterns to
preserve:

- Plain interrogatives. No preamble ("I'd love to understand…").
- No hedge stems ("could you maybe share…").
- Question is *one sentence*. If you need two, you're asking two questions.
- Specific over generic. "How does the model's retention curve diverge for
  customers acquired via inbound vs. partnership channels?" > "Tell me about
  retention."

---

## Step 4: Draft the question dict

Emit JSON matching `QA_CONTENT_SCHEMA.md` — a flat list of `[label, question]`
pairs per section, no subsections:

```json
{
  "product":      [["The Wedge", "Which single workload do you lead every new clinic with, and why that one?"], ...],
  "distribution": [["ICP Definition", "How precisely do you define the super-elastic small clinic?"], ...],
  "market":       [...],
  "team":         [...],
  "other":        [...]
}
```

The drafter MUST honor:

- **One question per list entry.** No multi-question paragraphs.
- **Short bold lead-in label** (1–3 words) per pair. Do NOT bold it or add a
  trailing period — the builder runs `normalize_label()` and bolds it.
- **Coverage minimum: 3 questions per mandatory section**
  (`COVERAGE_MINIMUM_QUESTIONS`). Below that, find more open threads in Step 2
  or drop the section entirely.
- **All four sections render** — Product / Distribution / Market / Team are
  mandatory inquiry surfaces per Tom's spec.
- **Optional `other` catch-all** — questions that don't fit the four-bucket
  framework go under the optional `other` key (renders as "Other", always
  last). EXEMPT from the 3-question minimum; a single stray question is fine.
  Omit the key when there are none.

Save the draft to `/tmp/qa_factir.[Company].json` (the builder reads this path
via `--content`).

---

## Step 5: Resolve subfolder, copy template, build the body

### 5a. Get-or-create the per-Opp Diligence subfolder

```python
import requests
import canonical_spec as spec

resp = requests.post(spec.DRIVE_APPS_SCRIPT_URL, json={
    "action":         "createFolder",
    "folderName":     company_name,           # MUST be the Notion Opp name verbatim
    "parentFolderId": spec.DILIGENCE_ROOT_FOLDER_ID,
}, allow_redirects=True, timeout=60).json()
opp_subfolder_id = resp["folderId"]
```

The Apps Script's `createFolder` is idempotent — if the folder already
exists, it returns the existing one (`alreadyExisted: true`).

### 5b. Run the builder

`build_qa_doc.py` copies the logo-only template into the subfolder, inserts the
whole body as one plain-text string, then styles every paragraph by computed
character offset — date (bold), title (HEADING_1 bold), section headers
(HEADING_2 italic), and each question (disc bullet, JUSTIFIED, 18pt indent,
bold lead-in label). No `replaceAllText`, no markdown markers, no delete
passes. It prints `doc_id` then `doc_url`.

```bash
python3 ~/.claude/skills/diligence-qa/build_qa_doc.py \
    --company "$COMPANY" \
    --subfolder "$OPP_SUBFOLDER_ID" \
    --content "/tmp/qa_factir.$COMPANY.json"
```

The date and title are generated inside the builder — there is NO lede, NO
placeholders, and NO subsection headers. Do not hand-apply any `batchUpdate`
styling in this SKILL.md; all rules live in `build_qa_doc.py` driven by
`canonical_spec.py`.

To rebuild into an already-copied doc (e.g. after a guard failure), pass
`--doc-id "$DOC_ID"` instead of `--subfolder`.

---

## Step 6: Format guard (HARD GATE)

```bash
python3 ~/.claude/skills/diligence-qa/format_guard.py \
    --doc-id "$DOC_ID" --company "$COMPANY"
```

Re-fetches the live Doc and enforces nine checks:

| ID | Check |
|---|---|
| G1 | No `{{PLACEHOLDER}}` tokens survive |
| G2 | Logo present (leading inline image) |
| G3 | Date line: bold, NOT bulleted |
| G4 | Title is HEADING_1, bold, NOT bulleted, == `TITLE_FORMAT.format(company=…)` |
| G5 | All 4 sections present as HEADING_2, italic, NOT bulleted, in order |
| G6 | Every question paragraph: bulleted, JUSTIFIED, indent≈18pt, bold lead-in |
| G7 | No bullet glyph on any non-question paragraph (date/title/section/blank) |
| G8 | No raw markdown markers (`#`, `##`, `- `, `**`) leaked into rendered text |
| G9 | Each section has ≥ `COVERAGE_MINIMUM_QUESTIONS` question bullets |

**G7 is the load-bearing check** — it catches the historical defect where every
paragraph rendered with a bullet. A green guard with a bulleted date/title/
section means G7 regressed; investigate before publishing.

**Exit 0 = proceed to Step 7.** Exit 1 = at least one check failed; STOP and
surface failures to Tom. Do NOT proceed to Notion linkage or Slack alert
with a non-zero guard exit — the artifact is not publishable.

Common failure modes + fixes:
- `G1: placeholder survived` → drafter omitted a section, or the template
  carried a stray token. Fix the content JSON and re-run the builder.
- `G6: question align …` / `indent …` → re-run `build_qa_doc.py --doc-id`.
- `G7: non-question paragraph is bulleted` → template regressed to a bulleted
  trailing paragraph; re-run `scaffold_template.py` (Step T) and rebuild.
- `G9: section has N questions` → drafter undershot the coverage minimum.
  Re-run Step 4 with more questions, or drop the section.

The guard re-reads the live doc rather than trusting the builder's print
output — per `feedback_skill_self_report_diverges_from_actual_write`. A green
guard is necessary but not sufficient: also dump the paragraph styles once and
eyeball them against the Factir reference before reporting done.

---

## Step 7: Link the Q&A back into the Opp

Append the Drive URL to the Opp's `Diligence Materials` property via the
shared helper:

```bash
python3 ~/.claude/scripts/notion_files_property.py \
    --page-id "$OPP_PAGE_ID" \
    --prop "Diligence Materials" \
    --label "$COMPANY - Diligence Q&A" \
    --url "$DOC_URL"
```

The helper flags are `--page-id`, `--prop`, `--label`, `--url` — there is no
`--action`. The write is idempotent by URL and by canonical label.

Per memory `reference_notion_files_property_helper` — that helper is the
canonical path for Files-property writes. Do NOT use `notion-update-page`
with an inline Files schema.

Do NOT create a separate Notes-DB entry. The Drive doc is the canonical
artifact; Notes DB is reserved for Notion-native Claude artifacts (first-pass,
pre-mortem). Mirrors `draft-investment-memo` Step 7.

---

## Step 8: Slack alert via send-alert

Read `~/.claude/skills/send-alert/SKILL.md` and post to `#claude-alerts`:

- Company + Drive URL (GFM link — `[text](url)`, never Slack mrkdwn `<url|text>`,
  per memory `feedback_send_alert_gfm_not_mrkdwn`)
- Per-section question count: `Product: N / Distribution: N / Market: N / Team: N`
- Format guard summary: `format guard: G1-G9 all PASS`

---

## Behavior rules

### Never ask permission inside this flow

End-to-end Tom-initiated. Per memories
`feedback_first_pass_no_permission_prompts` and
`feedback_no_permission_for_user_initiated_analysis`, every prescribed write
executes autonomously: Drive folder create, builder run (Drive copy + Docs API
batchUpdate), guard run, Notion file-property write, Slack alert.

Stops are limited to:
- Step T not yet run (TEMPLATE_ID is None)
- Step 0 idempotency (Q&A doc exists)
- Step 1a ambiguous Opp resolution
- Step 1a FO-suffix gate
- Step 6 format guard fail (exit 1)

### Never auto-version

Step 0 refuses if a Q&A doc exists. No `v2`, no side-by-side, no append.

### No trailing offers

Per memory `feedback_no_trailing_permission_offers`, after Step 9 report and
stop. Do not offer to chain into `draft-investment-memo` or any other skill.

### Confirm before deletes

Per memory `feedback_always_confirm_before_delete`. The only delete-shaped
op in this skill is template re-scaffolding (Step T), which is manual and
out-of-flow. No deletes inside the per-Opp run.
