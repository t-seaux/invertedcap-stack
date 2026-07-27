---
name: finalize-diligence
description: >
  Synthesize the end-to-end diligence record into a standalone Final Assessment artifact at
  the top of the Master Diligence Doc. Reads the original first-pass plus every appended
  update plus all linked source materials, then writes Overview + Thesis (with NTBs) +
  Diligence Journey + Standing Open Questions + Footnotes. Never includes a
  Decision/Disposition block — the invest/pass call stays with Tom and is reflected via the
  Opp Status. Re-running replaces the prior Final Assessment in place (block delete +
  re-prepend). Rebuilds the consolidated PDF as
  `[Company]_Master_Diligence_MM.DD.YYYY_vFinal.pdf`. Trigger on "finalize diligence on [company]",
  "final assessment on [company]", "make the final call on [company]", "wrap diligence on
  [company]", "publish the final memo on [company]", "close out diligence on [company]", or any
  variant requesting the closing synthesis on an existing first-pass page. Always trigger inline
  — no confirmation needed.
---

# Finalize Diligence

Produce the closing synthesis on a first-pass diligence page after all diligence material has
been gathered. The Final Assessment is a single verdict block — Disposition, Evidence Synthesis
mapped to the original framework pillars, Net Assessment, and Standing Open Questions — written
in the same evidence-grounded voice as the in-doc net-assessment blocks throughout first-pass.

**This skill never flips the Opp Status.** The Final Assessment surfaces Tom's read; the
invest/pass decision and the corresponding Status flip stay with Tom.

**Re-running replaces, not stacks.** If a Final Assessment block already exists on the page, this
skill deletes it and prepends a fresh one. There is only ever one Final Assessment on the page at
a time. The corresponding `_vFinal.pdf` in Drive replaces any prior `_v*.pdf`,
`_vFinal.pdf`, or legacy `_v*_Final.pdf` snapshots (same retention rule as
`update-diligence-priors`).

**Re-finalize requires new material.** If a Final Assessment already exists, this skill refuses
unless an `## Update — …` block exists ABOVE the existing Final Assessment (i.e., a newer update
has been added since the last finalize). The intended workflow is: log new materials → run
`update-diligence-priors` → run `finalize-diligence`. Re-finalizing without intervening updates
is a no-op masquerading as work; refuse it.

---

## Execution model: three-subagent split

Run finalize-diligence as three sequential subagent stages with clean handoffs through `/tmp`
artifacts. Single monolithic subagent runs (one agent owning draft + lint + audit + PDF + Notion
+ Drive + Slack) bloat context, cascade failures, and obscure debugging. The split below isolates
synthesis from verification from publishing.

**Subagent A — Drafter (Opus, MANDATORY).** Owns Step 2 (assemble bundle + link map) + Step 3
(draft Final Assessment). Outputs:

- `/tmp/<company>_final_assessment.md` — the new Final Assessment block only.
- `/tmp/<company>_notion_link_map.json` — `{notion_id: title}` map of every Notion-housed source.
- `/tmp/<company>_finalize_sources.md` — full source bundle for the audit step.

Returns when draft is complete. Does NOT touch Notion, audit, PDF, Drive, or Slack.

**Subagent B — Verifier (Sonnet OK, parent OK).** Owns Step 5 (lint) + Step 6 (audit, iteration
loop up to 3). Inputs: the draft + sources + link map from Subagent A. Returns: lint pass/fail +
audit JSON summary. If iteration is needed, surfaces specific claims with notes; parent decides
whether to re-invoke Subagent A for fixes vs. surface as `⚠️ Audit:` warning.

**Subagent C — Publisher (Sonnet OK).** Owns Step 7 (PDF build — operates on a
local consolidated markdown snapshot built from the new FA draft + the existing page
content, NOT a post-Notion-prepend re-fetch), Step 4 (Notion prepend — uses REST
canonical per memory `feedback_mcp_insert_content_ordering_bug`), Step 8 (canonical
Drive upload + retention sweep + Notion link patches), and Step 9 (final Slack alert).
Inputs: verified draft + audit JSON + link map. Pure mechanical publish, no
re-drafting.

