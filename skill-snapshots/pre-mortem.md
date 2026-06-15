---
name: pre-mortem
description: "Draft a structured pre-mortem for an investment opportunity Tom is actively considering or has committed to. A pre-mortem imagines the investment has already failed and works backward to surface the most plausible failure modes before confirmation bias sets in. Trigger this skill whenever Tom says 'run a pre-mortem', 'pre-mortem this deal', 'stress-test this investment', 'what could go wrong with X', 'devil's advocate on X', 'what breaks the thesis on X', or any variant asking for a forward-looking failure analysis on a specific opportunity. Also trigger when Tom asks to 'pressure-test' or 'stress-test' a deal thesis, even if he doesn't use the word pre-mortem. Always trigger inline — no confirmation needed before acting."
---

# Pre-Mortem Skill

Draft a rigorous, evidence-grounded pre-mortem for an investment opportunity and save it as a linked note in the Notion Notes database. The pre-mortem imagines a future failure and works backward — it is a structured intellectual exercise designed to surface kill shots before capital is deployed, not a refutation of the thesis.

---

## Step 1: Gather Diligence Materials

Before drafting, you need the full evidence base. Do all of the following:

1. **Fetch the Notion Opportunity page** for the company. Use `notion-fetch` on the Opportunity URL or search for it by name in the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Extract:
   - All properties (stage, round details, status, scores, open questions)
   - All linked Notes (✍️ Notes relation field) — fetch each one
   - Diligence Materials field — fetch any linked Google Docs

2. **Fetch all linked Notes** — these typically contain call notes, Claude research threads, backchannel references, and framework mapping. Fetch each URL in the Notes relation. Prioritize:
   - Founder meeting notes
   - Backchannel / reference call notes
   - Market research reports
   - Framework mapping / scorecard entries
   - Pre-memo diligence questions

3. **Fetch Diligence Materials** — use `google_drive_fetch` on any Google Doc IDs in the Diligence Materials field. These typically include the investment memo and scratch notes.

4. **Review Tom's investment frameworks** — list all files in the Google Drive investment memos
   folder (`1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`) and read every memo in the folder. These memos
   are the source of truth for Tom's Inverted Lens frameworks and portfolio vocabulary. Extract
   the recurring analytical lenses (data assets, founder orientation, intentionality, distribution
   leverage, etc.) from the memos themselves rather than relying on a static list — the frameworks
   evolve as the portfolio grows. The pre-mortem should be written in Tom's analytical voice and
   consistent with his framework vocabulary as expressed in the memos.

---

## Step 2: Draft the Pre-Mortem

Write the pre-mortem in Tom's voice: analytical, first-person, evidence-specific. Do not use generic VC risk language. Every failure mode must be grounded in something specific from the diligence record — a quote, a gap, an unresolved question, a competitive data point. See the reference file at `references/writing-guide.md` for detailed style and voice guidance.

### Structure

**Opening Frame** *(1 paragraph)*
Set the hypothetical: "It is [Year+3]. The investment in [Company] returned zero..." Name the three to four load-bearing beliefs the investment rests on. State that the exercise stress-tests each in turn.

**Failure Mode Taxonomy** *(4–5 failure modes)*
For each failure mode, write a titled section with:
- **Probability and Severity label** (e.g., *Probability: Medium-High | Severity: Fatal*)
- **2–3 paragraphs** of analytical prose grounded in the actual diligence record
- Each mode should be *specific* — not "market risk" but the precise mechanism by which the market fails for this company

Use these four canonical failure categories as your framework, then add a fifth if the company has a specific structural risk worth isolating:

1. **Category / Market Formation Risk** — Does the market category exist as a distinct, payable thing? Who owns it if it crystallizes? What incumbent or platform could absorb it?
2. **Product / Technical Loop Risk** — Is the core product loop (the thing that makes the thesis compound) actually built and validated, or is it roadmap? Where does the evidence trail go cold?
3. **GTM / Commercial Execution Risk** — Can the founder(s) sell? To whom, with what motion, at what pace? Is there evidence of a repeatable commercial motion beyond the first design partner?
4. **Team / Organizational Fragility Risk** — What does the company look like when it hits organizational stress — a bad hire, a co-founder gap, a key customer churn? Does the team structure hold? Also surface **founder–thesis journey-length mismatch**: does the founder's profile match the journey length the thesis requires? Conservative founders who hit an early acquisition window are predictably likely to take it (Mason / Vantager — exited to a publicly-traded acquirer at 24/25). If the thesis depends on a multi-year build (category creation, deep tech, regulatory unlock), check whether the founder's track record signals stamina for that journey vs. propensity to take a clean intermediate exit. Surface as a probability adjustment on the venture-outcome distribution, not a kill shot.
5. **Thesis-Specific Risk** (if applicable) — A fifth failure mode unique to this deal's central thesis (e.g., the data moat thesis, the regulatory tailwind thesis, the category-definition bet).

