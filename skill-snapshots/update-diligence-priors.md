---
name: update-diligence-priors
description: >
  Update priors on a first-pass diligence analysis. Identifies new information (call notes,
  transcripts, materials, backchannel feedback) added since the last analysis, assesses impact
  on existing priors, and prepends a dated update section to the Notion diligence page. Trigger
  on "update first pass", "update priors on [company]", "update diligence on [company]",
  "refresh diligence", "new info on [company] — update the analysis", "incorporate the new call
  notes for [company]", "revise priors on [company]", "I just had another call with [company],
  update the analysis", or any variant requesting a prior update on an existing diligence
  analysis. Always trigger inline — no confirmation needed.
---

# Update Diligence Priors

Incorporate newly available information into an existing first-pass diligence analysis and prepend
a dated update section to the Notion page. This skill is designed to be run multiple times over the
life of an opportunity — each run adds a new update section at the top of the page, creating a
chronological record of how the thesis has evolved as new evidence emerges.

**Each run is strictly incremental.** Only look at materials that have been added to the
opportunity since the most recent prior update (or, on the first run, since the original first-pass
publication date). Do not reprocess items that any earlier update section already cited in its
"New Information Processed" list. The new update block always goes ON TOP of the existing stack of
update sections — newest first.

The update should be opinionated. The point is not to summarize new information neutrally — it is
to assess whether the new evidence strengthens or weakens specific priors from the original
analysis, and to say so clearly.

---

## Step 1: Locate the Existing Diligence Page