**NO preview gate — publish runs straight through.** Tom killed the old Step 7.5
preview-approval HALT on 2026-07-21 ("you shouldn't ask for my permission to do
this"): once Subagent B's verification passes (lint + audit reconciled +
speaker-attribution clean) and the three deterministic PDF gates pass, publish
autonomously — Notion prepend, Drive upload, retention sweep, links, alerts. The
Step 9 completion alert is Tom's review surface; if he wants changes he says
"revise <company> [feedback]" after the fact and the skill re-runs (re-finalize
replaces the FA block in place, so post-publish revision costs nothing).

**Parallelism inside Subagent C.** Step 7 (PDF build) operates on the consolidated page state
AFTER Step 4 (Notion prepend) completes, but Step 6 (audit, owned by Subagent B) operates on the
FA draft only — not the consolidated page. So Step 6 + Step 7 can run in parallel after Step 4
lands. If audit fails iteration, the in-progress PDF is discarded and rebuilt after the fix. Net
savings ~3-5 min on typical runs.

**Coordination.** Parent dispatches A → B → C with explicit `/tmp` handoffs. If B fails iteration
max (3 untraced iterations), parent decides whether to escalate to Tom or surface a publish-time
warning. The parent never inlines Drafter logic — Opus tier must own synthesis end-to-end (memory
`feedback_model_tier_framework`).

---

## Step 1: Locate the First-Pass Page + Guard Check