**Kill Shot Ranking** *(table)*
A 4–5 row table ranking each failure mode by Probability, Severity, and whether it's a Kill Shot (independent path to zero) or Conditional.

**Load-Bearing Priors and What Would Break Them** *(numbered list)*
3–4 explicit prior beliefs that underpin the investment. For each, write the precise conditions under which it breaks. These should be falsifiable.

**Diligence Requirements Derived From This Pre-Mortem** *(bullet list)*
5–7 specific, actionable questions or evidence requests that flow directly from the failure mode analysis. These should be concrete enough to bring into a founder conversation or backchannel call.

**Closing note** *(1 line)*
Date the document and specify when it should be revisited (e.g., "Q3 2026 or first paid customer milestone, whichever comes first").

---

## Step 2.5: LLM Audit Gate — MANDATORY

Per memory `feedback_research_artifact_self_audit`, long-form research artifacts get the
research-artifact-audit discipline before delivery. A pre-mortem is squarely in scope —
factual claims about analog company failures, competitor histories, market dynamics, and
regulatory precedent are the evidentiary backbone of the failure-mode analysis, and they
must trace to the source bundle.

The audit mechanics (HARD EXIT GATE, un-chunked re-verification, iteration loop, Step 3.5
partial normalization, exit codes, the never-use-the-bypass-alert prohibition) are defined
ONCE in `~/.claude/skills/research-artifact-audit/SKILL.md`. **Read that file in full
before continuing this step** so the A.0 verbatim mandate, A.1 bundle completeness gate,
and B.2.1 un-chunked re-verification all inherit correctly.

**Scope:** the audit fires on the full pre-mortem draft, but the judge prompt is narrowed
to factual claims only. Failure-scenario speculation, prospective probability assignments,
and opinion-class statements about how leadership might respond are EXCLUDED — that's the
whole point of a pre-mortem exercise. The judge prompt at
`~/.claude/skills/pre-mortem/pre_mortem_audit.prompt.md` enforces this scope.

### Pre-mortem-specific wiring — apply these bindings when following research-artifact-audit/SKILL.md

| Binding (per research-artifact-audit) | Value (pre-mortem) |
|---|---|
| `DRAFT` | `/tmp/<company>_premortem_only.md` |
| `SOURCES` | `/tmp/<company>_premortem_sources.md` |
| `AUDIT_JSON` | `/tmp/<company>_premortem_audit.json` |
| `JUDGE_PROMPT` | `/Users/tomseo/.claude/skills/pre-mortem/pre_mortem_audit.prompt.md` |
| `AUDIT_RUNNER` | `/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.py` |
| `MAX_ITER` | `3` |
| `WEB_RESEARCH_CAP` | `6` |
| `ITER_SNAPSHOT_PREFIX` | `/tmp/<company>_premortem_draft.iter` |
| `NORMALIZED_DRAFT` | `/tmp/<company>_premortem_draft.normalized.md` |

### Pre-mortem-specific source bundle structure (Step A in research-artifact-audit)

Write `/tmp/<company>_premortem_sources.md` with this layout. The A.0 verbatim mandate
from research-artifact-audit applies in full — every section must contain the full
verbatim body of each cited source, NOT pointer manifests or summaries.