Search the Notes database (`collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) for the existing
first-pass diligence page. Use `notion-search` with the query "[Claude] [Company Name] First-Pass
Diligence".

If no existing diligence page is found, tell Tom and suggest running the `first-pass-diligence`
skill first. Do not proceed.

Fetch the full page content with `notion-fetch` — you need the original analysis text to understand
what priors were established.

---

## Step 2: Identify New Information

Fetch the Notion Opportunity page for the company from the Opportunities DB
(`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Compare the current state of the opportunity against
what was already processed by prior runs of this skill.

### Build the "already processed" set first

Before identifying new items, assemble the complete set of materials that have already been
incorporated into the diligence page:

1. **Original Sources section** — every item cited in the original first-pass memo's Sources list.
2. **Every prior Update section's "New Information Processed" list** — walk down each `## Update —
   [date]` section already on the page and collect every bullet (call notes, references, materials,
   research items) it cites.

Union those two sets. Anything in the opportunity's current relations / property fields / page body
that is NOT in that union is fair game for THIS update. Anything already in the union is off-limits
— do not re-cite it, do not reassess it.

### What counts as new information

Fetch every linked item in the opportunity's relation fields and compare against the
"already processed" set assembled above:

- **✍️ Notes relation** — fetch each linked note. Any note not in the "already processed" set
  (Sources + every prior Update's New Information Processed list) is new. These are typically
  call notes, backchannel references, research threads, or Claude analysis pages.
- **Diligence Materials** — check for new Google Drive files, PDFs, or attachments not referenced
  in the existing Sources. Fetch and read any new materials using the type-specific access methods
  documented in the `first-pass-diligence` skill (Step 1c): Google Drive PDFs via `read_pdf_bytes`,
  Google Docs via `google_drive_fetch`, Notion-hosted attachments (`attachment:{UUID}:{filename}`)
  via the Chrome-based pipeline (getSignedFileUrls API → navigate to signed URL → pdf.js text
  extraction), and DocSend links (`docsend.com/view/...`) via the `docsend-to-pdf` skill. If
  Chrome is unavailable for Notion attachments, note them as unreadable and flag the gap.
  - **Inline attachments from Tom's invocation:** If Tom attached files inline when invoking the
    skill (e.g., "update priors and refer to @file.pdf"), check the Diligence Materials property
    FIRST before uploading them. The `materials-handler` webhook typically auto-uploads inline
    Gmail/iMessage attachments to the Drive subfolder and adds them to the property field before
    this skill runs. Re-uploading creates duplicate Drive files (different fileIds) and duplicate
    property-field entries. Match by base filename (e.g., `Preseed plan notes.pdf` vs
    `Factir_Preseed_Plan.pdf` — the former is the auto-add canonical version). Only upload if the
    property field genuinely does not contain it.
- **Page body changes** — the opportunity page body itself may have been updated with new context
  (round details, team changes, status updates).
- **Property changes** — check for changes to Status, Stage, Round Details, or other properties
  that signal deal progression.

For each new item found, fetch its full content. Call notes and transcripts are especially
important — read them carefully for founder signal, new data points, and answers to previously
open questions.

**CRITICAL — `include_transcript: true` is mandatory on every Notes-DB fetch in this step.**
When fetching a linked note via `notion-fetch`, ALWAYS pass `include_transcript: true`. Without
it, meeting notes return only the Notion AI summary block — and that summary compresses
metrics, omits verbal commitments, and silently strips the transcript content this skill needs
to update priors with fidelity. The param is a no-op on non-meeting-note pages (Claude
threads, backchannel notes, research artifacts). No exceptions, no "only if needed" — default
ON. This same rule applies in `first-pass-diligence` Step 1b; we mirror it here so an update
run never operates on thinner data than the original analysis.

If the current conversation contains information Tom has shared directly (e.g., "I just learned
that..." or forwarded content), treat that as new information too.

### If no new information is found

If nothing has changed since the last analysis or update, tell Tom directly: "No new materials,
notes, or changes found for [Company] since [date of last analysis/update]. Nothing to update."
Do not create an empty update section.

---

## Step 3: Assess Impact on Priors

This is the analytical core of the skill. For each piece of new information, evaluate its
impact on the priors established in the original analysis (or the most recent update).

**Product-section refreshes.** When the new evidence touches the §2 Product section of the
existing analysis (new demo recording, product walkthrough, deck v2 with architecture
detail, technical backchannel, founder follow-up on the stack), apply the canonical
framework spec at
`/Users/tomseo/.claude/skills/shared-references/product-build-teardown-framework.md`
and the cost-calibration reference at
`/Users/tomseo/.claude/skills/shared-references/product-build-cost-calibration.md`
to keep refresh outputs aligned with first-pass and standalone-teardown framings.
Single source of truth, no drift across invocation paths — same six lenses (Product
Anatomy, Delivery Mechanism, Build Cost & Time to v1, Path to Production-Grade, Moat
Read, Killshots), same citation rules, same `(per calibration §<N>)` cite format.

Work through these questions for each new data point:

1. **Which specific prior does this affect?** Map new information to the Need to Believe items,
   framework assessments, open questions, or section-level conclusions from the existing analysis.
2. **Does it reinforce or challenge the prior?** Be direct. "This reinforces..." or "This
   challenges..." or "This is orthogonal to the existing analysis but introduces a new
   consideration."
3. **How material is the update?** Not all new information changes the thesis. A second call that
   confirms what the first call established is reinforcing but not thesis-altering. A finding that
   directly contradicts a Need to Believe item is material.
4. **Does this resolve any open questions?** Check the Open Questions from each section of the
   original analysis. If new information answers one, note that explicitly.
5. **Does this introduce new questions?** New information often raises new questions that weren't
   on the radar before.

---

## Step 4: Write the Update Section

### MANDATORY — drafting runs on Opus tier

Per memory `feedback_model_tier_framework`: Opus = no-rubric + final-artifact tasks.
Each Update is appended to the Master Diligence Doc as a final artifact and reads at
the same standard as the original first-pass — Opus territory.

- If interactive Opus session: proceed inline.
- If delegating to a subagent: pass `model: "opus"` to the Agent tool explicitly.
- If stuck in Sonnet: stop, surface to Tom, don't ship a Sonnet update.

Compose the update section. The writing should be analytical and direct — same voice and standards
as the original first-pass diligence. Each update is a self-contained section that a reader can
understand without re-reading the full original analysis, though it references specific sections
and priors from the original.

### Format

```markdown
---

## Update — [Month Day, Year]

### New Information Processed

Group items into categories using bold sub-headers. Standard categories (use only those
that apply — omit any category with zero items):

**Research & Analysis**
- [Pre-mortems, Claude analysis threads, market research, diligence question frameworks]

**References**
- [Backchannel reference calls, formal reference checks]

**Customer Feedback**
- [Customer/prospect feedback notes, NPS data, churn interviews]

**Call Notes**
- [Founder calls, team calls, investor calls]

**Materials**
- [Updated decks, memos, data rooms, financial models]

Order: Research & Analysis → References → Customer Feedback → Call Notes → Materials.

Within each bullet, **bold the label text before the em dash**. For example:
- **Pre-Mortem: Tuor** — March 18, 2026. Structured failure mode analysis...
- **John Cotter (Harness, ex-Arthur AI)** — March 16, 2026. Enterprise sales leader...

### Prior Assessment

[Analytical prose organized by the priors that are affected. Use subsections if multiple
distinct priors are impacted. For each affected prior:]

#### [Prior / Section Reference] — [Reinforced | Challenged | Revised | New Consideration]

[2-3 paragraphs of analytical prose explaining what the new evidence says, how it maps to
the original assessment, and what it means for the investment thesis. Ground every claim
in specific evidence from the new materials.]

[If information-dense, include a supporting table — e.g., comparing original assumptions
to updated data points, before/after metrics, or a revised competitive landscape.]

**Every prior subhead uses `#### Title Case Header`.** Per
`shared-references/label-hierarchy.md`: the prior subhead is always an H4 — never an
inline-bold paragraph leader. Title Case throughout. Drop the trailing period that
paragraph-leader labels usually carry. This applies whether the prior has one child
paragraph or many, and whether it's a multi-child collection (Killshots, Risks) or a
single-prior reinforcement.

**Multi-child priors (Killshots, Risks, multi-signal founder eval changes).** When the
prior being updated is a *collection* that contains 2+ labeled children (introducing
two new killshots, revising three risk subsections), the parent gets the `####` header
as above; each labeled child stays as a `**Label.**` bold-inline paragraph leader.
Example:

```markdown
#### Killshots — Product-Anchored Failure Modes — New Consideration

**Killshot 1 — Foundation-Model Commoditization Cuts the Extraction Differentiation.**
[analytical paragraph — what about the product makes this real…]

***Failure mode:*** [bold-italic paragraph — the precise mechanism…]

**Killshot 2 — Incumbent Customs Broker Ships Landed Cost as a Feature.**
[analytical paragraph…]

***Failure mode:*** [bold-italic paragraph…]
```

### Revised Open Questions

#### Resolved or Sharpened by This Update
- [Any open questions that have been answered, marked as resolved]

#### Newly Raised
- [Any new open questions that emerged from the new information]

### Net Assessment

[One paragraph synthesis: on balance, does the new information make you more or less
convicted in the thesis? Be specific about what moved and what didn't.]

---
```

### Guidance

- **Include tables when the update involves data.** If the new call provided specific metrics
  (loss rates, conversion rates, pipeline numbers), create a table comparing the original
  assumptions or open questions to the new data points. This makes the delta visible at a glance.
- **Reference the original analysis by section.** Say "The original Market section (1.1) noted..."
  or "Need to Believe item 2 (cash flow underwriting) is directly addressed..." so the reader
  can cross-reference.
- **Be explicit about confidence levels.** "This substantially de-risks NTB item 3" is more
  useful than "This is encouraging."
- **Don't repeat the original analysis.** The update section should be additive — new information
  and its implications only. Don't restate what the original analysis already covered.

---

## Step 4.5: LLM Audit Gate — MANDATORY

Per memory `feedback_research_artifact_self_audit`, long-form diligence artifacts get the
research-artifact-audit discipline before delivery. An Update block is squarely in scope —
it's analytical prose anchored on new call notes, references, materials, and feedback, with
inline `[N]` / `^N` citations to those sources.

The audit mechanics (HARD EXIT GATE, un-chunked re-verification, iteration loop, Step 3.5
partial normalization, exit codes, the never-use-the-bypass-alert prohibition) are defined
ONCE in `~/.claude/skills/research-artifact-audit/SKILL.md`. **Read that file in full
before continuing this step** so the A.0 verbatim mandate, A.1 bundle completeness gate,
and B.2.1 un-chunked re-verification all inherit correctly.

**Scope:** the audit fires on JUST the new Update block — NOT the full consolidated doc.
The original first-pass and every prior Update have already been audited by their own runs
(first-pass-diligence Step 4b; this skill's Step 4.5 on prior runs). Re-auditing them is
wasted judge time and burns iteration budget on already-cleared content.

### Update-priors-specific wiring — apply these bindings when following research-artifact-audit/SKILL.md

| Binding (per research-artifact-audit) | Value (update-diligence-priors) |
|---|---|
| `DRAFT` | `/tmp/<company>_update_block_only.md` |
| `SOURCES` | `/tmp/<company>_update_sources.md` |
| `AUDIT_JSON` | `/tmp/<company>_update_audit.json` |
| `JUDGE_PROMPT` | `/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.prompt.md` |
| `AUDIT_RUNNER` | `/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.py` |
| `MAX_ITER` | `3` |
| `WEB_RESEARCH_CAP` | `6` |
| `ITER_SNAPSHOT_PREFIX` | `/tmp/<company>_update_draft.iter` |
| `NORMALIZED_DRAFT` | `/tmp/<company>_update_draft.normalized.md` |

The canonical first-pass judge prompt is appropriate here — no scope exclusions are needed
for the Update block (every section is in scope: New Information Processed is a citation
list, Prior Assessment is analytical prose anchored on cited sources, Revised Open
Questions and Net Assessment reference specific evidence from the new materials).

### Update-priors-specific source bundle structure (Step A in research-artifact-audit)

Write `/tmp/<company>_update_sources.md` with this layout. The A.0 verbatim mandate from
research-artifact-audit applies in full — every section must contain the full verbatim
body of each cited source, NOT pointer manifests or summaries.

```markdown
==== ORIGINAL FIRST-PASS MEMO ====

<full verbatim text of the # First-Pass Diligence — … block as it currently stands on
the Master Diligence Doc page. Needed for cross-reference / prior-anchor verification —
the Update's Prior Assessment cites specific first-pass priors by section name.>

==== PRIOR UPDATE BLOCKS ====

<every prior ## Update — … block on the page in chronological order, separated by ---
delimiters; full verbatim text. Needed for "is this claim already covered" verification —
the Update should not re-cite material already processed by an earlier update.>

==== NEW LINKED NOTES ====

--- Note: <title> (<date>) ---
<full verbatim body of every new call/note cited in this update, including the Notion ID
anchor, the page URL, the full transcript (always fetch with include_transcript: true
for call/meeting notes per Step 2's mandate), the AI-summary block, and structured
fields. NOT a pointer manifest. NOT a summary. The audit judge compares draft claims
against this section LITERALLY.>

==== NEW DILIGENCE MATERIALS ====

--- Material: <name> ---
<full extracted text from every new Drive PDF / Google Doc / DocSend / Notion attachment
cited in this update — pdftotext / google_drive_fetch output, not a pointer or summary.>

==== OPPORTUNITY PAGE ====

<current Notion Opp page properties + full verbatim page body, for round-terms /
status / property-change verification.>
```

### Source bundle completeness check — MANDATORY GATE before audit

The bundle completeness gate is canonical in `research-artifact-audit` Step A.1 — read
that for the full spec. The update-diligence-priors binding is:

```bash
python3 ~/.claude/skills/shared-references/bundle_completeness_check.py \
    --draft  /tmp/<company>_update_block_only.md \
    --bundle /tmp/<company>_update_sources.md \
    --notes-section 'NEW LINKED NOTES' \
    --required-sections 'NEW LINKED NOTES,NEW DILIGENCE MATERIALS,ORIGINAL FIRST-PASS MEMO' \
    --self-page-ids <Master Diligence Doc page ID, 32hex no hyphens>
```

The `--self-page-ids` argument is REQUIRED — Update blocks frequently cite prior Update
sections or the first-pass section via the page's own URL (e.g., footnote `^N` pointing
at `https://notion.so/<page_id>` for "Update #2 on this page" or "the original Market
section"). Without exempting the self-page-id, B2 will false-positive on those
self-references. Look up the Master Diligence Doc page ID from Step 1's notion-search
result.

Exit 0 = proceed to audit. Exit 1 = bundle is missing verbatim content. STOP, rebuild,
re-run. The audit verdict on a broken bundle is structurally meaningless.

### Draft to audit

Build `/tmp/<company>_update_block_only.md` containing JUST the new Update block markdown
(starting from the `---` divider above `## Update — …` and running through the trailing
`---` divider before the prior content). Do NOT include the first-pass or prior updates
in the draft — those are in the source bundle as ground truth.

### Run the audit + iterate

Follow research-artifact-audit Steps B-D in full — invoke the runner, check the HARD EXIT
GATE, run B.2.1 un-chunked re-verification if chunked + untraced > 0, iterate per B.3,
normalize partials per Step C, surface remaining findings per Step D.

The final published Update markdown is `$NORMALIZED_DRAFT` when Step C ran (partials >
0); otherwise `$DRAFT`. Select the source-of-truth file deterministically before Step 5's
prepend so partials Step C just resolved aren't re-introduced:

```bash
if [ -f /tmp/<company>_update_draft.normalized.md ]; then
  FINAL_UPDATE_MD=/tmp/<company>_update_draft.normalized.md
else
  FINAL_UPDATE_MD=/tmp/<company>_update_block_only.md
fi
echo "prepending from: $FINAL_UPDATE_MD"
```

### Audit-result surface in Step 6 Slack alert

Append `⚠️ Audit: <N> untraced after <K> iterations, <M> partials normalized` as a
fourth line of the Step 6 Signal alert when there are residual untraced findings OR any
partials were normalized. If the audit ends with 0 untraced and 0 partial cleanly, no
`⚠️` line — the alert stays at the standard three lines. The substance (residual untraced
claims with judge notes; normalized partials as before→after diffs) is required by
research-artifact-audit Step D; this only specifies the Slack format.

---

## Step 5: Prepend to the Notion Page

### Tooling choice — MCP vs REST API

Two transports are available; use them per their strengths:

- **MCP `notion-update-page`** — required for the initial markdown→blocks conversion
  (it knows how to render `<table>` markup, embeds, callouts, etc. into Notion blocks).
  Use it for the *initial prepend* of the new update content because that content can
  contain tables.
- **REST API direct** (PATCH/DELETE via `urllib`) — faster and more reliable for
  pure-text operations: title updates, property writes, deleting existing blocks,
  patching a single block's `rich_text`, listing children. Use it for everything in
  this step EXCEPT the new-content prepend.

Why: MCP `update_content` and `insert_content` both time out client-side on
multi-KB payloads (observed repeatedly in production — including the Factir
2026-05-21 consolidation run where two `update_content` calls timed out before
a third `insert_content` finally completed server-side). The REST API is
single-shot 200s on the same operations. Caught after Factir 2026-05-21 —
prior runs that relied on MCP for everything would often appear stuck.

REST API token resolution (mirrors `~/.claude/scripts/notion_files_property.py`):

```python
import os, json, subprocess
from urllib.request import Request, urlopen

env = {**os.environ, 'SOPS_AGE_KEY_FILE': os.path.expanduser('~/.config/sops/age/keys.txt')}
token = subprocess.check_output(
    ['sops', '-d', os.path.expanduser('~/code/notion-backup/.notion-token.enc.txt')],
    env=env,
).decode().strip()
HDR = {'Authorization': f'Bearer {token}',
       'Notion-Version': '2025-09-03',
       'Content-Type': 'application/json'}
```

### 5.1 Prepend the new update content (MCP)

Use `notion-update-page` with the `insert_content` command and `position: {"type":"start"}`
to insert the new update section at the top of the page. The content goes immediately
after the page title and above the first existing section (typically the previous update's
leading `---` or the "Framework Mapping" header on a never-updated page).

The update sections should stack chronologically — newest at the top, oldest at the bottom,
with the original analysis below all updates. This means a page that has been updated three times
will read: Update 3 → Update 2 → Update 1 → Original Analysis.

When prepending the first update to a page that has never been updated before, also insert a
section divider and header before the original first-pass content:

```markdown
---
# First-Pass Diligence — [Original Publication Date]
---
```

This goes immediately before `## Framework Mapping — Inverted Lens` (or whatever the first H2
of the original analysis is). On subsequent updates, this header already exists and should not
be duplicated.

**Legacy `# Original First-Pass Memo — …` anchor.** Pages created before the 2026-05-22
rename use `# Original First-Pass Memo — …` instead of `# First-Pass Diligence — …`. When
operating on such a page (no `# First-Pass Diligence — …` anchor present but
`# Original First-Pass Memo — …` is), rename the existing anchor in place as part of this
prepend step (PATCH the `heading_1` block's rich_text) so all output going forward uses the
canonical `# First-Pass Diligence — …` form. The PDF page-break trigger handles both for
backward compat, but the Notion page should converge.

**MCP timeout handling — MANDATORY.** MCP `insert_content` frequently returns
`notionhq_client_request_timeout` on payloads > ~10KB even when the write
succeeds server-side. Do NOT retry the same MCP call after a timeout; instead:

1. Wait ~5–10 seconds for Notion to propagate.
2. Verify via REST API by listing the page's first few children:
   ```python
   resp = json.loads(urlopen(Request(
       f'https://api.notion.com/v1/blocks/{PAGE_ID}/children?page_size=5',
       headers=HDR)).read())
   first_h2 = next((b for b in resp['results']
                    if b.get('type') == 'heading_2'), None)
   ```
   If the first H2's text matches the new update header, the write succeeded.
3. Only retry MCP if verification confirms the content is NOT there.

### 5.2 Consolidating / replacing an existing update (REST API)

When this run *replaces* an already-published update (rather than adding a fresh
one) — e.g., the user asks you to roll an earlier same-day update into the new
one, or to redo a flawed published update — use the REST API to delete the old
update's blocks after the new prepend has landed. MCP `update_content` with a
large `old_str` matching the entire old block will time out without applying.

```python
# 1. List all top-level blocks (paginated, page_size=100)
all_blocks = []
cursor = None
while True:
    url = f'https://api.notion.com/v1/blocks/{PAGE_ID}/children?page_size=100'
    if cursor: url += f'&start_cursor={cursor}'
    r = json.loads(urlopen(Request(url, headers=HDR)).read())
    all_blocks.extend(r['results'])
    if not r.get('has_more'): break
    cursor = r['next_cursor']

# 2. Identify the OLD update's block range by walking from the heading_2
#    matching "Update — <old date>" forward until the next "Update —",
#    "First-Pass Diligence —", or (legacy) "Original First-Pass Memo —"
#    header. Include any trailing dividers that bracket the block.

# 3. DELETE each block. Notion's per-block DELETE endpoint:
from urllib.request import Request
import time
for b in to_delete:
    urlopen(Request(f'https://api.notion.com/v1/blocks/{b["id"]}',
                    headers=HDR, method='DELETE'))
    time.sleep(0.15)  # be gentle on rate limit
```

### 5.3 Update the page title (REST API)

After prepending the update, update the page title to reflect the most recent update date.
The title should always reflect the most recent update — do not stack multiple date
suffixes. Strip the old date portion entirely and replace with the new one.

New title format: `[Claude] [Company Name] Master Diligence Doc — MM.DD.YYYY Update`

```python
new_title = '[Claude] <Company> Master Diligence Doc — <MM.DD.YYYY> Update'
payload = {'properties': {'Name': {'title': [
    {'type': 'text', 'text': {'content': new_title}}]}}}
urlopen(Request(f'https://api.notion.com/v1/pages/{PAGE_ID}',
                headers=HDR, method='PATCH',
                data=json.dumps(payload).encode()))
```

---

## Step 5a: Pre-PDF Lint — MANDATORY GATE

After the Notion prepend in Step 5 but BEFORE PDF build in Step 5b, run the
deterministic shape-rule lint over the FULL updated markdown (new update
prepended to original). This catches structural failures — malformed update
headers, missing "New Information Processed" sub-sections, empty NIP lists,
missing `---` page-break sentinel before the inner First-Pass Diligence anchor — that
would otherwise leak into the PDF and the Slack alert.

Write the rebuilt full markdown to a tmp path (e.g.,
`/tmp/<company>_updated_full.md`) and run:

```bash
python3 /Users/tomseo/.claude/skills/update-diligence-priors/update_priors_lint.py \
    --draft /tmp/<company>_updated_full.md
```

**Hard gate.** Exit code 0 = proceed to 5b. Exit code 1 = STOP. For each
finding, fix the underlying issue (typo in update header, forgot the bolded
NIP bullets, missing `---` sentinel) and re-run the lint. Do NOT proceed past
a failing lint silently; the rules it catches all manifest in the published
artifact.

The lint covers (first wave shipped 2026-05-15):
- **U1** — update header well-formed: `## Update — Month DD, YYYY` (em dash,
  spelled-out month). Catches numeric dates (`05.15.2026`), wrong dash
  (`-` vs `—`), missing year/comma/space.
- **U2** — each `## Update — ...` block contains `### New Information Processed`.
- **U3** — each NIP section has ≥1 bolded bullet (`- **<label>** — <description>`).
- **U4** — `---` page-break sentinel directly above `# First-Pass Diligence —` (new) or
  `# Original First-Pass Memo —` (legacy pages not yet re-rendered).

See `update_priors_lint.py --self-test` for the full fixture set.

---

## Step 5b: Build the Updated PDF

Generate a new PDF containing the full updated analysis — the new update section(s) prepended
to the complete original analysis. This is a standalone document, separate from the original
first-pass PDF. Do NOT overwrite or modify the original.

### Filename convention

```
[Company]_Master_Diligence_MM.DD.YYYY_v[N].pdf
```

Where `MM.DD.YYYY` is today's date (the update date) and `N` is the update number
(2 for Update #2, 3 for Update #3, etc.). The original first-pass date is dropped
entirely from the filename — the update date is the only date in the name. Example:
`Tuor_Master_Diligence_04.02.2026_v2.pdf`

**Retention rule — keep only the latest version.** Each new update's PDF
*contains* all prior updates plus the original first-pass (it's a full
consolidated snapshot). Once the new `_v[N].pdf` is uploaded and the Notion
links are swapped, **trash every prior diligence-snapshot PDF for this company
in the same Drive subfolder** (any file matching
`[Company]_Master_Diligence_*.pdf` OR the legacy
`[Company]_First_Pass_Diligence_*.pdf` patterns, except the new one). Use rclone:

```bash
# List the company subfolder
rclone lsf "gdrive:Diligence/[Company]"
# Trash old snapshots (any _v<N-1>.pdf, _Update.pdf, _v5.pdf, legacy
# _First_Pass_Diligence_*.pdf, etc.)
rclone deletefile "gdrive:Diligence/[Company]/<OLD_NAME>.pdf" --drive-use-trash
```

Do **NOT** trash the source materials in the same folder (founder memo, deck,
plan, ACV build, founder positioning notes, Q&A docs, etc.) — only the prior
diligence-snapshot PDFs that this skill itself wrote.

Then scrub stale Drive URLs from BOTH:
- The Opp's `Diligence Materials` files-property (drop entries whose URL is no
  longer in Drive)
- The Opp page body's `## 📎 Diligence Materials` bulleted list (replace the
  whole list with the single latest `_v[N].pdf` link)

The latest snapshot is canonically the only diligence PDF that should be linked
anywhere. Caught after Factir 2026-05-20 Update #4: prior `_v5`/`_Update1`/
`_Update2`/`_Update.pdf` accumulated across runs and confused the reader.

### Content

The PDF should contain:
1. **The full update section(s)** — all updates prepended since the original, newest first,
   rendered exactly as they appear on the Notion page.
2. **Page breaks between EVERY major section.** Insert a `PageBreak()` flowable before:
   - **Every `## Update — ...` section after the first** (so Update #2 starts on its own
     page, Update #3 starts on its own page, etc. — the very first / topmost Update is
     the natural top of the doc, no break needed there).
   - **The `# First-Pass Diligence — ...` heading** (always — the inner first-pass content
     starts on a fresh page after all the prepended updates). Legacy
     `# Original First-Pass Memo — ...` anchor also triggers the page break for backward
     compat with pages not yet re-rendered.
   The canonical PDF templates at `~/.claude/skills/shared-references/pdf_builder_template.py`
   and `pdf_builder_from_md_template.py` implement this automatically in `parse_content()`
   — just use them. See `long-form-pdf-spec.md` §"Page Breaks Between Sections" for details.
3. **An H1 header** — `First-Pass Diligence — [Original Publication Date]` rendered as
   an underlined H1, marking the start of the original analysis.
4. **The full original first-pass analysis** — everything from Framework Mapping through Sources,
   rendered in the same format as the original PDF.

The document is a complete, self-contained artifact. A reader should be able to understand the
full diligence picture from the PDF alone without referencing the Notion page.

### PDF header

- **Title (14pt bold):** `[Company Name] — Master Diligence Doc`
- **Subtitle (9pt italic, dark gray):** `Latest Update: [Update Date] | Initially Published: [Original First-Pass Date] | Notion`
  (where "Notion" is a clickable hyperlink to the Notion page URL). Both dates should be present
  so the reader knows when the original analysis was written and when the most recent update was applied.

### Formatting

Follow the canonical long-form PDF format spec. Install reportlab if needed:
`pip install reportlab --break-system-packages -q`

Read the formatting spec at `/Users/tomseo/.claude/skills/shared-references/long-form-pdf-spec.md`
for exact style definitions. Apply the same post-table/post-chart `Spacer(1, 10)` spacing rule.

**Page numbers:** Add centered page numbers (8pt Helvetica) at the bottom of every page using a
`canvas` callback on `SimpleDocTemplate`:
```python
def page_number(canvas, doc):
    canvas.saveState()
    canvas.setFont("Helvetica", 8)
    canvas.drawCentredString(PAGE_W / 2, 0.4 * inch, f"{doc.page}")
    canvas.restoreState()
```
Pass `onFirstPage=page_number, onLaterPages=page_number` to `doc.build()`.

**Markdown-to-reportlab conversion:** All prose text must pass through a `clean_text()` function
that converts markdown formatting to reportlab XML before being wrapped in `Paragraph()`:
- `***text***` → `<b><i>text</i></b>`
- `**text**` → `<b>text</b>`
- `*text*` → `<i>text</i>`
- `[text](url)` → `<a href="url" color="black"><u>text</u></a>`
This ensures no stray asterisks appear in the rendered PDF.

Save the PDF to:
`/sessions/loving-modest-fermat/Users/tomseo/Downloads/[Company]_Master_Diligence_MM.DD.YYYY_v[N].pdf`

### Upload and link in Notion

Upload the updated PDF to the same company subfolder in Google Drive used by the original
first-pass PDF (under Diligence root `1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`). Use the Apps
Script endpoint — never Zapier. Read the full reference at
`/Users/tomseo/.claude/skills/shared-references/drive-upload.md`.

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"
DILIGENCE_ROOT = "1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d"

# createFolder is idempotent — returns existing folder if it already exists
folder_resp = requests.post(DRIVE_URL, json={
    "action": "createFolder",
    "folderName": "<COMPANY_NAME>",
    "parentFolderId": DILIGENCE_ROOT
}, allow_redirects=True, timeout=60)
subfolder_id = folder_resp.json()["folderId"]

with open(pdf_path, 'rb') as f:
    pdf_b64 = base64.b64encode(f.read()).decode('utf-8')

upload_resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": "[Company]_Master_Diligence_MM.DD.YYYY_v[N].pdf",
    "fileBase64": pdf_b64,
    "mimeType": "application/pdf",
    "folderId": subfolder_id
}, allow_redirects=True, timeout=120)
file_url = upload_resp.json()["url"]
```

If Tom attached supplementary materials (decks, plans, models) inline with the skill invocation,
do NOT re-upload them — they are almost always already in the Diligence Materials property field
courtesy of the `materials-handler` webhook that fires on inbound email/iMessage attachments.
Reference the existing Drive URLs from the property field in the update section's "New Information
Processed" list. The only file this skill should upload is the regenerated update PDF itself.

After upload, link the updated PDF in two places on the Notion opportunity page.
**Both writes go through the REST API**, not MCP — they are pure-text block/property
patches and MCP is slower + less reliable on them.

1. **Page body** — the `## 📎 Diligence Materials` section already contains a
   bulleted link to the prior `_v[N-1].pdf` snapshot. PATCH the existing bullet's
   `rich_text` in place instead of appending a new bullet (per the retention rule
   above — only one diligence-snapshot link should be linked anywhere). REST API:
   ```python
   # Find the bullet block by listing the Opp page children and matching
   # "Master_Diligence" (or legacy "First_Pass_Diligence") in its rich_text.
   # Then PATCH:
   patch = {'bulleted_list_item': {'rich_text': [
       {'type':'text',
        'text':{'content': '<v[N] filename>', 'link':{'url': '<drive_url>'}},
        'annotations':{'bold': True}},
       {'type':'text',
        'text':{'content': ' \u2014 Latest Claude diligence snapshot through Update #[N] (consolidates all prior updates + original first-pass)'}},
   ]}}
   urlopen(Request(f'https://api.notion.com/v1/blocks/{BULLET_ID}',
                   headers=HDR, method='PATCH',
                   data=json.dumps(patch).encode()))
   ```
   If there is no existing bullet (first-ever update PDF), instead use
   `POST /v1/blocks/{OPP_PAGE_ID}/children` with `after: <heading_2 id>` to
   insert one beneath the `## 📎 Diligence Materials` header.

2. **Diligence Materials Files property field** — follow the shared reference at
   `/Users/tomseo/.claude/skills/shared-references/add-link-to-diligence-materials.md`. Pass the opportunity
   page ID, the Drive file URL, and display name
   `[Company]_Master_Diligence_MM.DD.YYYY_v[N].pdf`.

   **MANDATORY verification — never trust the 200 response alone.** Immediately after the property write, re-fetch the Opportunity page and confirm an entry in the `Diligence Materials` files array has `external.url` matching the Drive URL you just wrote. If absent, the write silently failed (observed Factir 2026-05-15 — PATCH returned 200 but Notion kept the stale URL underneath the new display label). Re-PATCH the full files array explicitly, then re-verify. After 3 retries, surface to Tom rather than publish silently. Reference snippet:

   ```python
   opp = json.loads(urlopen(Request(f"https://api.notion.com/v1/pages/{OPP_ID}", headers=HDR)).read())
   urls = {f.get("external",{}).get("url") for f in opp["properties"]["Diligence Materials"]["files"]}
   assert drive_url in urls, f"Property write did NOT take — URL {drive_url} not in {urls}"
   ```

   If Chrome is unavailable, skip the property field and rely on the page body link.

Act autonomously — do not ask for permission. Report what was done in the summary.

---

## Step 6: Send Signal Alert

Read the `send-alert` skill (`**/send-alert/SKILL.md`) for the chatID and guardrails.

Use `send_message` (Beeper MCP) with the chatID from the send-alert skill. Compose the body:

```
🔄 [Company Name] — Diligence Priors Updated

New information processed:
- [Source 1 one-liner]
- [Source 2 one-liner]

Net assessment: [One sentence — more convicted, less convicted, or thesis unchanged]

Notion: [Notion page URL]
PDF: [Google Drive URL to updated PDF]
```

---

## Quality Bar

A good prior update:

- Maps every piece of new information to a specific prior, open question, or section from the
  original analysis
- Is direct about whether evidence reinforces or challenges the thesis — no hedging for comfort
- Uses tables when the new information includes quantitative data or metrics
- Resolves open questions explicitly rather than leaving them hanging
- Introduces new questions that the new information raises
- Ends with a clear net assessment that a reader can act on

A bad prior update just summarizes the new call notes or materials without connecting them back
to the original framework. If you find yourself writing "the founder discussed X" without then
saying "which [reinforces/challenges] the prior that Y," stop and add the analytical layer.