Search the Notes database (`collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) for the existing
first-pass diligence page. Use `notion-search` with the query
`[Claude] [Company Name] Master Diligence Doc`.

If no existing page is found, tell Tom and suggest running `first-pass-diligence` first. Do not
proceed.

Fetch the full page content with `notion-fetch` (pass `include_transcript: true` — the page may
embed linked call notes with transcripts you need to read). Walk the top-level block list and
identify the structural anchors:

- The topmost block (after the page title).
- Any existing `# Final Assessment — …` H1 anchor.
- Each `## Update — …` H2 anchor and its date.
- The `# First-Pass Diligence — …` H1 anchor (present if any update has run).

### Guard: re-finalize requires new material

If an existing `# Final Assessment — …` anchor is present, check whether at least one
`## Update — …` anchor sits ABOVE it in document order. If yes, proceed. If no, refuse:

> "There's already a Final Assessment dated [date] at the top of the page with no newer Update
> above it. Run `update-diligence-priors` first to log new materials, then re-run finalize."

If no existing Final Assessment is present, proceed regardless of update count (first-time
finalize is always allowed, even directly off the original first-pass).

---

## Step 2: Assemble the Full Diligence Bundle (Subagent A)

Unlike `update-diligence-priors` (which reads only NEW material since the last run), finalize
reads everything. The closing call must reconcile against the entire diligence record.

**HARD RULE — every company-provided material is in scope, regardless of filename labels.**
Process every entry in the Diligence Materials property field. Do NOT skip files because the
filename or title contains `[DEPRECATED]`, `(deprecated)`, `OLD`, `[OLD]`, `superseded`, `v0`,
`archived`, `graveyard`, or any other founder-applied "this is no longer current" tag. Every
artifact the founders provide goes into the Final Assessment. Founder-marked deprecation is
itself diligence signal — include the file with a `(deprecated)` qualifier inline, and treat
the contrast between deprecated and current as material for the Diligence Journey and Net
Take sections (what changed, what was walked back, what the rename or pivot signals). The
only valid filters when building the materials set are (a) is this the diligence output the
skill itself wrote (e.g., the prior consolidated PDF), and (b) is this a duplicate of another
material in the set. Nothing else. Memory:
`feedback_diligence_materials_deprecated_not_skip`. Caught after AgentBay 2026-06-25.

**Classification rule.** All company-provided artifacts (decks, financial models, docs,
slides, transcripts, screenshots founders sent) belong in the Diligence Findings as
company-provided materials — not as "Customer Feedback" or other reader-feedback categories.
Tom-authored framework docs (Diligence Q&A, question lists, pre-mortems) are Research &
Analysis. Drive owner email is the tiebreaker: company-domain owner = Materials;
tom@invertedcap.com owner = Research & Analysis.

### 2.0 Enumerate the Notion link map — MANDATORY

Before reading content, enumerate every URL in the Opp's `✍️ Notes` relation. For each Notion
page in the relation:

1. Run `notion-fetch` lightweight (title only — do NOT pass `include_transcript: true` at this
   stage; that fires in step 2.3 only for sources the draft actually consumes).
2. Capture `{notion_id, title, url}`.

Persist the result to `/tmp/<company>_notion_link_map.json` as:

```json
{
  "<notion_id_a>": {"title": "Mike Barbosa — Diligence Call", "url": "https://www.notion.so/<id_a>"},
  "<notion_id_b>": {"title": "AtoB Reference Cohort Notes", "url": "https://www.notion.so/<id_b>"},
  ...
}
```

**Drafting in Step 3 MUST cross-reference this map.** Any footnote citing a Notion-housed source
emits a clickable `[Title](https://www.notion.so/<id>)` markdown link in the `^N` footnote line
— not a bare label. This applies uniformly: founder calls, backchannel/feedback notes, expert
references, pre-mortem entries, AtoB reference notes, anything in the Notes relation. No
exceptions.

**Caught after:** Factir 2026-05-22 finalize Final Assessment shipped with bare-label footnotes
on the founder-call and Mike Barbosa references — Tom flagged the missing Notion URLs as a hard
drafting miss. Memory `feedback_finalize_diligence_notion_link_map` applies in full.

### 2.1–2.7 Fetch and read

1. **The full Notion diligence page** — original analysis + every update section. Read the
   prose, not the summary blocks. If transcripts are embedded, read them.
2. **The current Opportunity page** — Notion Opportunities DB
   (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Get current Status, Round Details, every relation
   field, page body.
3. **Every linked Note** — `✍️ Notes` relation on the Opp. Pass `include_transcript: true` on
   every call note / meeting note fetch (per memory feedback: summarize call/note from full
   transcript, not summary block). **Speaker-label every transcript** via
   `~/.claude/skills/first-pass-diligence/relabel_transcript.py` and cache to
   `/tmp/firstpass_labeled_transcripts/` (already-labeled transcripts from prior runs are
   reused). The Final Assessment lint check `attribution_mismatches` in Step 5 reads from
   this directory to verify deterministic speaker attribution — without labeled transcripts,
   the check is skipped with a warning. See memory
   `feedback_transcript_speaker_attribution_to_tom` for the failure mode this catches.
4. **Every Diligence Material** — Drive PDFs, Google Docs, Notion attachments, DocSend links.
   Use the type-specific access methods from `first-pass-diligence` Step 1c.
5. **Investment memo manifest** — read every memo in the canonical Drive folder
   `1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`. The Evidence Synthesis pulls comparative anchors from
   the manifest the same way the original first-pass does.
6. **Feedback / pre-mortem artifacts** — if the Opp has linked feedback notes (intro replies,
   backchannel diligence) or a pre-mortem note, read them.
7. **`FEEDBACK_PATTERNS.md`** at
   `/Users/tomseo/.claude/skills/first-pass-diligence/FEEDBACK_PATTERNS.md` — taste corrections
   on prior first-passes. Apply them to voice/structure decisions in Step 3.

You should be able to enumerate, from memory after Step 2, every piece of evidence on the page
and what specific prior it bears on. If you can't, re-read.

### Source-bundle cache

Persist the assembled bundle to `/tmp/<company>_finalize_sources.md` as a side-effect of Step 2
(this same file is the audit input in Step 6). On any subsequent re-run within the same finalize
cycle (re-finalize, post-audit-iteration redraft), check `mtime` — if < 24h old AND the Opp's
`✍️ Notes` relation hasn't grown since the file's mtime, reuse the cached bundle instead of
re-fetching. The 211KB-class bundles typical of mid-diligence Opps take ~30s to re-assemble per
iteration; caching collapses that to a stat call. Invalidate by deleting the file.

---

## Step 3: Draft the Final Assessment (Subagent A)

Draft the standalone Final Assessment block — Overview + Thesis (with Need to Believes) +
Diligence Journey (organized by NTB) + Standing Open Questions + Footnotes — in the
evidence-grounded LP/memo voice, readable cold by someone who lands on the block alone.

**Full procedure in `references/step-3-final-assessment-format.md` — read it now before
proceeding.** It carries the section skeleton, the header/formatting contract table, the
voice & citation rules, and the "what this block must NOT do" list.

**MANDATORY — drafting runs on Opus tier.** Per memory `feedback_model_tier_framework`, the
Final Assessment is a no-rubric final-artifact task. If you delegate, pass `model: "opus"`
explicitly (a `general-purpose` subagent inherits the parent's model and silently degrades on
Sonnet); if you're on Sonnet and can't promote, stop and tell Tom. Full tier-routing detail in
the reference.

**Header / formatting contract — MANDATORY, DETERMINISTIC.** The block MUST emit headers exactly
per the contract table in the reference; deviations break the PDF visual hierarchy Tom
workshopped.

**No Decision / Disposition section.** The invest/pass call is Tom's — flipped via the Opp
Status in the Opportunities DB, never stated inside the Final Assessment
(`project_committed_is_pipeline_not_portfolio`).

---

## Step 4: Replace Existing Final Assessment (if any) + Prepend New (Subagent C)

Delete any existing Final Assessment block range and prepend the new one, PATCH the page title
to the `… Final` suffix, and confirm the `:claude-color:` icon. REST is canonical for delete +
prepend + title; MCP `insert_content` is a narrow fallback for complex embeds only (its
multi-block ordering is non-deterministic — `feedback_mcp_insert_content_ordering_bug`).

**Full procedure in `references/step-4-notion-write.md` — read it now before proceeding.** It
carries the parallel block-delete (4.1), the canonical insert-after-first-block + delete prepend
pattern (4.2), the title PATCH (4.3), the icon check (4.4), and publish-progress alert 1 of 3.

**Destructive op acknowledgment.** The block delete is the prescribed step of an
explicitly-invoked skill — the `feedback_always_confirm_before_delete` exemption applies.
Proceed autonomously, but state the delete scope in the run summary.

**MCP timeout handling — MANDATORY** (only if the MCP fallback is in use): on
`notionhq_client_request_timeout`, do NOT retry — wait, verify the H1 via REST, retry only if
absent.

**Post-prepend ordering verification — MANDATORY.** After the prepend lands, list the first 10
top-level blocks and confirm the H1 → Overview → Thesis → Diligence Journey → Standing Open
Questions → Footnotes order. If scrambled, delete the misplaced blocks and re-prepend — never
silently publish a scrambled FA.

---

## Step 5: Pre-PDF Lint — MANDATORY GATE (Subagent B)

BEFORE the Notion prepend (Step 4) and BEFORE PDF build, run the deterministic shape-rule lint
(`finalize_lint.py`, rules F1–F11) over the FULL rebuilt markdown (new FA + every Update + inner
First-Pass), built locally so failures are caught while the Notion page is still untouched.

**Hard gate.** Exit code 0 = proceed. Exit code 1 = STOP, fix the underlying issue, re-run. The
rules it catches all manifest in the published PDF.

**Full procedure in `references/step-5-6-audit-gates.md` — read it now before proceeding.** It
carries the exact invocation and the F1–F11 rule list.

---

## Step 6: LLM Audit Gate — MANDATORY (Subagent B)

Run the research-artifact-audit discipline (`feedback_research_artifact_self_audit`) over the new
Final Assessment block against the verbatim source bundle, iterating up to 3× on untraced claims.
If untraced claims persist after 3 iterations, publish with a `⚠️ Audit: <N> untraced` warning.

**Full procedure in `references/step-5-6-audit-gates.md` — read it now before proceeding.** It
carries the audit-runner invocation, the source-bundle skeleton + verbatim mandate, the
iteration loop, the chunking gotcha, and the judge scope rules.

This gate has three embedded sub-gates — each stays MANDATORY and must be run:

- **Source bundle completeness check — MANDATORY GATE before audit** (`bundle_completeness_check.py`,
  `--self-page-ids` required). Exit 1 = rebuild the bundle; an audit on a broken bundle is
  structurally meaningless.
- **Speaker-attribution script gate — MANDATORY** (`verify_speaker_attribution.py` over each
  labeled transcript from Step 2.3). Every flagged claim must be re-attributed (to Tom when Tom
  introduced the framing) or reworded before publish.
- **MANDATORY — verify the audit JSON before declaring this step done.** Run `jq '.summary'` and
  show the literal output in the run trace; if `untraced > 0`, dump the offending claims. Never
  self-report "audit clean" without the jq summary in hand
  (`feedback_skill_self_report_diverges_from_actual_write`).

---

## Step 7: Build the Consolidated PDF (Subagent C — runs in parallel with Step 6)

Build `[Company]_Master_Diligence_MM.DD.YYYY_vFinal.pdf` from the post-prepend consolidated page
(new Final Assessment at top + every Update newest-first + the original first-pass below), using
the canonical PDF template. This runs concurrently with the Step 6 audit; if the audit fails
iteration, discard the in-progress PDF and rebuild once the FA fix lands. There is only ever ONE
`_vFinal.pdf` per company — re-finalize overwrites it (retention sweep in Step 8).

**Full procedure in `references/step-7-pdf-build.md` — read it now before proceeding.** It
carries the filename rule, the page-break + multi-citation-regex requirements, the
header/subtitle spec, the iCloud save path, and the three publish gates.

**Deterministic publish gates — MANDATORY before Drive upload.** Three guards must pass before
the Step 8 Drive upload; do NOT upload until all three exit 0:

- **Gate 1 — markdown sanity** (`markdown_sanity_check.py`, after extractor / before PDF build):
  no debug-output leak into the body, correct leading H1.
- **Gate 2 — PDF header check** (`pdf_header_check.py`, after build / before upload): title +
  subtitle match the canonical Master Diligence Doc shape.
- **Gate 3 — PDF format guard** (`pdf_format_guard.py`, after build / before upload): every
  text-span's `(font, size, color)` matches the measured Factir reference profile — catches
  citations rendered as body text, colored links, wrong heading sizes, wrong page size.

---

## Step 8: Upload + Replace in Notion + Retention Sweep (Subagent C)

Upload the new PDF to the same company subfolder under Diligence root
`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`. Use the Apps Script endpoint — same as
`update-diligence-priors` Step 5b. The `createFolder` call is idempotent and returns the
existing subfolder if it's already there.

**Fire publish-progress alert (2 of 3) — immediately after the upload returns
`file_url`.**

```bash
COMPANY="<subject company name>"
PDF_URL="<file_url from upload response>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
📄 **${COMPANY}** Final PDF uploaded to Drive — [vFinal PDF]($PDF_URL). Running retention sweep + linking next.
EOF
```

### Retention sweep — destructive, autonomous

Per memory `feedback_diligence_pdf_retention`: the new `_vFinal.pdf` is a full
consolidated snapshot containing every prior update. Once uploaded and linked, trash every
prior diligence-snapshot PDF in the same Drive subfolder.

```bash
rclone lsf "gdrive:Diligence/[Company]"
# For each [Company]_Master_Diligence_*.pdf AND legacy [Company]_First_Pass_Diligence_*.pdf
# that is NOT the new one:
rclone deletefile "gdrive:Diligence/[Company]/<OLD_NAME>.pdf" --drive-use-trash
```

Do NOT trash source materials in the same folder (founder memo, deck, plan, ACV build,
positioning notes, Q&A docs, etc.) — only prior diligence-snapshot PDFs (`_v*.pdf`,
`_v*_Final.pdf`, `_Update.pdf`, etc.) that this skill or `update-diligence-priors` previously
wrote.

State the scope in the run summary: "Trashed N prior diligence-snapshot PDFs from
gdrive:Diligence/<Company>/."

### Update Notion links

The new `_vFinal.pdf` becomes the only diligence link anywhere on the Opp. Update both
locations via REST API (per `update-diligence-priors` Step 5b — MCP is slower and less reliable
on text-only patches):

1. **Opp page body** — PATCH the existing `## 📎 Diligence Materials` bullet's `rich_text` in
   place. New label text: `[Company]_Master_Diligence_MM.DD.YYYY_vFinal.pdf — Final
   Claude diligence snapshot (consolidates Final Assessment + all updates + original
   first-pass)`. If no existing bullet, POST a new one with `after: <heading_2 id>`.

2. **`Diligence Materials` files-property** — follow
   `~/.claude/skills/shared-references/add-link-to-diligence-materials.md`. After PATCH, re-fetch
   the Opp page and verify the new Drive URL is in the `files[*].external.url` set. If absent
   after 3 retries, surface to Tom rather than publish silently (the MANDATORY verification
   pattern from update-priors Step 5b applies in full).

3. **Scrub stale URLs** — any entry in the files-property pointing at a trashed Drive file must
   be removed. Any bullet in the page body still pointing at a trashed PDF must be patched to
   the new URL.

**Fire publish-progress alert (3 of 3) — once the Diligence Materials write
verifies and stale URLs are scrubbed.**

```bash
COMPANY="<subject company name>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
🔗 **${COMPANY}** Diligence Materials property updated + stale URLs scrubbed. Sending completion alert.
EOF
```

Per memory `feedback_no_permission_for_user_initiated_analysis` and
`feedback_first_pass_no_permission_prompts`: all writes (PATCH, POST, DELETE) execute
autonomously. No pre-flight confirmation prompts inside this skill.

---

## Step 9: Send Signal Alert (Subagent C)

Read `~/.claude/skills/send-alert/SKILL.md` for the chatID and guardrails. Use the canonical
`claude` Slack webhook (NOT the MCP, NOT Beeper — per memory
`reference_slack_notification_channel`, scheduled-style alerts post via the webhook as the
`claude` app).

Body format (GFM links, NEVER Slack mrkdwn — per memory
`feedback_send_alert_gfm_not_mrkdwn`):

```
✅ [Company Name] — Diligence Finalized

Thesis: [One-sentence verbatim from the opening Thesis paragraph]

Standing open questions: [N]

[Notion](<notion_page_url>) | [PDF](<drive_url>)
```

The alert names what the final artifact CONTAINS (the thesis the diligence record
underwrote), not what Tom decided. Status flips happen separately in the Opp DB and
trigger their own webhook events.

If the audit gate exited with residual untraced claims after max iterations, append:

```
⚠️ Audit: <N> untraced after 3 iterations — review before relying on cited claims.
```

---

## Quality Bar

A good Final Assessment:

- States the disposition up front, in one paragraph. The reader knows the call before reading
  the Evidence Synthesis.
- Reconciles every NTB item from the original analysis — confirmed, partially confirmed,
  unresolved, or refuted — with a single citation anchor each.
- Pulls evidence from BOTH the original first-pass AND the update blocks. If the Evidence
  Synthesis only cites the original, the updates didn't move the analysis — flag that
  explicitly rather than ignoring it.
- Resolves the strongest counter-argument explicitly in the Net Assessment.
- Standing Open Questions are real residual risks the Disposition accepts. Three sharp ones
  beats ten cosmetic ones.
- Does NOT flip the Opp Status. The Status flip is Tom's separate action.

A bad Final Assessment:

- Hedges. "On balance, this might be worth considering." Stop and rewrite.
- Re-narrates the original first-pass instead of reconciling against it.
- Reads as a neutral summary. The point is a call, not a recap.
- Adds Standing Open Questions that are framework boilerplate rather than real residual risks.
- Cites only the original first-pass and ignores every update block.