```markdown
==== MASTER DILIGENCE DOC ====

<full current state of the Master Diligence Doc page — first-pass + every Update block
in document order. The pre-mortem reconciles against the documented thesis, load-bearing
priors, and prior diligence findings. This section is authoritative ground truth.>

==== LINKED NOTES ====

--- Note: <title> (<date>) ---
<full verbatim body of every cited call/backchannel note, including the Notion ID
anchor, the page URL, the full transcript (always fetch with include_transcript: true
for call/meeting notes), the AI-summary block, and structured fields. NOT a pointer
manifest. NOT a summary.>

==== DILIGENCE MATERIALS ====

--- Material: <name> ---
<full extracted text from every cited Drive PDF / Google Doc / DocSend / Notion
attachment — pdftotext / google_drive_fetch output, not a pointer or summary.>

==== ANALOG / COMPETITOR REFERENCES ====

--- Reference: <name> ---
<verbatim text of every cited analog company case study, competitor reference, prior
postmortem, market report, or comparator memo. The pre-mortem may cite these heavily
for failure-mode triangulation — every analog-company failure mode, regulatory
precedent, or competitor-shutdown claim must be grounded here or in the diligence
record. Include the source URL / Drive ID / Notion page anchor for each entry.>
```

### Source bundle completeness check — MANDATORY GATE before audit

The bundle completeness gate is canonical in `research-artifact-audit` Step A.1 — read
that for the full spec. The pre-mortem binding is:

```bash
python3 ~/.claude/skills/shared-references/bundle_completeness_check.py \
    --draft  /tmp/<company>_premortem_only.md \
    --bundle /tmp/<company>_premortem_sources.md \
    --notes-section 'LINKED NOTES' \
    --required-sections 'LINKED NOTES,DILIGENCE MATERIALS,ORIGINAL FIRST-PASS MEMO' \
    --self-page-ids <Master Diligence Doc page ID, 32hex no hyphens>
```

The `--self-page-ids` argument is REQUIRED — pre-mortems frequently cite the Master
Diligence Doc's own first-pass section or Update sections via the page's own URL.
Without exempting the self-page-id, B2 will false-positive on those self-references.
Look up the Master Diligence Doc page ID from the Opportunity's `✍️ Notes` relation
(it's the `[Claude] [Company] Master Diligence Doc` page).

Note: `--required-sections` lists `ORIGINAL FIRST-PASS MEMO` for naming continuity with
the canonical caller table in research-artifact-audit Step A.1, but the actual section
header in the pre-mortem bundle is `MASTER DILIGENCE DOC` (which contains the original
first-pass plus all updates as a single ground-truth block). The check matches by
substring `in k`, so adjust `--required-sections` to
`'LINKED NOTES,DILIGENCE MATERIALS,MASTER DILIGENCE DOC'` if matching is tighter than
expected.

Exit 0 = proceed to audit. Exit 1 = bundle is missing verbatim content. STOP, rebuild,
re-run. The audit verdict on a broken bundle is structurally meaningless.

### Draft to audit

Build `/tmp/<company>_premortem_only.md` containing JUST the pre-mortem markdown body
(Opening Frame through Closing note). This is identical to what Step 3 will write to the
Notion page — the audit fires on the publish-ready text.

### Run the audit + iterate

Follow research-artifact-audit Steps B-D in full — invoke the runner, check the HARD EXIT
GATE, run B.2.1 un-chunked re-verification if chunked + untraced > 0, iterate per B.3,
normalize partials per Step C, surface remaining findings per Step D.

The final published pre-mortem markdown is `$NORMALIZED_DRAFT` when Step C ran (partials
> 0); otherwise `$DRAFT`. Select the source-of-truth file deterministically before
Step 3's Notion write so partials Step C just resolved aren't re-introduced:

```bash
if [ -f /tmp/<company>_premortem_draft.normalized.md ]; then
  FINAL_PREMORTEM_MD=/tmp/<company>_premortem_draft.normalized.md
else
  FINAL_PREMORTEM_MD=/tmp/<company>_premortem_only.md
fi
echo "publishing from: $FINAL_PREMORTEM_MD"
```

Use `$FINAL_PREMORTEM_MD`'s contents as the `content` field in Step 3's
`notion-create-pages` call.

### Audit-result surface

If the audit ends with residual untraced findings after max iterations OR any partials
were normalized, surface this in the final summary message to Tom alongside the Notion
URL: `⚠️ Audit: <N> untraced after <K> iterations, <M> partials normalized`. Include
the residual untraced claims with judge notes and any normalized partials as
before→after diffs per research-artifact-audit Step D.

---

## Step 3: Save to Notion

Create a new page in the Notes database (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) with the following:

```
parent: { data_source_id: "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" }
pages: [{
  properties: {
    "Name": "Claude Pre-Mortem: [OPPORTUNITY NAME]",
    "Opportunity": "[Opportunity page URL]"
  },
  content: "[Full pre-mortem content in Notion-flavored markdown]"
}]
```

After creating the page, confirm the URL to Tom.

---

## Step 4: Set Claude Icon

Read the shared reference at `/Users/tomseo/.claude/skills/shared-references/claude-note-icon.md` and follow its instructions to set the custom Claude logo emoji as the page icon on the newly created note. This is required for all Claude-generated notes — do not skip.

---

## Timing

The pre-mortem is most valuable when written:
- **Before first substantive call** (cold, before the founder has built rapport) — highest signal, hardest to do
- **Before closing** — the second-best window; still pre-deployment, but confirmation bias has partially set in
- **After closing, before board seat** — useful for portfolio monitoring even if the pre-closing window was missed

When in doubt about timing, note it in the opening frame of the document.

---

## Quality Bar

A good pre-mortem has the following properties:
- Every failure mode cites specific evidence from the diligence record (a quote, a data point, a gap explicitly named in the notes)
- No failure mode is purely generic — each is calibrated to this company's specific product, market, and team
- The kill shot ranking reflects honest probability assessment, not wishful thinking
- The load-bearing priors are stated as falsifiable claims, not vague thesis summaries
- The diligence requirements are concrete enough to be actioned in the next founder or backchannel conversation

A bad pre-mortem reads like a generic VC risk section. If you find yourself writing "execution risk" or "market timing risk" without grounding those in specific evidence from this company's diligence record, stop and go back to the materials.

**Specificity gate (enforced, not aspirational).** Each failure mode's evidence must be specific to THIS company's conditions — named facts from the diligence record: this team's composition, this product's architecture, this round's terms, this customer set's behavior. Category-generic risk prose with generic citations FAILS the gate even when every citation technically traces. On a violation, re-prompt the drafter ONCE with the failing failure modes named and the specific diligence-record facts they should anchor on.

**Semantic-grounding re-prompts are capped at MAX_ITER = 2.** Quality-bar / specificity re-prompts (re-drafting a failure mode because it is generic or ungrounded) get at most two rounds total. After the second re-prompt, STOP re-drafting. Surface every unresolved violation in the run report / final summary to Tom flagged as **"Audit caveat"** — never silently publish past unresolved violations, and NEVER write an escape-hatch caveat line into the published artifact itself. Caveats live in the run report only; the published pre-mortem stays clean. (This cap governs quality-bar re-prompts only — the Step 2.5 LLM audit gate keeps its own MAX_ITER = 3 per research-artifact-audit.)


---

## Final Step: Classify the Note

After the Notion page has been created and the Opportunity relation (if any) has been set, run the `note-classifier` skill to assign the correct `Category` field.

Read the skill at `/Users/tomseo/.claude/skills/note-classifier/SKILL.md` and follow its classification logic against the note just created. Do not skip this step — every note created by this skill must have a Category set.

## Behavior Rule — PDF rendering uses the long-form spec

A pre-mortem is a long-form document. When rendering it (or any authored long-form artifact — memo, PRD, LP letter, investor report) to PDF, default to the canonical long-form PDF spec at `~/.claude/skills/shared-references/long-form-pdf-spec.md` using `~/.claude/skills/shared-references/pdf_builder_template.py` as the template.

That means: reportlab (not Chrome headless), Helvetica (not JetBrains Mono), black + white only (no orange/accent colors), underlined H1, italic 9pt #333 subtitle, page numbers bottom-center, no horizontal dividers.

**Why:** `design-language.md` (dark theme, JetBrains Mono) is for PNG exports and on-brand visual artifacts (skill maps, charts, LP letter exhibits). It is NOT the default for long-form authored documents. Tom had to redirect the first Founder Framework PRD render after it used Chrome + design-language styling; the canonical long-form spec is the right default.

**How to apply:** If the thing being rendered is prose-heavy with sections/tables/bullets (memo, PRD, pre-mortem, LP letter, investor report), use the long-form spec. If it's a single visual artifact (chart, diagram, skill map card, single-page exhibit embedded in Docs), use design-language.md. When in doubt on anything PRD-shaped, long-form wins.
