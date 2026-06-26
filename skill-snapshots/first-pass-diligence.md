---
name: first-pass-diligence
description: >
  Run a first-pass diligence analysis on a pipeline opportunity from the Notion CRM.
  Pulls all available context (opportunity page, call notes, diligence materials, investment
  memos), evaluates the company against Tom's Inverted Lens frameworks, conducts web research
  to enrich open questions with data, and produces three outputs: (1) a Notion page with the
  full analysis as the living artifact, (2) a formatted PDF, and (3) a Signal alert via Beeper
  with the Notion link. Trigger whenever Tom says "first pass", "first-pass diligence",
  "run diligence on [company]", "diligence analysis on [company]", "first pass on [company]",
  "analyze [company]", "diligence [company]", or any variant requesting an initial structured
  diligence analysis on a named opportunity. Also trigger when Tom says "run the diligence
  skill on X" or "do a first pass on the Lex opportunity" or similar. Always trigger inline —
  no confirmation needed before acting.
---

# First-Pass Diligence

Generate a comprehensive first-pass diligence analysis for a pipeline opportunity. This skill
produces an evidence-grounded, framework-driven analysis that Tom can use to decide whether to
proceed to deeper diligence. The analysis is deliberately opinionated — it should surface yellow
flags, contradictions between founder claims and independent findings, and gaps in the evidence
base, rather than presenting a sanitized summary.

---

## Skip first-pass on follow-on rounds

Never run this skill on a follow-on round Opportunity. First-pass is for net-new
companies — Tom already has full context on prior investments and the framework is
mis-calibrated for follow-ons. Detect via either signal:

1. `"FO"` in parens anywhere in the Opportunity name (e.g., `Caplight (FO)`, `Caplight FO`) — Tom's near-universal convention
2. Earlier Opportunity entries for the same company in Notion

If either is present, do not run first-pass. Confirm with Tom what he actually wants
— likely an `update-diligence-priors` refresh against the original thesis, or just
materials handling for the new round.

---

## Step 0: Set up per-job workspace

Concurrent first-pass-diligence jobs (webhook-triggered runs and
diligence-agent parallel workers) used to share `/tmp/firstpass_*` paths,
which caused workspace collisions on 2026-06-24: AgentBay's webhook run
aborted after diligence-agent's parallel workers for Durable / Orbital /
Quiet AI overwrote `/tmp/firstpass_draft.md` and `/tmp/firstpass_manifest.json`
mid-run.

Every job MUST scope its working files under a per-page-id directory. Export
`WORKSPACE` once at the top and use `$WORKSPACE/...` for every per-job path
in this skill.

```bash
# PAGE_ID = the Opportunity page UUID with hyphens.
# Derive it from whichever input you have:
#   - Webhook invocation: from args.page_id (e.g., "38200bef-f4aa-81ba-94bd-edfa3d08b3b3").
#   - diligence-agent worker invocation: extract from the Opportunity URL
#     (last 32 hex chars; insert hyphens as 8-4-4-4-12).
#   - Manual invocation: from the Notion URL you fetched in Step 1a.
export PAGE_ID=<page_id from args, or extracted from Opportunity URL>
export WORKSPACE=/tmp/firstpass-${PAGE_ID}
mkdir -p "$WORKSPACE"
```

Every `/tmp/firstpass_<file>` path in this skill is rewritten to use
`$WORKSPACE/<basename>`. The mapping is:

| Old (shared, collision-prone) | New (per-job, isolated) |
|---|---|
| `/tmp/firstpass_draft.md` | `$WORKSPACE/draft.md` |
| `/tmp/firstpass_sources.md` | `$WORKSPACE/sources.md` |
| `/tmp/firstpass_audit.json` | `$WORKSPACE/audit.json` |
| `/tmp/firstpass_manifest.json` | `$WORKSPACE/manifest.json` |
| `/tmp/firstpass_start_ts.txt` | `$WORKSPACE/start_ts.txt` |
| `/tmp/firstpass_raw_transcripts/` | `$WORKSPACE/raw_transcripts/` |
| `/tmp/firstpass_labeled_transcripts/` | `$WORKSPACE/labeled_transcripts/` |
| `/tmp/firstpass_draft.iter*` | `$WORKSPACE/draft.iter*` |
| `/tmp/firstpass_draft.normalized.md` | `$WORKSPACE/draft.normalized.md` |
| `/tmp/firstpass_demo_*` | `$WORKSPACE/demo_*` |
| `/tmp/firstpass_product_screens_*` | `$WORKSPACE/product_screens_*` |

Exception (shared, read-mostly cache): **`/tmp/firstpass_memos/`** stays at
that path so portfolio memos accumulate across runs and across concurrent
jobs. The cache is keyed by company name and only read after a successful
file-exists check; concurrent writers to the same memo file are not a
real-world hazard.

---

## Step 1: Gather All Available Context

**Anchor the start time first** so the audit-start Slack alert (Step 4b) can
report elapsed-from-start minutes. Write this once, before any other work:

```bash
date +%s > "$WORKSPACE/start_ts.txt"
```

Before writing anything, assemble the full evidence base. Completeness matters — the analysis
quality is directly proportional to how much source material you have.

### 1a. Fetch the Notion Opportunity

Search for the company in the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`) using
`notion-search`. Fetch the full opportunity page with `notion-fetch`. Extract:

- All properties (stage, round details, status, description, scores, HQ, founders)
- Full page body content (summary, team bios, round context, source context)
- All linked Notes (the ✍️ Notes relation) — fetch each linked note page
- Diligence Materials property — note any Google Drive file IDs or URLs

### 1b. Fetch All Linked Notes

These are typically call notes, research threads, backchannel references. Fetch every URL in
the Notes relation. Pay close attention to:

- **Founder meeting / call notes** — the richest source of founder signal
- **Backchannel / reference notes** — third-party perspective on the team
- **Prior Claude analysis** — earlier research or framework mapping

**CRITICAL — `include_transcript: true` is mandatory on every call note fetch.** Every
`notion-fetch` call against a Notes-DB page in this step MUST pass `include_transcript: true`.
Without it, the meeting-notes widget returns only the AI summary block, and the `<transcript>`
section comes back as the placeholder "Transcript omitted." The raw transcript is the
highest-fidelity founder-signal input this skill has — silently working off the summary
contradicts this skill's "transcripts are primary" rule downstream (Step "Founder Profile"
explicitly cites specific transcript moments). If the page is not a meeting note (no
`<meeting-notes>` widget), the param is a harmless no-op. Default ON, no exceptions.

**MANDATORY — speaker-label every fetched transcript before drafting.** Notion's transcript
feature stores turns as raw paragraph blocks with NO speaker metadata (verified 2026-06-06
by walking the REST tree). This is a real problem: without labels, the drafter can't
reliably tell whether an analogy was raised by the founder or by Tom, and misattributing
Tom's reframings to the founder inflates the founder-evaluation signal (see
deviation #28 in Common Deviations and memory
`feedback_transcript_speaker_attribution_to_tom`). Run the relabel pass on every call
note transcript:

```bash
mkdir -p $WORKSPACE/raw_transcripts $WORKSPACE/labeled_transcripts

# 1. Extract the raw transcript text (one turn per line) into a per-call file.
#    Strip the <transcript>...</transcript> wrapper from the MCP fetch output.

# 2. Relabel each transcript via Haiku.
python3 ~/.claude/skills/first-pass-diligence/relabel_transcript.py \
    --transcript $WORKSPACE/raw_transcripts/<call_slug>.md \
    --attendees 'TOM (investor at Inverted Capital, asks investor-style questions and raises analogies);\
SONIA (founder + CEO of [Company], [healthcare/fintech/etc background], owns market positioning);\
MANISH (founder + CTO of [Company], ex-Google, owns engineering architecture)' \
    --subject '[meeting subject line — date, company, deal stage, who attended]' \
    --output $WORKSPACE/labeled_transcripts/<call_slug>.md
```

Cost: ~$0.03–$0.05 per 75KB transcript at Haiku 4.5. Latency: ~20–30 seconds serialized
per transcript (parallelize across transcripts to bound wall-clock at the slowest single
call). The labeled output is `[TOM]:` / `[SONIA]:` / `[MANISH]:` / `[UNKNOWN]:` prefixed
per turn. UNKNOWN is reserved for genuinely ambiguous backchannel turns ("yeah", "right")
where Haiku couldn't infer — these are non-blocking in the downstream attribution lint.

Build attendee descriptors with enough role context to let the labeler distinguish
voices from content cues (investor-vocabulary vs founder-vocabulary, engineering-deep
vs market-deep). The descriptors are the only knob the model has for disambiguation —
generic descriptors produce worse labels.

The labeled transcripts are consumed by `first_pass_lint.py --labeled-transcripts-dir`
in Step 4a to enforce deterministic attribution.

### 1c. Fetch Diligence Materials

**HARD RULE — every company-provided material is in scope, regardless of filename labels.**
Process every entry in the Diligence Materials property field. Do NOT skip files because the
filename or title contains `[DEPRECATED]`, `(deprecated)`, `OLD`, `[OLD]`, `superseded`, `v0`,
`archived`, `graveyard`, or any other founder-applied "this is no longer current" tag. Every
artifact the founders provide goes into the analysis. Founder-marked deprecation is itself
diligence signal — include the file with a `(deprecated)` qualifier inline, and treat the
contrast between deprecated and current as material for the analysis (what changed, what was
walked back, what the rename or pivot signals). Memory:
`feedback_diligence_materials_deprecated_not_skip`. Caught after AgentBay 2026-06-25.

**Classification rule.** All company-provided artifacts (decks, financial models, docs,
slides, transcripts, screenshots founders sent) belong in the Sources list as company-provided
materials — not as "Customer Feedback" or other reader-feedback categories. Tom-authored
framework docs (Diligence Q&A, question lists, pre-mortems) are Research & Analysis. Owner
email in the Drive metadata is the tiebreaker: company-domain owner = Materials; tom@
invertedcap.com owner = Research & Analysis.

The Diligence Materials field may contain five types of sources. Identify each by its URL
or file extension and use the corresponding access method:

**Type 1: Google Drive PDFs** (`drive.google.com/file/d/{FILE_ID}/...`)
Use the PDF MCP tool `read_pdf_bytes` with the download URL format:
`https://drive.google.com/uc?export=download&id={FILE_ID}`. This works without Chrome
authentication for files shared to Tom's account.

**Type 2: Google Docs** (`docs.google.com/document/d/{DOC_ID}/...`)
Use `google_drive_fetch` with the document ID.

**Type 3: Notion-hosted attachments** (`attachment:{UUID}:{filename}.pdf`)
These are PDFs uploaded directly to Notion and cannot be accessed via `read_pdf_bytes`
(requires Notion auth cookies). Use the Chrome-based extraction pipeline:

1. Navigate Chrome to the opportunity's Notion page (if not already there).
2. Execute JavaScript to call Notion's internal API and get a signed URL:
   ```javascript
   (async () => {
     const resp = await fetch('/api/v3/getSignedFileUrls', {
       method: 'POST',
       headers: { 'Content-Type': 'application/json' },
       body: JSON.stringify({
         urls: [{
           url: 'attachment:{UUID}:{filename}',
           permissionRecord: { table: 'block', id: '{PAGE_ID_WITH_DASHES}' }
         }]
       })
     });
     const data = await resp.json();
     window.location.href = data.signedUrls[0];
   })()
   ```
3. Once Chrome navigates to the `file.notion.so` PDF URL, load pdf.js from CDN and extract
   text content:
   ```javascript
   (async () => {
     const script = document.createElement('script');
     script.src = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js';
     document.head.appendChild(script);
     await new Promise(r => { script.onload = r; });
     pdfjsLib.GlobalWorkerOptions.workerSrc =
       'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
     const resp = await fetch(window.location.href);
     const buf = await resp.arrayBuffer();
     const pdf = await pdfjsLib.getDocument({ data: buf }).promise;
     let text = '';
     for (let i = 1; i <= pdf.numPages; i++) {
       const page = await pdf.getPage(i);
       const tc = await page.getTextContent();
       text += '\n--- Page ' + i + ' ---\n' + tc.items.map(x => x.str).join(' ');
     }
     return text;
   })()
   ```
4. Navigate Chrome back to the Notion page afterward if you need to process additional
   attachments.

**Important:** This Chrome-based pipeline requires Chrome access via the Claude in Chrome
MCP. If Chrome is unavailable (e.g., scheduled task without Chrome), note the Notion
attachment in the analysis as "material available but not readable in current environment"
and flag it as a gap in the Sources section.

**Type 4: DocSend links** (`docsend.com/view/...`)
Read the `docsend-to-pdf` skill (`**/docsend-to-pdf/SKILL.md`) and follow its instructions
to convert the DocSend document to a local PDF. Once converted, read the resulting PDF with
`read_pdf_bytes` using the local file path. DocSend data rooms (multi-document links) should
have each document converted individually.

**Type 5: Video files** (`.mp4`, `.mov`, `.m4v`, `.webm` — typically product demos, recorded
calls, founder pitches uploaded to Notion or Drive)

Download the file to `$WORKSPACE/demo_<original_name>.<ext>`, then transcribe via the
local Whisper helper:

```bash
python3 ~/.claude/scripts/transcribe_video.py $WORKSPACE/demo_<name>.<ext> --model small > $WORKSPACE/demo_<name>.txt
```

Use `--model small` as the default — better fidelity than `base` for diligence-grade
transcripts (founder names, product terminology, technical claims) without the latency
cost of `medium`/`large`. First run downloads the model (~500 MB for `small`); subsequent
runs reuse the cached weights.

Treat the resulting transcript as a **first-class diligence artifact** — fold it into the
source bundle (Step 4b) under `==== DILIGENCE MATERIALS ====` with a `--- Material:
<name> demo (transcribed) ---` header, and the lint/audit gates will check claims against
it the same way they check call transcripts. Specifically watch for product claims the
founder makes only in the demo (loop architecture, latency numbers, partnership names)
that don't appear in the deck or one-pager — these are common analytical gold and easy
to miss without the transcript.

**System deps:** `ffmpeg` (`brew install ffmpeg`) and `openai-whisper`
(`pip install openai-whisper --break-system-packages`). If either is missing, the script
exits 2 and prints the install command. Flag the gap in the analysis (Materials & Sources
Reviewed + Suggested Additional Analysis) rather than skipping silently.

**Type 6: URL-based video** (Loom — `loom.com/share/...` — and comparable hosted video
platforms: Wistia, Vidyard, etc.)

For Loom links, navigate Chrome to the URL. Loom renders a transcript panel alongside the
player — extract it via:

```javascript
document.querySelector('[data-testid="transcript-panel"]')?.innerText
// fallback if selector changes:
Array.from(document.querySelectorAll('[class*="transcript"]')).map(el => el.innerText).join('\n')
```

If the transcript panel is absent (some Loom plans don't generate transcripts), look for a
Download button to save the `.mp4` and fall back to the Type 5 Whisper transcription
pipeline. For other hosted platforms (Wistia, Vidyard), attempt the same transcript
extraction first; if unavailable, check whether the embed page exposes a download link,
then fall back to Whisper.

Treat the resulting transcript as a first-class diligence artifact — fold it into the
source bundle under `==== DILIGENCE MATERIALS ====` with a `--- Material: <name> (Loom
transcript) ---` header, and cite it in the Context section footnotes and terminal Sources
section like any other diligence material.

### 1d. Gather Founder-Specific Evidence

For each founder named on the Opportunity (extract from the Opportunity page properties and the
team section in the page body), assemble a per-founder evidence packet that feeds Section 5's
Founder Evaluation. The same evidence is also useful elsewhere in the analysis (Founder-Market
Fit, Team Dynamics & Composition), but Section 5 is the primary consumer.

- **Raw LinkedIn profile scrape.** For each founder's LinkedIn URL, navigate Chrome to the
  profile and extract `document.body.innerText` (LinkedIn renders client-side — do NOT
  `fetch()`; the HTML response is a shell). Capture the full About section, every Experience
  entry with descriptions, Education, recommendations, posts, and articles. Raw scrape is
  preferred over a structured ContactOut pull because how the founder phrases their own work
  is itself signal — nuance gets compressed in structured fields.
- **Founder bio / profile artifacts from Diligence Materials.** Cross-reference 1c — flag any
  deck slides, one-pagers, memos, or DocSend pages that contain a founder bio or photo as
  inputs to Section 5.
- **Linked Notes that touch on founders.** Cross-reference 1b — call transcripts and Tom's
  notes from those calls are the highest-fidelity input. Title patterns + body content
  disambiguate raw transcripts (Granola/Zoom auto-summaries, "Call with X") from Tom's
  synthesized notes; both are valid inputs.

If a founder's LinkedIn URL is not on the Opportunity, attempt to find it via web search
(`"<founder name>" "<company>" site:linkedin.com/in`). If still not findable, note the gap
explicitly in Section 5 — the evaluation runs with whatever evidence is available.

### 1e. Read Tom's Investment Framework Memos

List all files in the Google Drive investment memos folder (`1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`)
using `google_drive_search` or by listing the folder contents. Then read **every memo** in the
folder using the appropriate access method (Google Drive fetch for Docs, `read_pdf_bytes` with
the download URL format `https://drive.google.com/uc?export=download&id={FILE_ID}` for PDFs).

**Why read all memos, not a subset:** The frameworks Tom applies evolve as the portfolio grows.
New memos may introduce new frameworks, refine existing ones, or reveal cross-portfolio patterns
that only emerge across the full body of work. Reading every memo ensures the analysis reflects
the complete and current Inverted Lens vocabulary — not a stale snapshot. This is the mechanism
by which the Inverted Lens stays current as Tom writes new memos.

From the memos, extract the recurring investment frameworks Tom has applied across the portfolio.
These are the analytical lenses he uses to evaluate founder quality, business model durability,
and structural advantages. Typical framework themes include (but are not limited to): data assets
as drivers of enduring value, forward-deployed founder orientation, intentionality and intellectual
honesty, unbounded end states with focused wedges, structural distribution leverage, capital
efficiency, founder slope over y-intercept, domain expertise without arrogance, and non-transactional
relationship-building style. But **derive the actual framework menu from the memos themselves** —
do not rely on this illustrative list as a substitute for reading the source material. The memos
are the source of truth.

Select the frameworks most relevant to the specific opportunity — not all will apply with equal
force to every deal. The selection should feel natural and evidence-driven, not forced. Evaluate
each selected framework against the opportunity, with attention to whether the founders are
proactively surfacing these themes (unprompted) vs. responding to investor questioning.

**Memo manifest — capture and use as the authoritative source set.** As you read each memo,
build an explicit manifest: `{company name → Drive file URL → frameworks extracted}`. This
manifest is the **only** set of companies you may later cite as "the [X] memo" or quote from
as memo content. Any portfolio company NOT in this manifest has no memo readable for this run
— do not invent one, do not infer one from the company's Opportunity page body, and do not
upgrade decision-retro snippets or call-note prose into "memo" framing. Carry the manifest
forward into Step 3; the Reference Documents section in Step 5 must reconcile against it.

**Memo text cache — MANDATORY, enables the Step 4a analog-grounding lint check.** As you
read each memo's full text, ALSO write it to disk so the lint can verify analog
characterizations against the actual memo prose. Cache directory: `/tmp/firstpass_memos/`.
Filename convention: `<Company Name>.md` exactly as the company appears in the manifest
(spaces preserved). Example: `/tmp/firstpass_memos/Oun Homes.md`. Include the full memo
text — body prose, framework sections, founder-evaluation prose. Frontmatter optional.

```bash
mkdir -p /tmp/firstpass_memos
# For each memo fetched in this step:
#   cat > "/tmp/firstpass_memos/<Company Name>.md" <<'EOF'
#   <full memo text>
#   EOF
```

Why this matters: without the cache, Step 4a's `find_ungrounded_analog_overlays` LLM
check is skipped with a warning (Kalos first-pass 2026-05-28 shipped with the Oun-as-
physical-operating-surface hallucination because no such check existed). With the cache,
the lint runs an LLM judge per candidate analog overlay sentence against the analog's
actual memo and fails the gate if the overlay is ungrounded. Caching adds ~1 second per
memo, negligible against the rest of Step 1e's compute. **Skipping the cache means the
analog-grounding gate is disabled for this run — call out explicitly in the publish
summary if you skip it.**

**Manifest coverage gate — MANDATORY before proceeding to Step 1f.** Run
`verify_memo_manifest.py` to assert that `$WORKSPACE/manifest.json`'s
`memo_manifest` lists every memo currently in the Drive folder. Exit code 3 =
missing memos; the script names them so you can re-read them and re-write the
manifest. Do not proceed to Step 1f / Step 2 with a failing gate.

```bash
python3 ~/.claude/skills/first-pass-diligence/verify_memo_manifest.py \
  --manifest "$WORKSPACE/manifest.json" \
  --folder-id 1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO
```

Why: the model has historically short-circuited Step 1e and only read a subset
of memos (AgentBay first-pass 2026-06-24 missed 3 of 6 memos — Rengo, Oun
Homes, Signal7 — silently shipping an analysis built on a partial framework
inventory). Manual prose discipline ("read every memo") doesn't enforce
itself; the gate does, by listing the Drive folder via service account and
diffing against the manifest.

### 1f. Load Feedback Patterns

Read `/Users/tomseo/.claude/skills/first-pass-diligence/FEEDBACK_PATTERNS.md`. This file
accumulates Tom's Slack-thread feedback on prior published first-passes — soft taste
corrections that aren't absolute rules but should shape draft choices ("the Tuor analog
tends to be forced for B2B fintech", "Section 4 prose was too dense in Shine", "regulatory
framing should be cut when sector isn't actually regulated"). Treat each entry as a
context signal, not an override: if a pattern conflicts with this deal's specific evidence
base, the evidence wins — but flag the conflict in the publish summary so Tom knows you
saw the pattern and chose to deviate.

If the file is empty or only contains the header comment, that's fine — the corpus
builds over time. No action needed beyond reading.

---

## Step 2: Conduct Web Research

Before drafting, run targeted web searches to enrich the analysis with independent data points.
The goal is to verify founder claims, size markets, and ground open questions in real numbers
rather than leaving them as generic unknowns. Typical research areas:

- **Market sizing** — TAM/SAM data, growth rates, comparable market benchmarks
- **Regulatory environment** — licensing timelines, requirements, costs (compare to founder
  representations)
- **Competitive landscape** — comparable company metrics, loss rates, unit economics where
  public
- **Customer population** — sizing the target segment with demographic and economic data
- **Technology landscape** — infrastructure maturity, data availability, API ecosystem
- **Founder online presence** — for each founder named on the Opportunity, research public
  artifacts: personal website, blog, podcasts/talks, conference appearances, public posts on
  X/LinkedIn, and any artifacts associated with companies they were at (engineering blogs,
  product launches, internal-team interviews, talks given while there). Mirrors the Phase 2
  online research from `neg1-enricher` Step 4.5. Capture URLs and the highest-signal
  pull-quotes — these feed Section 5's Founder Evaluation. If a founder has no discoverable
  online presence after a deeper search pass, note the absence — it is itself signal for
  Intellectual Rigor scoring. Also check whether the founder is **associated with the
  company on LinkedIn** (listed as a member on the company page, has the company in their
  Experience block). **Absence of public presence is NOT inherently a flag** — many strong
  founders are private, and the framework explicitly does not downgrade Intellectual Rigor
  for thin public surfacing. Absence becomes a Conflict Callout *only* when the founder's
  role or function arguably requires public visibility: CEO during an active fundraise
  (warm-intro reach, investor diligence, narrative-building), GTM/sales lead (rolodex
  visibility), marketing or comms function, public-facing community operator. For roles
  where public presence is incidental (technical IC, back-office finance, ops, security,
  legal), absence is neutral — drop the callout. The question to ask is "does this role
  plausibly require public visibility?" — surface as Conflict Callout only if yes.

When research findings contradict founder claims, flag this explicitly. The gap between what
the founder says and what independent research shows is one of the most valuable outputs of
this analysis.

Every data point cited in the final output must be traceable to a source. Collect URLs as you
research — they will populate the Sources section.

---

## Step 3: Draft the Analysis

### MANDATORY — drafting runs on Opus tier

Per memory `feedback_model_tier_framework`: Opus = no-rubric + final-artifact tasks.
The first-pass analysis is a long-form synthesis artifact with no scoring rubric —
Opus territory. The Layer-2 audit gate in Step 4b is Sonnet (rubric-based review); the
drafting itself must be Opus.

- If you're in an interactive Opus session, proceed inline.
- If you delegate to a subagent for context-budget reasons, pass `model: "opus"` to the
  Agent tool explicitly (`subagent_type: "general-purpose"` inherits parent model — if
  the parent isn't Opus, the subagent degrades silently).
- If you're stuck in a Sonnet session: tell Tom, ask him to re-invoke from an Opus
  session. Don't silently ship Sonnet-quality first-pass content.

Write the analysis in a professional, analytical voice. The tone should be factual and objective —
not promotional, not dismissive. Each section should contain substantive paragraphs (not one-liners)
that convey ideas with high fidelity. When you lack sufficient context to make an informed assessment
on a topic, say so explicitly rather than filling space with generic language.

### Section Order

**MANDATORY first line of body:** the inner H1 anchor

```
# First-Pass Diligence — <Long Date>
```

(em dash, matches `pdf_header_check.py` P4 default pattern). This anchor is the
structural divider that lets `update-diligence-priors` prepend Update sections
above the first-pass content, and that `finalize-diligence` uses to find the start
of the first-pass section. The PDF renderer uses it as a page-break trigger. Skipping
it ships a doc that future updates cannot cleanly slot into. Concrete prior incident:
Kalos first-pass 2026-05-28 shipped without it; Tom flagged it during review.

Do NOT lead the body with a date stamp (`*April 27, 2026*` etc.). The Notion page title
already carries the date (`[Claude] [Company Name] Master Diligence Doc — MM.DD.YYYY`),
and the PDF subtitle does too. A leading body date is redundant and reads as a small
production error. After the inner H1 anchor, start the body directly with the **Context**
section, then proceed into Framework Mapping. Context has three subheaders — rendered as **bold text only, not
underlined headings and not H-level headings**: Company Overview, Working Thesis, and
Materials & Sources Reviewed.

**Citation style — document-wide:** Use footnote numbers in brackets — `[1]`, `[2]`, etc. —
rather than inline parenthetical references **throughout the entire document**, including
Framework Mapping, Founder Evaluation, Market Context, and every other section. Never write
"(per the deck)", "(per the May 13 call)", "(per the unit economics primer)", or any similar
parenthetical anywhere in the output. At the end of the Context section (immediately before
Framework Mapping), include a compact footnote block where each entry uses `^N` at the start
of the line (caret + number, no brackets, no colon):

> The deck positions the company as a data layer for property managers [1], and the March 19
> call elaborates on the regulatory strategy [2].
>
> ^1 Company Name — Investor Deck (March 2026)
> ^2 Reed & Nolan Intro Call — March 19, 2026

**Multi-source citations — single bracket, no space after comma:** When a sentence cites
multiple sources, group them inside ONE pair of brackets with `,` (no space) between
numbers — `[1,2]`, `[1,2,3]`, `[2,5,7]`. Do NOT write `[1] [2]` (adjacent brackets),
`[1][2]` (no space), `[1, 2]` (space after comma), or `[1-3]` (range form). Notion's
renderer squashes the space between adjacent brackets, so `[1] [2]` reads as `12` in the
published page and PDF; the unified `[1,2]` form keeps the superscript tight in the PDF
and is unambiguous across Notion, the PDF builder, and grep. The lint check
`adjacent_citation_brackets` blocks `] [` patterns between citation markers and any
`[N, M]` form with a space after the comma.

**Why `^N` not `[^N]:`:** Notion interprets `[^N]:` as a reference-link definition and
mangles it into `[\[N\]](N):` on save. `^N` at the start of a line is inert to Notion's
parser and survives the round-trip intact. Inline `[N]` markers are stored by Notion as
`\[N\]`; the PDF builder's artifact cleanup converts them back to `[N]` before applying
superscript rendering. Both formats are handled automatically — just write them as shown.

Sources that are obvious from context may be cited without a number — reserve footnote
markers for cases where the specific source artifact matters.

The analysis follows this exact structure:

**Context**

**Company Overview**
A factual 1–2 paragraph description of the company: what the product does, who it
serves, business model basics, stage, and the current round context. Draws on the
full Notion Opportunity record — page body, properties (Description, Founder
Description, Round Details, HQ, stage), all linked Notes (call notes, intro
context), and Diligence Materials (deck, one-pager, memo, data room contents).
The reader should finish this section knowing what the business is, who it's for,
and where it sits in its lifecycle — before any analytical framing kicks in.

This section is **descriptive, not evaluative**. No thesis claims, no framework
references, no "we like this because…" language. If the materials disagree on a
basic fact (e.g., deck says X customers, one-pager says Y), surface the discrepancy
factually here rather than resolving it.

**Working Thesis**
The founder's view of what this company is betting on.

If the founder has articulated their own thesis in any captured source — a call
(transcript or Tom's notes), the deck, a one-pager, an investor update, a written
memo — capture it verbatim or in close paraphrase, and cite the source inline
("From the March 19 intro call:…" / "The deck frames the thesis as:…").

If no explicit founder thesis is present in any captured source, **synthesize an
implicit thesis from the materials and call transcripts** — state, in 3–5 sentences,
what the cumulative evidence suggests the company is betting on. You have enough source
material to synthesize — do NOT punt with "thesis unclear" if call transcripts and
materials are in hand. Every claim in a synthesized thesis must be traceable to a source
via the Context footnote system — the synthesis is auditable or it doesn't ship.

**Stay in the founder's frame.** This section is the founder's bet, not Inverted's
read on it. Do NOT layer in framework references, decision retros, memo analogs,
or "this maps to our [X] framework" framing — that is the explicit job of Framework
Mapping below. Everything that follows develops, stress-tests, or qualifies this
working thesis.

**Materials & Sources Reviewed**
An upfront inventory of what was scanned to produce this analysis, organized by
type. This is distinct from the terminal **Sources** section at the end of the
document (which is the clickable audit trail with full URLs and annotations); this
upfront block gives the reader at-a-glance scope of what informed the doc before
they read the analysis.

Format as a bulleted list grouped by source type — one line per entry, terse:
- **Notion Opportunity** — page name (no URL; the terminal Sources section carries it)
- **Call notes** — title + date for each note in the ✍️ Notes relation
- **Diligence Materials** — name each: deck, one-pager, memo, data room contents
- **Founder evidence** — LinkedIn profiles scraped, online presence sources
  consulted (no URLs here)
- **Inverted memos referenced** — list the investment memos actually drawn on in this analysis
- **External research** — high-level categories surveyed (competitive landscape,
  market sizing, regulatory environment) — full URLs live in the terminal
  Sources section

Keep entries terse — this is an inventory, not the audit trail. If a category
was not consulted (e.g., no diligence materials yet), omit the row rather than
writing "None" — the absence is itself informative.

**Framework Mapping — Inverted Lens**
Evaluate the opportunity against the Inverted Lens frameworks that are most relevant to this
specific deal. The full menu is documented in Step 1e — select whichever apply with force
based on the evidence, not a fixed set.

Open with a brief preamble paragraph (2-4 sentences) explaining which frameworks were selected
and why, noting that the evaluation considers both structural fit and whether the founders are
proactively surfacing these themes unprompted. This orients the reader before the subsections.

For each selected framework, write a subsection with a descriptive header (e.g., "The Data Asset
as the Driver of Enduring Value" — not numbered, not abbreviated). Each framework subsection
must contain **multiple substantive paragraphs** (3-5 paragraphs minimum) that:
- Develop the analytical argument with evidence from the diligence record
- Cross-reference how the framework has been applied in other Inverted portfolio memos — but
  **only at the framework-label level** unless quoting verbatim from a manifest memo (see
  "Analog claims — verbatim-or-label rule" below). Acceptable: "Rengo's memo anchors the
  data-asset compounding framework in PE-firm portfolio monitoring data." Unacceptable
  without a quoted manifest source: any biographical or operating-history claim about an
  analog founder ("X spent years doing Y", "X's anchor was Z", "X built the company by W").
- Surface tensions, caveats, and honest assessments of where the mapping is strong vs. weak
- Note whether the founder is proactively raising the theme vs. responding to investor questioning

**Analog claims — verbatim-or-label rule.** When invoking a portfolio company as an analog,
you have two and only two options:

1. **Framework-label reference (no quote required):** State the framework the analog company
   maps to, using only the company name and a high-level description of HOW the framework
   applies, sourced from that company's `Description` or memo first paragraph. Example:
   "Rengo applies the data-asset compounding framework — structured portfolio monitoring data
   accruing per PE firm." No founder names, no biographical detail, no operating-history
   claims, no quotes.
2. **Verbatim-quote reference (memo manifest entry required):** Quote a specific passage from
   the analog's memo (which must be in the Step 1e manifest) and attribute it explicitly:
   "The Tuor memo writes: '<verbatim quote>'." Then connect the quoted passage to the
   subject company's situation. Quotes must be exact; do not paraphrase and frame as a quote.

**Anything in between is forbidden.** Specifically, do NOT write any of the following without
a verbatim manifest-memo quote backing it up:

- Founder biographical detail for the analog company ("X spent a decade in finance", "X built
  in vertical Y for years", "X's prior failure was Z")
- Operating-history framing for the analog company ("X lived inside the workflow", "X earned
  the right to build by doing Y")
- Behavioral or stylistic characterization of the analog founder ("X is a forward-deployed
  operator", "X's edge is unprompted structured thinking")
- Implied causal stories about why the analog succeeded ("X's bet was that A would lead to B")
- **Architectural overlays applied to the analog** ("Oun Homes owns the physical operating
  surface", "Suppli applies the vertical-B2B-with-physical-anchor framework", "Rengo
  monetizes the digital coordination layer"). Architectural labels — anything attaching
  the analog to a structural pattern like physical-vs-digital, marketplace-vs-SaaS,
  hardware-vs-API, distribution-vs-product — MUST trace verbatim or near-verbatim to the
  analog's own memo / Opp Description / Founder Description. **Never pattern-match the
  subject company's architecture onto an analog whose actual business model doesn't share
  it.** Concrete prior incident: Kalos first-pass (2026-05-28) labeled Oun Homes as owning
  the "physical operating surface" because Kalos has physical scan locations — but Oun is
  a pure-software buyer-led operating system, not a physical-asset business. The doc
  then contradicted itself within a few paragraphs by acknowledging Oun is software-only.
  Internal contradictions like this are the smoking gun for framework hallucination.

These are the specific failure modes the Kestrel first-pass (2026-05-13) exhibited — none of
the analog biographical detail there was traceable to a source, and several claims were
fabricated to fit the analogy. When in doubt, drop to the framework-label form. Better to say
less, accurately, than more, imaginatively.

**Grounding portfolio references — MANDATORY guardrail.** When citing companies as Inverted
portfolio examples ("Just as Rengo…", "The parallel to the Inverted portfolio…", etc.), the
named companies MUST come from the authoritative portfolio set. Compute that set by querying
the Opportunities DB for `Status IN ("Active Portfolio", "Exited", "Committed")`. Note that
`Portfolio: Follow-On` rows are a SUBSET of those companies — every FO entry's underlying
company already appears under its `Active Portfolio` or `Exited` row, so do NOT add FO
companies as separate names; dedup by company.

Companies that surface in `DECISION_RETROS.md`, pass-note archives, the Notes DB, or anywhere
else that captures Tom's analytical commentary are NOT portfolio. They may be referenced — but
phrase them as "a deal Tom analyzed", "a comparable in the pipeline", or similar — never as
"Inverted portfolio". Specific historical cases that have been miscategorized as portfolio
include Clusia and Third Space (Tom analyzed and gave feedback on both; neither is portfolio).
If you find yourself about to name a company you haven't verified against the live Opportunities
DB query, stop and re-verify.

**Memo citation traceability — MANDATORY guardrail.** Every memo citation in the analysis must
trace to the Step 1e memo manifest. Concretely:

- Before writing "the [X] memo", "the original Inverted memo on X", or any quoted phrase
  attributed to a memo, look up X in the manifest. If X is not there, **you did not read its
  memo and it does not exist for this run.** Drop the citation, or reframe without memo
  attribution (e.g., "Suppli, also in the portfolio, took a similar approach" — no quote, no
  "memo says").
- **`DECISION_RETROS.md` is not a memo source.** Phrases, framework ideas, or characterizations
  pulled from retro nuggets are decision-retro content. Never quote a retro line as if it
  came from a memo, and never group retro-sourced citations under a "memo set" framing
  ("frameworks derived from the memo set covering A, B, C, D…" must list ONLY manifest
  entries). If a retro nugget is genuinely useful, cite it explicitly as "Tom's decision
  retro on [company]" — not as a memo.
- Portfolio status alone does NOT imply a memo exists. Suppli and MakersHub are portfolio
  companies; neither has a memo in the Drive folder (as of 2026-05-14). Do not infer the
  existence of a memo from Active Portfolio / Committed status.
- Before submitting Step 5, scan the draft for every occurrence of "memo" referencing a
  named company and verify each named company appears in the manifest. Drop the rest.

**Portfolio facts — analog truthfulness.** Naming a real portfolio company is necessary but not
sufficient. Every characterization of what that company does, sells, buys, or has done as a
business ("Quiet's wedge into AI-native compliance", "Rengo compounds a data asset for PE firms",
"Hardik previously failed and rebuilt a company") must come from a primary source on that
company — its memo, investor updates, call notes, or the Opportunity row's Description / Founder
Description. If the analogy requires a fact about the analog company you can't locate in those
sources, **drop the analog rather than invent the fact**. Forced parallels with fabricated detail
are worse than no parallel — they degrade Tom's trust in every other claim in the doc. Before
submitting, scan every sentence containing a portfolio-company name and ask: "Did I read this
fact, or did I shape it to fit the analog?" If the answer is the latter, cut it. The same rule
applies to characterizations of named individuals (founders, references, executives) — every
biographical or behavioral claim must trace to a source, not be confected to fit the analogy.

**Founder names — exact-match verification before writing.** Any time you write a founder's
name in the analysis (including analog companies in the Framework Mapping section), verify the
**exact spelling of both first and last name** against a primary source for THAT company —
the Opportunity row's `🏁 Founder(s)` relation, the Opp `Contact` email (e.g.,
`rayers@gosuppli.com` → "Ayers"), or the company memo if one is in the manifest. **Do not
write a founder name from memory or pattern-match against common surnames.** Concrete prior
incident: Kestrel first-pass (2026-05-13) named Suppli's founder as "Ryan Walsh" when the
Opp `Contact` field plainly reads `rayers@gosuppli.com` (Ryan Ayers). Pre-submit, scan every
proper noun naming a founder and confirm it traces to a primary source. If you cannot verify
the surname, use first-name-only ("Ryan at Suppli") rather than guessing.

**Analog-company fact pre-fetch — MANDATORY before writing Framework Mapping.** Before composing
any analog passage that names a portfolio company AND attributes biographical / operating-history
detail to a founder there ("X's anchor was Y", "X spent years doing Z", "X built the company by
doing W"), fetch that company's Opportunity row and read **at minimum** the `Name`,
`Description`, `Founder Description`, `Contact`, and `🏁 Founder(s)` properties. If the analog
also has a memo in the Step 1e manifest, read it. **Do not write analog-company biographical
detail from prior conversation, training data, or pattern-match against the company name.**
Concrete prior incident: Kestrel first-pass (2026-05-13) wrote "Oun Homes, where Robert's anchor
was years of physical builds" — the Oun Homes Opp `Contact` is `santi@oun.homes; pj@oun.homes`
(founders Santi and PJ, not Robert), and the Opp `Description` is "Buyer-led operating system
for the entire home buying journey" (a software OS, not physical building). Both the name and
the operating-history framing were confected to fit an analogy. If you cannot locate the
specific biographical fact you want to claim in the Opp properties or a manifest memo, **cut
the analog** — do not invent the fact, and do not weaken to a vaguer version of the same
fabrication ("operator orientation", "lived in the space" without grounding). A good analog
requires a real, sourced parallel; a bad analog masquerading as a real one is worse than no
analog at all.

**Third-party company truthfulness (non-portfolio).** The same source-or-drop rule applies to any
named third-party company used as a comp, GTM analog, or ecosystem reference (e.g., "Oun answers
through the property management software ecosystem", "Signal7 operates in a regulated-adjacent
space"). If a specific characterization of what a non-portfolio company does, how it routes GTM,
or which category it sits in can't be traced to the provided materials, write only what is
verifiable — "operates in [space]" or "builds in [category]" — rather than inventing product or
go-to-market specifics. Do NOT extrapolate product architecture or distribution mechanics for
companies whose memos or materials were not provided.

**Regulatory framing — sector must warrant it.** Only characterize a business as operating in a
regulated space, facing a regulated-partner requirement, or requiring a licensed counterparty if
(a) the diligence materials or call notes use that language, or (b) the business clearly operates
in a named regulated sector (financial services, healthcare, cannabis, etc.). Do NOT apply
regulatory framing to adjacent-infrastructure plays or non-regulated verticals (home services,
construction, logistics) even when the subject company operates nearby. If you're uncertain
whether a sector is regulated, use neutral language or flag it as an open question.

**Founder history must trace to source.** For any biographical or career claim about a named
individual — especially claims that imply failure, pivots, or prior company founding — trace
the claim to a primary source before writing it. If the Opportunity memo or Tuor memo says a
founder is "first-time" or has no prior company, that overrides any inference you would draw from
their operator arc. Do NOT use "prior failures and rebuilds" or similar framing unless a source
uses that language. If the source says "multi-hop operator arc," describe the arc — don't
editorialize about what it implies.

**NEVER** use scorecard-style assessment labels like `**Assessment: STRONG FIT**` or
`**Assessment: MODERATE-TO-STRONG**`. These reduce nuanced analysis to a rubric grade and strip
out the reasoning that makes the section valuable. The assessment should emerge from the prose,
not be stamped at the top of each subsection.

End the full section with a **Summary Assessment** subsection (2 paragraphs minimum) that weighs
the frameworks against each other, identifies where the mapping is strongest and weakest, and
surfaces the most important signal across all frameworks taken together.

**Need to Believe**
Working backwards from what must be true for this to be a compelling Inverted Capital investment.
Each item is a **bullet** (not numbered) with a bold label and a multi-sentence explanation. These
are necessary conditions — if any breaks, the thesis is materially impaired.

**NEVER use numbered lists** (1., 2., 3.) for Need to Believe items. Use bullet points only.

End with a synthesis paragraph identifying the highest-priority beliefs to diligence — which ones
have the most binary outcomes and the least existing evidence.

**1. Market**
Three subsections: Market Size & Dynamics, Competitive Landscape, Regulatory Environment.

Market Size & Dynamics should use **flowing prose with data woven in naturally** — not a bullet-
point dump of statistics. Data points should be in service of an argument, not presented as a
standalone list. A dedicated "Market Data Points" bullet list at the end of the subsection is
acceptable for reference, but the analytical prose must come first.

Competitive Landscape should analyze competitive dynamics through **prose paragraphs**, not
capability/checkmark matrices. NEVER use comparison tables with ✓/✗ or ✓✓ symbols across
capability columns — these strip out the nuance of competitive positioning. Instead, explain in
prose why specific competitors are or aren't positioned to compete, using the founder's own
reasoning from the calls where relevant. A data table for competitor funding/stage/metrics is
fine; a feature-comparison grid is not.

Regulatory Environment should compare founder representations against independent research and
flag any gaps explicitly, using a **bold "Independent Finding"** callout format when research
contradicts the founder's timeline, cost, or complexity estimates.

Each subsection ends with ***Open Questions*** in bold-italic.

**2. Product**

§2 has seven subsections. The first two pair the company's stated view of
the build (Product Anatomy & Roadmap, Core Technical Strategy) with the
deeper teardown lenses that follow (Delivery Mechanism, Build Cost & Time
to v1, Path to Production-Grade, Moat Read,
Killshots). Each subsection has a single clear job; do not collapse,
merge, or drop any. Evidence-thin subsections run short — a single
honest paragraph naming the gap is better than three paragraphs of
training-data extrapolation. **Structure stays full; depth scales with
evidence.**

**Read the canonical framework spec** at
`/Users/tomseo/.claude/skills/shared-references/product-build-teardown-framework.md`
once at the start of this section. It owns the spec for §2.3–§2.7 below
(structure, depth requirements, table formats, citation discipline,
cost-calibration cites, killshot taxonomy). The same file is read by the
standalone `product-build-teardown` skill and `update-diligence-priors`
Product refreshes — single source of truth, no drift across invocation
paths.

**Read the cost-calibration reference** at
`/Users/tomseo/.claude/skills/shared-references/product-build-cost-calibration.md`
once before drafting §2.4 / §2.5. Every cost, time, FTE-month, infra
unit cost, or third-party API integration cost cited must trace to a
specific row formatted as `(per calibration §<N>)` inline.

**Crawl prep — before drafting any subsection.** Apply the
public-surface, engineering-signal, peer-surface, and broad-based-web-
research crawls defined in the framework spec ("Public surface area
crawl", "Engineering signal crawl", "Competitor / peer surfaces",
"Broad-based web research" subsections). Save screenshots to
`$WORKSPACE/product_screens_<n>.png`. Fold all extracted signal
into the Step 4b source bundle under the `==== PUBLIC PRODUCT SURFACE
====`, `==== ENGINEERING SIGNAL ====`, `==== COMPETITOR / PEER SURFACES
====`, and `==== EXTERNAL RESEARCH ====` blocks so the audit can
verify every claim.

***2.1 Product Anatomy & Roadmap.*** Apply framework spec §1 for
Product Anatomy — three layers: Core Components (flowing prose + bullet
inventory, lead with the primary loop), Data Flows, and Integrations &
Data Sources (table with Source / Purpose / Access Tier / Build
Complexity columns). Then a short **Roadmap** sub-block (one paragraph)
covering the founder's stated direction over the next 6–18 months —
what's being built next and how it changes the anatomy above (does it
add components, integrations, delivery surfaces?). The anatomy is the
mechanical inventory; the roadmap is the forward-looking trajectory.
End with ***Open Questions***.

***2.2 Core Technical Strategy.*** Adapt the header to the company's
actual technical core: **Underwriting Strategy** for a credit business;
**Data Ingestion & Quality Strategy** for an AI/data product;
**Matching Algorithm & Liquidity Strategy** for a marketplace;
**Hardware-Firmware Strategy** for a physical-product business; etc.
This subsection covers the **vertical-specific strategic bet that
defines whether the business can be built at all** — distinct from
§2.1 (which inventories components generically) and §2.3 (delivery
mechanism). The question this subsection answers: *WHY is this the
right technical approach for this specific business, and what would
falsify the bet?* End with ***Open Questions***.

***2.3 Delivery Mechanism.*** Apply framework spec §2. Name the primary
delivery archetype (standalone web app, SDK, API-only, browser
extension, native mobile, marketplace listing, embedded inside another
platform, async agent, data feed, hardware-adjacent), call out
secondary surfaces, assess distribution implications. End with
***Open Questions***.

***2.4 Build Cost & Time to v1.*** Apply framework spec §3. Both
dollars and calendar time get explicit treatment — neither one is the
"primary" axis. Engineering Scope (FTE-month range citing calibration
§6 archetype), Team Composition (citing calibration §1 rates),
**Calendar Time to v1** (range in months from kickoff, with friction
adjustments named), Infra & API Run-Rate table, Total v1 Capital
Required, Assumptions paragraph. When the company has disclosed actual
costs or team size or timeline, surface any material disagreement with
the calibration as an analytical finding. End with ***Open Questions***.

***2.5 Path to Production-Grade.*** Apply framework spec §4. Data
quality maintenance, operational reliability, scale stress points,
customer-facing edge cases, security & compliance gates (calibration
§5), support tooling. End with a Production-Hardening Cost & Time
estimate (calibration §7 — typically a doubling of cumulative
engineering investment, with calendar timeline implications called
out). End with ***Open Questions***.

***2.6 Moat Read.*** Apply the Moat Read section of the canonical
framework spec. Survey the moat sources in prose (unique data,
self-improvement / data compounding loop, integration depth /
switching costs, distribution lock, regulatory barrier, technical
complexity / scarce expertise, network effects), and **score each
lens High / Medium / Low** with the rating leading the prose
(`**Unique data — Low.** [explanation]`). Technical complexity /
scarce expertise IS the technical-differentiation lens — no separate
sub-header. Close with an honest counter-case paragraph. Be
opinionated — well-rounded mediocrity on moat is a negative read,
not a wash.

***2.7 Killshots.*** Apply framework spec §6. Each killshot is a
`#### Killshot N — [Descriptive Title]` H4 sub-subsection (per
`shared-references/label-hierarchy.md` — the H4 anchor keeps killshot
children visually distinct from §2.7's inline bold-italic header and
from the body-size `**Killshot N — Title.**` paragraph leaders that
would otherwise collapse against each other). Each H4 is followed by
an analytical paragraph and a bold-italic ***Failure mode:*** paragraph.
End with a Killshot Summary table (Failure Mode | Key Evidence | Kill
Shot?). When a killshot here is the same mechanism that will appear
in §6 Risks, write it in full here (the product-architecture-anchored
framing is the right home) and cross-reference from §6 Risks rather
than re-narrating.

End the full Product section with ***Open Questions*** in bold-italic
— cross-cutting questions that span multiple subsections and that the
founder is best positioned to answer.

**3. Go-to-Market & Distribution**
Three subsections: Initial Customer Acquisition (is there an "unfair" advantage?), Steady-State
Scalable Distribution, Where Do Customers "Exist" Today? End each with Open Questions.

**4. Operating Model**
Three subsections: Pricing Model Hypotheses, Capital Efficiency, Long-Term Steady-State Unit
Economics. End each with Open Questions.

Pricing Model Hypotheses should be grounded in **evidence from the diligence record** — actual
pricing mentioned in the one-pager, deck, or calls, comparable pricing from competitors found in
research, and the specific revenue mechanics of the company's product. NEVER present speculative
hypothetical pricing tiers (e.g., "Hypothesis 1: Per-Volume", "Hypothesis 2: Per-Seat") that are
not traceable to any source material. If the company hasn't disclosed pricing, say so, then
analyze what comparable companies charge and what the product's value proposition implies about
willingness-to-pay.

For the **Capital Efficiency** subsection, the central question is: *can this business operate
capital efficiently at inception — and thus justify a true pre-seed round — or is there something
structural about the business that requires a large round out of the gate?* Factors that could
necessitate a larger initial raise include: building in a consensus or well-funded category where
mindshare must be established early; a product or customer profile where counterparties (enterprise
buyers, regulated partners, institutional clients) want to see material capital on the balance
sheet before engaging; infrastructure or licensing costs that cannot be deferred; or a GTM motion
that is inherently capital-intensive from day one (e.g., sales-led into large enterprises, physical
operations, regulated product deployment). Assess whether the business as scoped is genuinely
compatible with a lean initial raise, or whether the pre-seed framing understates what it will
actually take to get to a meaningful proof point.

Long-Term Steady-State Unit Economics should model what the business looks like at maturity —
cost of capital, loss rates, operating costs, revenue per customer — and identify the key unknowns
that make the model speculative at this stage. NEVER fabricate burn rate analyses with monthly
figures that aren't traceable to the diligence record. If the company hasn't disclosed burn rate
or team size, note the gap rather than inventing numbers.

**5. Team**
Three subsections: Founder-Market Fit, Team Dynamics & Composition, Founder Evaluation Through the
Inverted Lens. Include a team experience table.

The **Founder Evaluation Through the Inverted Lens** subsection is required — do not skip it.
This is the structured per-founder evaluation against Tom's Founder Eval Framework, distinct
from Founder-Market Fit (founder + market match) and Team Dynamics & Composition (team
makeup, skills, gaps).

**Read the framework fresh.** Open
`/Users/tomseo/.claude/skills/founder-outreach/FOUNDER_EVAL_FRAMEWORK.md` at the start of this
subsection and apply whatever signals §5.A currently defines, with thresholds and anchors from
§6. The signal set evolves — do NOT hardcode a list from prior runs or from the
`neg1-enricher` skill description. Whatever the current framework says, that's what you score
against.

**Inputs (in priority order, all gathered upstream):**
1. **Linked Notes — call transcripts and Tom's notes** (Step 1b). Transcripts are primary —
   live, unfiltered conversation; quote specific moments verbatim where they support a signal.
   Tom's synthesized notes are second-tier signal (they reflect his real-time read).
2. **Founder bio / profile slides from Diligence Materials** (Step 1c).
3. **Raw LinkedIn profile scrape** (Step 1d).
4. **Founder online presence research** (Step 2 — personal site, blog, talks, podcasts, public
   posts, company-of-record artifacts).

Transcripts and Tom's notes are primary; web evidence extends and corroborates. When they
diverge on a given signal, the divergence is itself the diligence finding.

**Hard separation from -1 Scanner.** Run this evaluation from scratch every time, even if the
founder was previously sourced through the -1 Scanner. The -1 Scanner is a top-of-funnel
sourcing artifact built on web-only evidence; first-pass diligence is its own analysis built
on a richer, transcript-grounded base. Do NOT pull, link, or reference prior -1 Scanner
scoring in this subsection.

**Scope — founders and co-founders only.** Profile only founders and co-founders. Do NOT
profile non-founder C-level execs, early employees, advisors, or contractors — at pre-seed
and seed stage the analysis is squarely about the founding team. If the team includes
C-level titles whose founder status is ambiguous (a "CTO" who joined three months ago,
a "Head of Product" with no founder/co-founder language on LinkedIn or the deck), confirm
status before including. If status remains ambiguous after a good-faith check, mention them
in the Team Dynamics rollup but do not run a per-founder block on them.

**Stand-alone prose — no framework-internal references.** The prose must read clearly to
someone who has never opened FOUNDER_EVAL_FRAMEWORK.md. Do NOT cite section numbers (`§5.A`,
`§6.2 #3`, `§Signal 2 9/10 trigger`), framework-internal anchors (`composite-case Tuor`,
`anti-accelerator sub-pattern`), or version-stamps (`v0.8.1`) in the published output.
Translate framework-internal language into plain analytical English. The framework is your
reasoning scaffold; it is not the reader's reading material.

**Per-founder block structure.** Each founder/co-founder named on the Opportunity gets a
separate block in this order: primary founder first (CEO if titled, otherwise the founder
doing most of the talking on calls), then co-founders. Within each block, write three
sub-blocks in this exact order:

***1. Headline — Working Description + Recommendation.*** Open the block with
`### [Founder Name]`, then immediately:
- **Working Description** — 2-3 sentence TL;DR anchored on the founder's peak signal. Same
  structural shape as the Working Description field on the -1 Scanner row, but framed for
  diligence (already-in-conversation) rather than sourcing (haven't-met-yet).
- **Recommendation** — `Strong ✅` (peak signal rated Strong), `Moderate 🤔` (peak signal rated Moderate), or `Weak ❌` (peak signal rated Weak). Strong = signal scored 7–10 on the internal 0–10 scale; Moderate = 4–6; Weak = 0–3. This is a take on founder caliber as one input into the broader diligence
  picture — distinct from the -1 Scanner's `Strong / Moderate / Weak` framing
  (which is sourcing-context, since at -1 the question is "should I initiate contact?").
  **Moderate is NOT a neutral verdict.** The framework is spike-based-MAX, so the absence
  of a singular spike is itself a negative signal — "well-rounded mediocrity" (multiple
  Moderate ratings with no clear peak) is a downweight, not a wash. A founder who rates
  Moderate across all six signals reads as a *negative* on the founder dimension, even
  though no individual signal is bad.
  Keep the Recommendation line itself terse: verdict label + a one-line rationale tied to
  the peak signal pattern. Do NOT use this line to forecast where the broader diligence
  conviction will come from instead (category timing, unit economics, etc.) — that's the
  job of other sections, not this one.

***2. Per-signal scoring + Spike Summary (the double-click).*** For each signal currently
defined in §5.A of the framework, write a sub-block:

```
**[Signal Name] — Strong / Moderate / Weak**
[2-3 sentence rationale weaving transcript/notes evidence with web evidence. Lead with
transcript when present — cite specific moments: what was asked, what was said, in which
call. Cite the URL when web evidence extends or corroborates. If transcript and web diverge,
flag the divergence here AND surface it again in Conflict Callouts below.]
```

Use **Strong** (internally 7–10), **Moderate** (4–6), or **Weak** (0–3). The internal 0–10 scale from the framework drives the tier; express it as the label, not the number.

After the per-signal sub-blocks, close with a **Spike Summary** paragraph. Be opinionated
and honest — Tom wants a thought partner here, not a pleaser. The framework is explicitly
spike-based-MAX: *"well-rounded mediocrity does not beat singular spikiness."* If the
founder has a clear spike (peak signal rated Strong), name it and explain in 2-3
sentences why this signal is the thesis anchor — bold the peak-signal sentence. **If the
founder does NOT have a singular spike (multiple signals rated Moderate with no clear peak),
say so plainly and call it a negative.** Do not search for the most-defensible-anchor
or invent narrative cover. The framework treats well-rounded mediocrity as a *negative
read* on the founder dimension, not a neutral one — and the prose should call that out
directly. Phrases like "the deal can still work" or "this isn't fatal" are pandering;
the founder eval's job is to deliver a clean read, not to hedge. The honest read is what makes this section useful as diligence input. If a
founder's spike sits in a domain that does not apply to the current bet (e.g., deep
cross-border finance reps in a healthcare AEO startup), call that out — score them on
*relevance-adjusted* spike, and if the relevance-adjusted spike is weak, write "Weak"
clearly, not euphemistically.

***3. Conflict Callouts.*** Final sub-block of each founder block. List every inconsistency
surfaced during scoring — places where the founder's verbal framing diverges from the public
record, where two transcripts contradict each other, where claimed achievements don't appear
in LinkedIn or third-party coverage, or where the live narrative has shifted across calls.
Each callout is a bulleted item:
- A bold one-line label naming the inconsistency.
- 1-2 sentences laying out both sides with sources (which call, which URL).

If no inconsistencies surfaced, write a single line: *No notable inconsistencies between
conversational framing and public record.* Do NOT fabricate to fill space — absence is itself
a signal that the public and private narrative cohere.

**Solo-founder handling.** If the company is solo-founder, run the per-founder block once and
skip the Team Dynamics rollup below. Replace it with a single-paragraph note on solo-founder
execution risk and what hires would derisk it.

**Team Dynamics rollup (multi-founder only).** After the per-founder blocks, close the
subsection with a short **Team Dynamics** paragraph (best-effort v1 — iterate from feedback).
Considerations: complementarity of spikes (e.g., one founder is a Range/Earned Reps spike,
the other is Intellectual Rigor — does the team cover the surface area), who appears to be
lead in calls, gaps the team doesn't fill that the next hire needs to.

**6. Risks**
Isolate the most plausible failure modes, framed as specific mechanisms by which the thesis breaks.

Each risk gets a numbered subsection with a **descriptive title** that names both the risk category
and the specific mechanism (e.g., "6.1 Platform Dependency — Rain as Single Point of Failure" or
"6.2 Underwriting Model — Unproven Credit Engine in a Data-Scarce Market"). Do NOT append
probability, severity, or impact labels to the title — no "(High Impact, High Probability)", no
"(Fatal/Kill Shot)", no "(Medium-High)". The title names the risk; the prose and evidence below
it convey severity. Use the pre-mortem
skill's failure mode taxonomy: Category/Market Formation Risk, Product/Technical Loop Risk,
GTM/Commercial Execution Risk, Team/Organizational Fragility Risk, and a fifth Thesis-Specific
Risk if applicable.

Each risk has two components:
1. An **analytical paragraph** (or multiple paragraphs) grounding the risk in specific evidence
   from the diligence record — data points, founder claims vs. independent findings, comparable
   company metrics, specific dollar amounts.
2. A ***Failure mode:*** paragraph in bold-italic describing the precise mechanism by which it
   breaks, with quantified impact where possible. This paragraph should read as a specific
   narrative of how the thesis collapses — not an abstract description.

**CRITICAL — formatting rules for the Risks section:**
- Do **NOT** use subjective probability/severity labels (e.g., "HIGH PROBABILITY, HIGH IMPACT",
  "MODERATE (65%)", "Medium-High / Fatal"). These are hand-wavy without data and make the
  analysis look like a generic risk matrix instead of investor-grade thinking.
- Do **NOT** create a Risk Summary table with "Probability" or "Impact" columns.
- **DO** end with a Risk Summary table with exactly these columns: **Failure Mode | Key Evidence |
  Kill Shot?** — where Kill Shot uses "Yes — independent path to zero" or "Conditional —
  compounds other risks / slows thesis" (not probability labels).
- Anchor every risk in **specific evidence** that makes it material, not subjective assessments
  of likelihood.

**7. Path to Next Round**
Backsolve what traction the company needs to unlock the next financing round. Six subsections:

*Seed Comps — What Seed Rounds Are Going For:* Search the Notion Notes database for Deal Digest
entries (Category = Research, Name contains "Deal Digest") and pull Seed-stage comps (any sector)
to establish what the prevailing Seed market looks like. Weight more recent data points higher.
Do not mislabel companies — if the product category is not stated in the digest, omit it rather
than guessing. Present as a table with columns: Company | ARR / Revenue | Valuation | Round
Size | Lead | Notes. Follow the table with a 2-3 sentence synthesis of where the Seed bar sits
today (e.g., "Seeds in this cohort are clustering around $X-Y ARR at $A-B post, typically led
by..."). Seed and Series A are distinct rounds with distinct markets — do not combine them into
a single table.

*Series A Comps — What Series A Rounds Are Going For:* Same sourcing pattern as Seed Comps, but
pulling Series A data points. Present as a table with columns: Company | ARR / Revenue |
Valuation | Round Size | Lead | Notes. Follow the table with a 2-3 sentence synthesis of the
Series A bar. This establishes what the company needs to look like to raise an A from the kinds
of firms that lead at this stage, and is the reference point for Scenario B of the Implied
Parameters subsection below.

*Adjacent Space Comps — Who Else Is Building in This Neighborhood:* From the same Deal Digest
entries (and targeted web research where the digest doesn't cover a known competitor), pull
comps for companies building in the specific or adjacent space this opportunity occupies. The
purpose is different from stage comps: this shows Tom the competitive capital landscape — who
else is attracting capital in this neighborhood, on what terms, and from whom. For each comp,
capture what they're building, what they've raised (total and most recent round), traction
figures (ARR, users, or whichever metric the digest reports), valuation, and who's invested
(lead and notable participants). Present as a table with columns: Company | Description | Total
Raised / Latest Round | Traction | Valuation | Investors | Notes. Sort by stage earliest to
latest. **Each row must be a named, discrete company — never a cluster aggregate (e.g., "Nauta
portfolio," "YC cohort"). If a fund's portfolio or a cohort is noteworthy, mention it in the
prose synthesis below the table, not as a row.** Follow the table with a short prose synthesis
(1-2 paragraphs) on what the adjacent landscape signals — is capital consolidating around a
particular approach, are incumbents over- or under-capitalized relative to the opportunity,
which investors are building conviction in this neighborhood.

*Implied Parameters for Next Round:* Based on comps, define two scenarios for the next round.
Scenario A should be the round achievable from the current raise (e.g., seed from pre-seed).
Scenario B should be the larger round that likely requires an intermediate raise first (e.g.,
Series A). **Name each scenario with its round type** (e.g., "Scenario A — Seed: Pedigree + Early
Signal ($8-12M raise at $40-60M post)" not just "Scenario A"). Each scenario should describe the
archetype — what the company looks like at that stage, what metrics are required, what investors
would need to see.

*Revenue Build — Can [Raise Amount] Get [Company] to the Next Round?:* The central analytical
exercise. Start with **capital allocation**: how much of the raise funds the product/portfolio vs.
operations vs. key hires vs. licensing/regulatory costs. Then build the **per-customer revenue
math** — this must be specific to the business model (e.g., for consumer credit: # of active
accounts × revenue per customer including annual fee + interchange; for enterprise SaaS: # of
customers × ACV). Then present a **chronologically ordered traction requirements table** for
Scenario A, with columns: Milestone | Target | Timeline | Rationale. The annualized revenue
run-rate is the anchor milestone; every other milestone (licensing, hires, loss rates, warehouse
lines, etc.) should connect back to what's required to hit the revenue target. Reconcile
explicitly: does the raise provide enough runway to reach these milestones?

If Scenario B is not reachable from the current raise, state this clearly and explain the
intermediate step required. Include a separate, shorter traction requirements table for Scenario B.

*What Breaks the Path:* Identify the tightest sequencing constraint and how delays cascade. This
should be a prose analysis (1-2 paragraphs) that traces the critical path — what has to happen
first, what depends on it, and how a slip at the bottleneck compresses everything downstream.

**Suggested Additional Analysis**
Bulleted list of specific follow-up diligence workstreams.

**Sources**
Grouped into exactly two sections. The overriding principle is: **every source should be a
clickable link wherever possible.** During Step 1 you fetched Notion pages, Drive files, and
web sources — you have the URLs. Use them. A source entry without a hyperlink when one was
available is a miss.

1. **Internal Materials** — every entry MUST be a clickable hyperlink to the Notion page or
   Google Drive file it references. You fetched these URLs in Step 1; link them here. Format:
   `[Document Name](https://www.notion.so/<page-id>)` with a brief annotation after the link.
   For example: `[Reed & Nolan Intro Call — March 19, 2026](https://www.notion.so/abc123) —
   Founder backgrounds, solution architecture, GTM strategy`. For Google Drive files, link
   directly: `[Clusia Memo](https://drive.google.com/file/d/<id>/view)`.

   **Memo entries must reconcile against the Step 1e manifest.** Every `Inverted Capital Memo:
   [X]` (or equivalent memo-citation) entry must correspond to a memo actually fetched in
   Step 1e, with the URL pointing to the manifest's Drive file — not a guessed Notion page ID
   for the Opportunity, and not the Opportunity page itself. If a portfolio company has no
   memo in the manifest, do not list it under memos. It may still appear elsewhere in
   Internal Materials (as an Opportunity page, a call note, etc.) with accurate labeling.

2. **External References** — every web source researched in Step 2 MUST include the URL.
   Format: `[Source Name](URL) — brief relevance annotation and key data contributed`. Group
   logically (Founder Backgrounds, Competitive Landscape, Market Data, Regulatory, etc.) with
   subheadings. For backchannel feedback from specific people, link to the Notion feedback note
   if one exists.

Include a **Key Data Points Referenced** sub-section where every market statistic or external
data point cited in the analysis is linked to its source URL.

### Writing Guidance

- **Be specific, not generic.** "Execution risk" is not useful. "Neither founder has built
  underwriting models for Mexican consumers" is useful.
- **Ground claims in evidence.** Every assertion should trace to a call note, a deck, a data point,
  or a research finding.
- **No unverified numeric precision.** Specific numbers not traceable to a source document ("1–2
  EMs per year globally", "sources 500 companies annually") are a hallucination risk marker. If a
  number can't be cited, use qualitative language instead ("a small number of", "selectively", "a
  few per year"). The absence of a number is better than a fabricated one.
- **Bear cases lead with counter-evidence.** When writing a bear case or concern, open with the
  strongest counter-evidence first, then state the residual risk. Burying the rebuttal as a
  parenthetical after the concern undersells the evidence Tom has in hand and produces a
  one-sided read. The structure should be: "Counter-evidence X suggests Y is not fatal, but
  the open question is Z" — not "Risk Z exists (mitigated somewhat by X)."
- **Flag contradictions.** When founder claims diverge from independent research, present both
  with a "yellow flag" notation.
- **Include visual artifacts.** For data-dense topics (market sizing, competitor comparison,
  regulatory findings, team experience), create tables. Tables are more information-dense than
  prose and help Tom scan quickly.
- **Open Questions should be answerable.** If you can take a stab at answering an open question
  with web research, do so. Only leave questions open when they genuinely require founder input
  or proprietary data.
- **Paragraphs must be substantive.** Each paragraph should be 4-6 sentences minimum, developing
  a single analytical thread with evidence and caveats. One-to-two sentence paragraphs lack the
  nuance and fidelity this analysis demands. If a paragraph is too short, it either needs more
  evidence or should be merged into an adjacent paragraph.
- **Prose over lists.** Default to flowing prose with data woven into the argument. Use bullet
  lists only for Open Questions, Need to Believe items, and Suggested Additional Analysis. For
  everything else — market dynamics, competitive positioning, product analysis, team assessment,
  risk analysis — write in paragraphs. If you find yourself building a bulleted list of observations,
  convert it to prose.
- **Operational specificity over jargon.** Vague mechanism claims — "scaling via API and hardware",
  "monetization via the digital layer", "compounding through the data flywheel" — are placeholder
  language masquerading as analysis. For every such phrase, spell out the concrete pieces: WHAT
  API (is the subject company building it, or consuming someone else's)? WHAT hardware (whose, who
  buys it, who installs it)? WHAT unit scales (locations? machines? scans? users? per what)? If
  the underlying mechanics aren't clear from the source materials, drop the claim rather than
  hand-wave. Kalos first-pass (2026-05-28): "scaling via API / hardware" appeared twice in the
  Context section with no concrete grounding — Tom flagged both as opaque.
- **Deictic completeness on quotes + references.** When quoting a founder verbatim, bracket-annotate
  any ambiguous referent so the quote reads cleanly out-of-context. Example: *"Function can't show
  you delta data sets [in core health metrics over time]"* — not just *"Function can't show you
  delta data sets"*. Same rule for named people / brands / programs the reader hasn't been introduced
  to in the same paragraph: bracket-annotate their role on first mention. *"Tony Orlando (largest US
  DEXA-scanner distributor) has agreed to…"* — not *"Tony Orlando has agreed to…"*. If the role
  isn't in source materials, find it before naming the person, or drop the reference. Kalos
  first-pass (2026-05-28): "Function delta data sets" and "Tony Orlando's distribution channel"
  both shipped with zero contextual scaffolding; Tom couldn't parse what was being claimed.

### Common Deviations — Do NOT Do These

The following are specific anti-patterns observed in prior outputs that deviate from this skill.
If you catch yourself doing any of these, stop and correct:

1. **Scorecard labels on frameworks** — writing `**Assessment: STRONG FIT**` instead of developing
   the assessment through prose. The assessment should emerge from the argument, not be stamped
   as a header.
2. **Numbered Need to Believe items** — using 1., 2., 3. instead of bullet points. Always use
   bullets with bold labels.
3. **Probability/severity risk labels anywhere** — writing "HIGH PROBABILITY, HIGH IMPACT",
   "MODERATE (65%)", or "(High Impact, Medium Probability)" in the Risks section — whether in
   subsection titles, body text, or summary tables. The skill explicitly prohibits this. The
   title names the risk mechanism; the prose conveys severity through evidence. Use the "Kill
   Shot?" framework in the summary table instead of probability/impact grids.
4. **Checkmark capability matrices** — using ✓/✗ tables to compare competitors. Analyze
   competitive dynamics in prose.
5. **Speculative pricing hypotheticals** — inventing pricing tiers not grounded in the diligence
   record. Ground pricing analysis in actual evidence.
6. **Fabricated burn rate numbers** — creating monthly burn estimates when no team size, salary,
   or operating cost data exists in the record. Note the gap instead.
7. **Missing "Founder Evaluation Through the Inverted Lens"** — skipping the third Team
   subsection. This is required.
8. **Generic Path to Next Round** — not following the six-subsection structure (Seed Comps,
   Series A Comps, Adjacent Space Comps, Implied Parameters with named scenarios, Revenue
   Build with capital allocation and per-customer math, What Breaks the Path). Each subsection
   has specific required content. Collapsing Seed and Series A into a single Stage Comps
   table — or folding Adjacent Space Comps back into a generic Benchmarks section — is a
   regression. Each cut serves a distinct analytical purpose and must be kept separate.
9. **Wrong Sources grouping** — using ad hoc categories instead of "Internal Materials" and
   "External References" with "Key Data Points Referenced."
10. **Thin paragraphs** — writing 1-2 sentence paragraphs that state conclusions without
    developing the reasoning. Every major claim needs evidence and context.
11. **Broken tables in Notion** — pasting standard markdown pipe tables that render with visible
    separator rows (`|---|---|`) as literal text. See Step 4's "Notion Table Formatting" section
    for how to handle this.
12. **Unlinkable sources** — listing source names as plain text when you have the Notion page URL
    or web URL available from the evidence-gathering phase. Every source entry should be a
    clickable hyperlink.
13. **Backslash-escaped special characters in Notion content** — writing `\\~2 yrs` or `\\$50M`
    instead of `~2 yrs` or `$50M`. Notion does not require backslash escaping of `~`, `$`, or
    other special characters. Write them directly — backslashes render as literal text.
14. **Cluster aggregate rows in comp tables** — inserting a named row like "Nauta portfolio" or
    "YC W23 cohort" as if it were a company. Every row in a comp table must be a named, discrete
    company. If the observation is about a cluster (a fund's portfolio, a cohort, a category
    bucket), move it to a prose sentence below the table — not a table row.
15. **Unverified numeric precision** — writing "1–2 EMs per year globally," "sources 300 companies
    annually," or any other specific figure that isn't traceable to a source document. See Writing
    Guidance — use qualitative language when no sourced number is available.
16. **Regulatory framing applied to non-regulated verticals** — characterizing a home services,
    construction, or logistics company as operating in a regulated-adjacent space, requiring a
    licensed partner, or needing a BaaS structure, simply because the current opportunity
    involves financial infrastructure. See the Regulatory framing guardrail in Framework Mapping.
17. **Citing a memo for a portfolio company whose memo you did not read** — naming "the [X]
    memo" or quoting from it when X is not in the Step 1e memo manifest. Portfolio status does
    not imply a memo exists. Concrete prior incident: Kestrel first-pass (2026-05-13) cited
    "the original Suppli memo" with a verbatim quote and grouped MakersHub into the "memo set"
    framing — the quote came from `DECISION_RETROS.md`, not a memo, and no MakersHub memo
    exists. See the Memo citation traceability guardrail in Framework Mapping.
18. **Upgrading decision-retro snippets to memo content** — quoting a line from
    `DECISION_RETROS.md` as if it appeared in a memo, or listing a retro-sourced company in
    "the memo set covering A, B, C…". Retro nuggets are retro nuggets — cite them as such if
    used at all.
19. **Hallucinated founder names** — writing a founder's surname from memory or pattern-match
    rather than verifying against a primary source on that specific company. The Suppli Opp
    `Contact` field is `rayers@gosuppli.com` (Ryan **Ayers**), but the Kestrel first-pass
    wrote "Ryan Walsh" — a pattern-match against a common surname, with no traceable source.
    See the Founder names guardrail in Framework Mapping.
21. **Fabricated time-with-founders figures** — stating "X minutes of conversation" or "X hours
    with the team" without deriving the number from a verifiable source. The only valid sources
    are: (a) the calendar event duration, (b) an explicit timestamp range in the transcript, or
    (c) multiple call events summed from calendar data. Never infer call length from transcript
    length or guess a round number. If the duration is not verifiable, write "one call" or
    "the May 13 call" — not a time figure.
22. **Architectural overlays projected onto portfolio analogs** — labeling a portfolio company
    with a structural pattern that fits the subject company but not the analog. See the
    "Architectural overlays" bullet in the Analog claims rule above. Concrete prior incident:
    Kalos first-pass (2026-05-28) called Oun Homes "the physical operating surface" because
    Kalos has physical scan locations — Oun is pure software. Same doc claimed portfolio
    companies "own the physical acquisition layer + digital monetization layer" framework
    when NONE of Tom's portfolio companies own a physical acquisition layer. Test before
    writing: would Tom recognize this characterization from the analog's actual business?
    If you're hand-wave-extending a pattern, drop the analog.
23. **Vague mechanism claims** — phrases like "scaling via API / hardware", "monetization via
    the digital layer", "compounding through the data flywheel" without spelling out the
    concrete mechanics (whose API, whose hardware, what scales per what). See the
    "Operational specificity over jargon" bullet in Writing Guidance. Kalos first-pass
    (2026-05-28) had two such phrases in the Context section with no underlying grounding.
24. **Bare-name references without role context** — naming a person, brand, or program the
    reader hasn't met without a parenthetical role description on first mention. *"Tony
    Orlando's distribution channel"* leaves Tom guessing who Tony Orlando is. Required:
    *"Tony Orlando (the largest US DEXA-scanner distributor)…"*. Same rule for verbatim
    founder quotes — bracket-annotate ambiguous referents. See "Deictic completeness on
    quotes + references" in Writing Guidance.
25. **Source-blurb voice misattribution to Tom** — the source blurb / dealflow-referrer email
    contains the REFERRER's voice and relationships, NOT Tom's. When a forwarded intro email
    says "I've known X for 5+ years", that "I" is the referrer (Keith Bender, Bryce Johnson,
    Chris Oh, etc.) — NOT Tom. Drafter must attribute correctly: *"per Keith Bender (the
    source referrer), who has known Galen and Vinay for 5+ years…"* — NEVER *"Tom's
    multi-year relationship with Galen and Vinay"* unless Tom independently confirmed the
    relationship in his own call notes or DECISION_RETROS. Concrete prior incident: Kalos
    first-pass 2026-05-28 attributed Keith Bender's 5+ year relationship with Galen + Vinay
    to Tom; Tom does not actually know either of them. Pattern check before writing any
    "Tom has known…" / "Tom's relationship-context…" / "per Tom…" sentence: does the cited
    [Source Blurb] / [2] / etc. originate from Tom's own materials, or from a third-party
    forwarded email? If the latter, re-attribute to the actual narrator.
26. **Founder mislabeling of senior hires / acquihires / advisors** — only people who appear
    in the Opp's `🏁 Founder(s)` relation field OR who are explicitly titled "Founder" or
    "Cofounder" in primary source materials (Opp page body, deck team page) qualify as
    founders. Heads of X (Head of AI, Head of Product, Head of GTM), acquihired senior
    hires, advisors, and validators are NOT founders. Use their actual title. Concrete prior
    incident: Kalos first-pass 2026-05-28 filed Vinay Kanumuri (Head of AI, Spotr acquihire)
    and Galen Lewis (Head of Product, Spotr acquihire) under a "Other Founders" H3; Tom
    flagged. The Spotr team came in via acquihire, NOT as cofounders. Verify against the
    Opp's `🏁 Founder(s)` relation before writing any "founder" or "cofounder" label.
27. **Internal drafting-taxonomy labels in section headers** — H2/H3 section titles must not
    contain parenthetical category names from the drafter's organizing taxonomy. *"Other
    Founders (rollup)"*, *"Team (composite)"*, *"Risks (kill-shot bucket)"* — the parenthetical
    is the synthesizer's internal label for how the material was grouped, not a label the
    reader needs. Drop the parenthetical: *"Other Founders"*, *"Team"*, *"Risks"*. Same class
    as deviation #20 (internal Step references). Concrete prior incident: Kalos first-pass
    2026-05-28 shipped *"### Other founders (rollup)"* with the drafter's "(rollup)" tag in
    the rendered H3.
28. **Transcript speaker misattribution — Tom's voice attributed to the founder.** Inverse
    of deviation #25. When Tom raises a framework, analogy, push-back, reframing, or
    industry parallel in a call transcript and the founder agrees (or extends the point),
    the framework/analogy belongs to *Tom*, not the founder. Drafter must attribute
    correctly: *"Tom raised the Brex analogy on the June 2 call; Sonia agreed it captured
    the framing"* — NOT *"Sonia's Brex analogy"* or *"the Brex framing she described
    without prompting"*. Concrete prior incident: Alongside Medical first-pass 2026-06-06
    attributed the Brex-default-expense-policy analogy to Sonia and characterized it as
    her unprompted intellectual-honesty signal; in fact Tom introduced the analogy after
    Manish described the baseline-plus-gold content layer, and Sonia agreed it was
    "exactly the same thing." This misattribution inflates the founder-evaluation signal
    and is dangerous because the next reader (Tom himself, or a future drafter reading
    the doc) treats the analogy as founder-originated.

    **Mandatory pre-submit check.** Before final delivery, scan every sentence in the
    draft that contains a portfolio/industry analogy, framework name, or strategic
    reframing attributed to a founder, and verify against the transcript who *introduced*
    the framing vs. who *agreed with* it. The introducer owns the attribution. Tom-as-
    questioner is a high-frequency analogy-introducer in his transcripts; default toward
    Tom-introduction unless the transcript clearly shows the founder bringing it first.
    Also note: when Tom asks a sharp diagnostic question and the founder gives a clean
    answer, the *answer* belongs to the founder; the *question framing* belongs to Tom —
    these are separate attribution claims and the drafter must handle them separately.
20. **Internal step references in output** — writing "Step 1e memo", "Step 1e manifest",
    "the Step 1e manifest entry", or any reference to internal skill step numbers in the
    published analysis. These are drafting-process labels that should never appear in output.
    In the document, always say "the investment memos", "the Inverted portfolio memos", or
    name the specific memo by company — never "Step 1e".

---

## Step 4: Create the Notion Page

### 4a. Pre-Publish Lint — MANDATORY GATE

Before writing the analysis to Notion, run the deterministic verification pass at
`/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_lint.py`. The lint
catches the specific hallucination classes that prose-level guardrails have failed to
prevent (Kestrel-Suppli "Ryan Walsh", Kestrel-Oun "Robert", fabricated memo
citations, Clusia/Third Space portfolio misattribution, fabricated burn-rate
numerics, and uncited synthesized Working Thesis claims).

**Build the manifest first.** Write a JSON file to a tmp path with this shape:

```json
{
  "subject_company": "<name>",
  "memo_manifest": {"<analog company>": "<drive_url>", ...},
  "portfolio_set": ["<active+exited+committed companies from Opps DB>"],
  "founder_names_by_company": {
    "<subject company>": ["<founder full names from 🏁 Founder(s) + Contact email surnames>"],
    "<each analog company referenced in draft>": ["<their founders>"]
  },
  "narrator_names": ["Tom", "Claude"]
}
```

Sources for each manifest field:
- `memo_manifest` — exactly the manifest you built in Step 1e (company → Drive URL)
- `portfolio_set` — query Opps DB for `Status IN ("Active Portfolio", "Exited", "Committed")`
- `founder_names_by_company` — for the subject company AND every analog company named
  in the draft's Framework Mapping section, pull from that company's Opp row: the
  `🏁 Founder(s)` relation (People DB names) + `Contact` email local-parts where they
  resemble surnames (`rayers@gosuppli.com` → add "Ayers" as a candidate). Analog
  company founder sets MUST be populated — this is the check that catches the Kestrel
  failure mode.
- `narrator_names` — defaults `["Tom", "Claude"]`; extend if other narrator names
  appear in the doc.

**Write the draft markdown to a tmp file**, then run the lint:

```bash
python3 /Users/tomseo/.claude/skills/first-pass-diligence/first_pass_lint.py \
  --draft $WORKSPACE/draft.md \
  --manifest $WORKSPACE/manifest.json \
  --memo-text-dir /tmp/firstpass_memos \
  --labeled-transcripts-dir $WORKSPACE/labeled_transcripts \
  --meeting-note $WORKSPACE/meeting_note_raw.md
```

`--meeting-note` is MANDATORY in the canonical flow — it points to the raw
meeting note .md file. Notion-AI meeting notes routinely emit action items
of the form "Tom to share spelling / details for X and Y" when speech-to-text
mishears a company or person name. The `unverified_entity_grounding` check
extracts those entity names and fails the lint if the draft externally
grounds them (assigns a URL, funding stage, product positioning) without an
explicit unverified marker. Without this flag, the check is skipped and
transcription-error hallucinations like AgentBay's "Kibo / KSAP" fabrication
(2026-06-24 — the transcript mishearing of "Casap" became a wholly invented
company complete with a fake `kibocommerce.com` URL) can ship.

`--labeled-transcripts-dir` is MANDATORY in the canonical flow — it points to the
directory of speaker-labeled transcripts produced by `relabel_transcript.py` in
Step 1b. With it, the lint runs the deterministic `attribution_mismatches` check:
for every draft sentence attributing an analogy / framework / strategic reframing
to a named person, it locates the keyword's first occurrence in the labeled
transcripts and fails if the speaker who introduced the keyword is not the
attributed person. UNKNOWN turns (Haiku couldn't infer) are non-blocking — they
just can't confirm OR deny attribution. Keywords whose first mention is in
non-transcript material (deck, memo) are also non-blocking — attribution to the
founder is acceptable on founder-authored material. If the flag is omitted, the
attribution check emits a single informational warning instead of running —
surface that warning in the publish summary so Tom knows the gate was disabled.
Concrete prior incident: Alongside Medical first-pass 2026-06-06 attributed the
Brex-default-expense-policy analogy to Sonia when Tom actually introduced it on
the June 2 call; the attribution check exists to catch this class deterministically.

**Speaker-attribution script gate — MANDATORY (same hard-gate semantics as the
lint).** After the lint passes, run the deterministic speaker-attribution
verifier on the draft against EACH labeled call transcript:

```bash
for T in $WORKSPACE/labeled_transcripts/*.md; do
  python3 ~/.claude/scripts/verify_speaker_attribution.py \
      --draft $WORKSPACE/draft.md --transcript "$T"
done
```

Exit 0 = clean. Exit 1 = the JSON output's `{"flagged": [...]}` lists claims
whose speaker attribution contradicts the transcript. Every flagged claim MUST
be resolved before publish: re-attribute to Tom (when Tom introduced the
framing and the founder agreed) or reword to drop the attribution entirely.
This is code-enforcement of deviation #28's prose rule; the prose rule stays
in force for attributions sourced from non-transcript material the script
can't see.

`--memo-text-dir /tmp/firstpass_memos` is MANDATORY in the canonical flow —
Step 1e populates that directory with per-company memo text files. The lint
then runs an LLM-judge grounding check on every sentence that attaches a
portfolio analog to an architectural overlay (e.g., "Oun Homes owns the
physical operating surface"), verifying the overlay against the analog's own
memo text. If Step 1e was skipped or the cache doesn't exist, the lint emits
a single informational warning instead of running the check — surface that
warning in the publish summary so Tom knows the gate was disabled. Filename
conventions accepted by the lint: `<Company Name>.md`, `<Company Name>.txt`,
`<company_lower_underscored>.md`, `<company_lower_underscored>.txt`.

**Hard gate.** Exit code 0 = proceed to 4b. Exit code 1 = STOP. For each finding,
either (a) fix the draft and re-run, or (b) verify the finding is a genuine false
positive against the source materials and explicitly note the suppression in the
publish summary you surface to Tom. Do NOT publish past a failing lint without
explicit acknowledgement of each finding. "Lint failed but I shipped anyway" is the
exact behavior this gate exists to prevent.

Warnings/skipped notes (e.g., "memo-text-dir not provided") surface in the lint
output but do NOT fail the gate — they're informational.

The lint is a floor, not a ceiling — passing it doesn't mean the draft is correct,
only that the documented failure classes are absent. The prose-level guardrails in
Step 3 still apply. Three checks added 2026-05-28 (Kalos first-pass): the
analog-overlay grounding check above, `vague_mechanism_phrases` (catches "scaling
via API / hardware" and friends), and `bare_name_references` (catches possessive
person references without role context, e.g., "Tony Orlando's distribution
channel").

### 4b. LLM Audit — MANDATORY SURFACE

The audit mechanics (HARD EXIT GATE, iteration loop, Step 3.5 partial
normalization, exit codes, the never-use-the-bypass-alert prohibition) are
defined ONCE in `~/.claude/skills/research-artifact-audit/SKILL.md`. Read
that file in full before continuing this step.

**Diligence-specific wiring — apply these bindings when following research-artifact-audit/SKILL.md:**

| Binding (per research-artifact-audit) | Value (first-pass-diligence) |
|---|---|
| `DRAFT` | `$WORKSPACE/draft.md` |
| `SOURCES` | `$WORKSPACE/sources.md` |
| `AUDIT_JSON` | `$WORKSPACE/audit.json` |
| `JUDGE_PROMPT` | `/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.prompt.md` |
| `AUDIT_RUNNER` | `/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.py` |
| `MAX_ITER` | `3` |
| `WEB_RESEARCH_CAP` | `6` |
| `ITER_SNAPSHOT_PREFIX` | `$WORKSPACE/draft.iter` |
| `NORMALIZED_DRAFT` | `$WORKSPACE/draft.normalized.md` |

**Fire the audit-start Slack alert BEFORE building the source bundle.** This
is a diligence-specific progress ping (not part of research-artifact-audit's
generic flow):

```bash
ELAPSED_MIN=$(( ($(date +%s) - $(cat $WORKSPACE/start_ts.txt)) / 60 ))
COMPANY="<subject company name>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
🧪 First-pass audit starting for **${COMPANY}** — T+${ELAPSED_MIN} min from job start
EOF
```

Single line, no feedback prompt, no links. Do NOT include `💬 Reply in thread`
(that's reserved for Step 6b — the listener routes thread replies based on
that exact string).

**Diligence-specific source bundle structure (Step A in research-artifact-audit).**
Write `$WORKSPACE/sources.md` with this layout (the runner chunks at the
`==== … ====` boundaries):

```markdown
==== OPPORTUNITY PAGE ====

<full Opp page body, including properties summary>

==== LINKED NOTES ====

--- Note: <title> (<date>) ---
<full content, including transcript where present>

--- Note: <title> (<date>) ---
<full content>

==== DILIGENCE MATERIALS ====

--- Material: <name> ---
<full extracted text from deck/one-pager/memo/data room>

==== INVESTMENT MEMOS (Step 1e manifest) ====

--- Memo: <company> ---
<full memo text>

==== ANALOG COMPANY OPP ROWS ====

--- Opp: <company> ---
<Description, Founder Description, 🏁 Founder(s), Contact, page body — for every
analog portfolio company named in the draft's Framework Mapping section>

==== PUBLIC PRODUCT SURFACE ====

--- Page: <URL> ---
<key extracted text / structured signal from the company's marketing, product, pricing,
docs, integrations, security pages — only populated when §2 Product section was drafted>

==== ENGINEERING SIGNAL ====

--- JD: <URL> ---
<verbatim stack mentions + team-shape signals from careers pages>

--- Blog: <URL> ---
<key passages from engineering blog posts>

--- GitHub: <org URL> ---
<observable signal: languages, repo names, commit cadence>

==== COMPETITOR / PEER SURFACES ====

--- Peer: <name> (<URL>) ---
<extracted signal from each peer's product / docs / pricing — only when §2 cites the peer>

==== EXTERNAL RESEARCH ====

--- Source: <URL or title> ---
<key data points and quotes from web research>

==== COST CALIBRATION ====

<full text of /Users/tomseo/.claude/skills/shared-references/product-build-cost-calibration.md
so the audit can verify every "(per calibration §X)" cite the Product section makes>
```

The bundle layer (this list) is diligence-specific; the auto-chunking, judging,
gate-checking, iterating, normalizing, and publish-summary requirements are all
in research-artifact-audit/SKILL.md.

**Lint/audit relationship.** The Layer 1 deterministic lint at Step 4a catches
surface-form hallucinations with regex. The Layer 2 LLM-judge audit (per
research-artifact-audit) catches the subtler class — claims that don't trip
any regex but whose substance isn't traceable to the source bundle. Both gates
must pass before publish.

**Diligence-specific Slack publish-summary surface (Step D in research-artifact-audit).**
The Slack alert appends `⚠️ Audit: <N> untraced after <K> iterations, <M>
partials normalized` as a fourth line when there are residual untraced findings
OR any partials were normalized. If the audit ends with 0 untraced and 0
partial cleanly, no `⚠️` line — the alert stays at three lines. The substance
(residual untraced claims with judge notes; normalized partials as before→after
diffs) is required by research-artifact-audit; this paragraph only specifies
the Slack format.

### Notion Table Formatting

Notion's markdown parser is finicky with tables. Standard markdown tables with separator rows
(|---|---|) often render the separator as literal text instead of being parsed as a table
delimiter. To avoid broken tables in Notion:

- **Use simple tables** with the Notion enhanced markdown `<table>` block syntax when available.
  If the Notion MCP supports it, prefer that over raw pipe tables.
- **If using pipe tables**, ensure each row has exactly the same number of columns. The separator
  row must use at least three dashes per column (`|---|---|`) with no leading/trailing whitespace
  on the line. Some Notion environments require the separator row to use exactly `---` (three
  dashes, no more).
- **Test table rendering** after creating the page. If tables render as raw text with visible
  pipe characters and separator rows, replace the table content using `notion-update-page` with
  `update_content` — try removing the separator row entirely, or reformatting as a bulleted
  summary if the table refuses to render.
- **Fallback**: If tables consistently fail to render in Notion's markdown, convert the data into
  bold-label prose format instead (e.g., "**Scrut** — Seed+, $13M ARR, $65M–$100M valuation,
  led by Boldstart"). A clean prose summary is always better than a broken table.

### Character Escaping in Notion Content

When writing the analysis content to Notion, do NOT backslash-escape special characters like
`~`, `$`, or other markdown-adjacent symbols. Notion's markdown parser does not require these
escapes, and they render as literal backslashes (e.g., `\\~2 yrs` instead of `~2 yrs`). Write
tildes, dollar signs, and other special characters directly without any preceding backslash.

**Select the final draft.** Before writing to Notion, pick the source-of-truth
file deterministically:

```bash
if [ -f $WORKSPACE/draft.normalized.md ]; then
  FINAL_DRAFT=$WORKSPACE/draft.normalized.md
else
  FINAL_DRAFT=$WORKSPACE/draft.md
fi
echo "publishing from: $FINAL_DRAFT"
```

Load `$FINAL_DRAFT` and use its contents as the `content` field below. Do
NOT use the unnormalized draft when a normalized version exists — that
re-introduces the partials Step 3.5 just resolved.

Create a new page in the Notes database (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) using
`notion-create-pages`:

```
parent: { data_source_id: "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" }
pages: [{
  properties: {
    "Name": "[Claude] [Company Name] Master Diligence Doc — MM.DD.YYYY",
    "Category": "Diligence",
    "Opportunity": "[Opportunity page URL]"
  },
  content: "[Full analysis content in Notion-flavored markdown]"
}]
```

Save the resulting Notion page URL — it will be referenced in the PDF and the Signal alert.

**Fire publish-progress alert (1 of 3).** The publish phase is silent for 15-20
min between the audit-start alert and the final completion alert. These
three progress pings make the silent stretch legible to Tom and surface any
stuck step quickly.

```bash
COMPANY="<subject company name>"
NOTION_URL="<URL from the page just created>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
📝 **${COMPANY}** Notion page published — [first-pass]($NOTION_URL). Building PDF next.
EOF
```

### 4c. Set the Claude Logo Icon

Immediately after creating the page — before moving to Step 5 — set the page icon to the custom
Claude logo emoji. This is what lets Tom visually distinguish Claude-generated analysis from his
own notes in the Notion list view, so skipping it defeats the purpose of having a structured
notes system.

Call `notion-update-page` with the icon field set to `:claude-color:`:

```
notion-update-page({
  page_id: "<newly created page ID>",
  command: "update_properties",
  properties: {},
  content_updates: [],
  icon: ":claude-color:"
})
```

The colon-wrapped name format is the only format that works with the Notion MCP for custom emojis.
Do not use `custom_emoji::` prefix, raw UUIDs, or JSON objects — those will silently fail or set
the wrong icon. Use `:claude-color:` exactly as shown above.

---

## Step 5: Build the PDF

**Start from the canonical template** at
`/Users/tomseo/.claude/skills/shared-references/pdf_builder_template.py` — copy it to the
outputs directory and fill in the four CUSTOMIZE blocks at the top. Do NOT write a PDF script
from scratch; the template already contains all known encoding fixes and formatting logic.

Install reportlab first: `pip install reportlab --break-system-packages -q`

**CRITICAL — Notion API encoding artifacts:** The Notion MCP returns content as a JSON string
where certain characters are double-escaped. You MUST apply these three preprocessing lines to
the raw markdown content extracted from the tool-result JSON **before** passing it to the parser:

```python
md_content = md_content.replace('\\n', '\n')   # literal \n two-char → real newline
md_content = md_content.replace('\\"', '"')     # literal \" → real quote
md_content = re.sub(r'\\+\$', '$', md_content) # \\$ or \\\$ → $
md_content = re.sub(r'\\+~', '~', md_content)  # \\~ → ~ (tildes)
md_content = md_content.replace('\\<', '<')     # \< → < (HTML tag opener)
md_content = md_content.replace('\\>', '>')     # \> → > (HTML tag closer)
md_content = re.sub(r'(?m)^\\\^', '^', md_content)  # \^ → ^ at line start (footnote markers; Notion roundtrip escapes ^N)
md_content = md_content.replace('\\\\', '\\')   # \\ → \ (literal backslash) — apply LAST so prior rules don't double-strip
```

Skipping these lines produces a broken PDF with literal `\n` characters everywhere, raw `\$50M`
strings, and HTML table tags rendered as escaped text (`\<table\>\<tr\>\<td\>...`) instead of
actual tables. The `\<` / `\>` / `\\` rules were added 2026-04-27 after observing the Inlets
first-pass output ship escaped table syntax through the entire 20-page comp section.

**HTML table format:** Notion enhanced markdown uses `<table header-row="true"><tr><td>…</td></tr></table>`
syntax for tables, not standard markdown pipe tables. The canonical template's `parse_content()`
function handles both formats. If you write a custom parser, it must handle both.

Read the formatting spec at `/Users/tomseo/.claude/skills/shared-references/long-form-pdf-spec.md` for the exact style definitions.

**Post-table / post-chart spacing:** After every `Table` flowable (and any chart or visual
exhibit), insert a `Spacer(1, 10)` to enforce 10pt of whitespace before the next element.
This prevents body text or headers from crowding the bottom edge of tables and exhibits.

**Page numbers:** Include page numbers on every page — bottom centered, 8pt Helvetica, black.
Use reportlab's `onFirstPage` and `onLaterPages` callbacks with `canvas.drawCentredString()`.

**Markdown-to-PDF text conversion:** When converting markdown content to reportlab Paragraphs,
properly convert all markdown formatting to reportlab XML tags. Do NOT allow raw markdown
asterisks to appear in the rendered PDF. Write a `clean_text()` helper that handles:
- `***text***` → `<b><i>text</i></b>`
- `**text**` → `<b>text</b>`
- `*text*` → `<i>text</i>`
- `[text](url)` → `<a href="url" color="black"><u>text</u></a>`
Apply `clean_text()` to every string before passing it to `Paragraph()`.

**CRITICAL — XML escaping order in `clean_text()`:** Reportlab's `Paragraph` uses XML-like tags
(`<b>`, `<i>`, `<a>`) for inline formatting. If you escape `<` and `>` to `&lt;`/`&gt;` before
converting markdown to reportlab tags, the tags themselves get escaped and render as literal text
(e.g., `<b>Sponsor banks</b>` visible in the PDF). The correct order is:
1. First, escape `&` to `&amp;` (to avoid breaking XML entities).
2. Then, escape `<` and `>` to `&lt;`/`&gt;` (to prevent user content from being parsed as XML).
3. **Then** convert markdown formatting (`**bold**`, `*italic*`, `[links](url)`) to reportlab
   XML tags (`<b>`, `<i>`, `<a>`).
This ensures user content is safely escaped while the formatting tags are valid XML that
reportlab can render.

**Title and subtitle as flowables, not page template:** The title and subtitle must be added as
`Paragraph` flowables at the top of the story — NOT drawn via `onFirstPage`/`onLaterPages`
canvas callbacks. Canvas callbacks execute on every page (or the first page), which causes the
header to repeat. Page numbers should still use canvas callbacks (`onFirstPage` and
`onLaterPages`), but the title/subtitle should be regular flowables that appear once.

Save the PDF to `/tmp/[Company]_Master_Diligence_MM.DD.YYYY.pdf`
(replace spaces in company name with underscores). Do NOT save to iCloud Downloads or
`~/Downloads` — the Google Drive upload in Step 6 is the permanent artifact; the local
file is staging only and lives in `/tmp/`.

The PDF header should be:
- **Title (14pt bold):** `[Company Name] ([Stage]) — Master Diligence Doc`
  - One title across the full diligence lifecycle — initial first-pass, every update
    prepended by `update-diligence-priors`, and the final assessment prepended by
    `finalize-diligence`. The artifact's identity is "Master Diligence Doc" because as
    updates accumulate the doc is no longer just a first-pass.
  - `[Stage]` comes from the Opportunity row's `Stage` property (e.g. `Pre-Seed 💡`,
    `Seed 🌱`, `Series A 🅰️`). Strip any trailing emoji and surrounding whitespace
    before rendering — the parens should contain only the textual stage label
    (e.g. `Pre-Seed`, `Seed`, `Series A`). If the Stage property is empty, omit the
    parenthetical entirely rather than render `()` — fall back to
    `[Company Name] — Master Diligence Doc`.
- **Subtitle (9pt italic, dark gray):** Canonical first-pass shape is exactly
  `First-Pass: <Long Date> | Notion` — where "Notion" is a clickable hyperlink to
  the Notion diligence page URL. Drop in the literal SUBTITLE assignment when
  customizing the template:

  ```python
  NOTION_PAGE_URL = "https://www.notion.so/<diligence-page-id-no-dashes>"
  SUBTITLE = f'First-Pass: May 28, 2026 | <a href="{NOTION_PAGE_URL}" color="#1A5FB4">Notion</a>'
  ```

  reportlab's Paragraph renderer accepts `<a href>` markup natively. The
  `color="#1A5FB4"` makes the link visually distinguishable.

  **Update-priors / finalize use a different shape** (managed by their own
  skills): `Latest Update: <date> | First-Pass: <date> | Notion`. The
  `pdf_header_check.py` gate (Step 5b below) accepts EITHER shape, so this
  template works across all three flows by date.

### Step 5a. Inner H1 Anchor — MANDATORY at top of body

Before publishing to Notion AND before building the PDF, the markdown body MUST
open with the inner H1 anchor:

```
# First-Pass Diligence — <Long Date>
```

(em dash, matches the `pdf_header_check` P4 pattern `^First-Pass Diligence — .+$`).

This anchor is the structural divider that lets `update-diligence-priors` prepend
Update sections above the first-pass content, and that `finalize-diligence` uses
to find the start of the first-pass section. The PDF renderer also uses it as a
page-break trigger. Skipping it means future updates can't cleanly slot in, and
the PDF won't visually mark where first-pass content begins. Concrete prior
incident: Kalos first-pass 2026-05-28 shipped without this anchor; Tom flagged it
during review and required a rebuild.

### Step 5b. PDF Header Check — MANDATORY GATE before Drive upload

After the PDF builds successfully and before uploading to Drive, run:

```bash
python3 ~/.claude/skills/shared-references/pdf_header_check.py --pdf /tmp/<output>.pdf
```

The check enforces:
- **P1** title shape `<Company> — Master Diligence Doc`
- **P2** subtitle shape — accepts BOTH first-pass-only and latest-update shapes
- **P3** no debug-pattern leaks on first page
- **P4** inner H1 anchor `# First-Pass Diligence — <date>` present in body

**Exit 0 = proceed to Step 5c. Exit 1 = STOP. Do NOT upload to Drive.** Fix the
build script (subtitle format, title em dash, missing H1 anchor) and rebuild.
Shipping past this gate is the exact behavior the gate exists to prevent.

### Step 5c. PDF Format Guard — MANDATORY GATE before Drive upload

The header check verifies the title/subtitle/anchor *text*. This gate verifies
the *typography* — every text-span's `(font, size, color)` against the MEASURED
Factir reference profile. Run it after the PDF builds, before upload:

```bash
python3 ~/.claude/skills/shared-references/pdf_format_guard.py \
    --pdf /tmp/<output>.pdf --company "<Company>"
```

Checks F1-F10: F1 US-Letter page geometry; F2 text-color allowlist (black +
#333333 subtitle only — catches a blue/colored hyperlink or heading); F3 every
span signature matches one the reference uses (catches an un-bolded title, a
heading at the wrong size, a body paragraph that lost its font); F4 14pt-bold
title; F5 #333333 subtitle shape; F6 13pt-bold `First-Pass Diligence — <date>`
anchor; F7 NO `[N]`/`[N,M]` citation rendered at body size (the
superscript-regression bug); F8 table-cell fill + grid/underline stroke colors
within the reference set (catches a colored table background); F9 heading
underlines present on the levels the reference always underlines; F10 page
numbers centered. The profile is MEASURED from the approved Factir vFinal PDF
(`templates/master-diligence-doc/reference_profile.json`), not hand-typed, so
the gate's expectation cannot drift from the real reference. (Paragraph spacing
+ indents are deliberately NOT gated — multi-valued in a rendered PDF, would
false-positive.)

**Exit 0 = proceed to Step 6. Exit 1 = STOP. Do NOT upload to Drive.** Fix the
builder and rebuild. Regenerate the profile only when Tom re-approves a changed
reference artifact (see `templates/master-diligence-doc/reference.json` →
`profiler.regenerate`).

---

## Step 6: Upload to Drive, Link in Notion, and Send Alert

Upload the PDF to Drive first so the Drive URL is available to include in the alert, then link
it in Notion and send the alert with the URL (no file attachment).

### 6a. Upload PDF to Google Drive and Link in Notion

Upload the generated PDF to the company's opportunity folder on Google Drive, then add the
Drive link to the Diligence Materials field on the Notion opportunity page. This makes the
PDF accessible alongside the other diligence materials (deck, memo, one-pager) in a single place.

**Uploading:** Use the Google Apps Script endpoint for all Drive operations. **NEVER use Zapier** — it is deprecated and unreliable. Read the full reference at `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`.

**MANDATORY — always create a NEW Drive file. Never overwrite an existing one in place.** Even when re-running for an Opp that already has a prior first-pass PDF, upload as a fresh file with a new Drive file ID. The reason: Notion caches PDF previews keyed on the URL/file ID. If you `files().update()` the existing file's content, the file ID stays the same, the URL stays the same, and Notion keeps showing the OLD cached preview no matter what's actually in Drive — observed 2026-04-27 on the Inlets re-run. A fresh file ID forces Notion to fetch a fresh preview. Tom can manually delete the stale prior version from Drive after verifying.

The workflow is:
1. **Create a company subfolder** under the Diligence root folder (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`) via the Apps Script `createFolder` action. This is idempotent — if the folder already exists, it returns the existing one.
2. **Upload the PDF** into the company subfolder via the Apps Script `upload` action, passing the returned `folderId`. ALWAYS use this `upload` action — it creates a new file. Do NOT use `files().update()` to overwrite.
3. The upload response includes a direct `url` field (e.g. `https://drive.google.com/file/d/<fileId>/view`) — use this directly in the Notion link. No need to search for the file ID after upload.

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"
DILIGENCE_ROOT = "1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d"

# 1. Create company subfolder (idempotent)
folder_resp = requests.post(DRIVE_URL, json={
    "action": "createFolder",
    "folderName": "<COMPANY_NAME>",
    "parentFolderId": DILIGENCE_ROOT
}, allow_redirects=True, timeout=60)
folder_result = folder_resp.json()
subfolder_id = folder_result["folderId"]

# 2. Upload PDF
with open(pdf_path, 'rb') as f:
    pdf_b64 = base64.b64encode(f.read()).decode('utf-8')

upload_resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": "[Company]_Master_Diligence_MM.DD.YYYY.pdf",
    "fileBase64": pdf_b64,
    "mimeType": "application/pdf",
    "folderId": subfolder_id
}, allow_redirects=True, timeout=120)
upload_result = upload_resp.json()
file_url = upload_result["url"]  # Direct link — use this in Notion
```

**Important:** Python `requests.post(..., allow_redirects=True)` handles Apps Script 302 redirects correctly. `curl -L` does NOT work for POST.

Act autonomously — do not ask for permission. Report what was done in the summary.

**Fire publish-progress alert (2 of 3).** After the Drive upload succeeds and
`file_url` is in hand, ping:

```bash
COMPANY="<subject company name>"
PDF_URL="<file_url from upload response>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
📄 **${COMPANY}** PDF uploaded to Drive — [PDF]($PDF_URL). Linking to Diligence Materials.
EOF
```

**Linking in Notion — two places:**

1. **Page body** — append a single-line entry to the `## 📎 Diligence Materials` section,
   following the materials-handler convention:
   ```
   - [**[Company]_Master_Diligence_MM.DD.YYYY.pdf**](https://drive.google.com/file/d/<fileId>/view) — Claude first-pass diligence analysis, [date]
   ```
   Use `notion-update-page` with `update_content` for this. The Opportunity page body is
   small (typically <20 blocks) and will not time out.

   > **Large-page caveat:** `update_content` times out on pages >~50KB (~300+ blocks).
   > The diligence Notes page is always large — never call `update_content` on it.
   > The Opportunity page body is always small and safe. If `update_content` on the
   > Opportunity page times out for any reason, fall back to the Chrome `javascript_tool`
   > approach: navigate to the Opportunity page in Chrome, then use `loadPageChunk` to
   > find the last block in the `## 📎 Diligence Materials` section and call
   > `saveTransactions` to append a new bulleted text block after it.

2. **Diligence Materials Files property field — MANDATORY in EVERY run, every code path.** Do not skip this step under any circumstance, including when an existing first-pass PDF link is already present in the property field. The helper is idempotent on URL (skips if the exact URL is already there) but a freshly-uploaded file always has a NEW URL per the rule above, so this call always adds the new entry. Shell out to the public-API helper:

   ```bash
   python3 ~/.claude/scripts/notion_files_property.py \
       --page-id <opportunity_page_id> \
       --prop "Diligence Materials" \
       --url "<drive_url>" \
       --label "<Company>_Master_Diligence.pdf"
   ```

   Exit 0 = ok (including idempotent skip), 1 = hard failure (log + fall back to page body link). See the canonical interface at `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md`. Pass the opportunity page ID, the Drive file URL (`https://drive.google.com/file/d/<fileId>/view`), and a display name like `[Company]_First_Pass_Diligence.pdf`.

   The helper uses the public Notion API (PATCH `/v1/pages/{id}` with the Files property's `files` array). No Chrome dependency.

   **MANDATORY verification — never trust the 200 response alone.** Immediately after the PATCH, re-fetch the Opportunity page and scan the `Diligence Materials` files array for an entry whose `external.url` matches the Drive URL you just wrote. If not present, the write silently failed (observed on Factir 2026-05-15 — PATCH returned 200 but Notion kept the prior URL in the files array). Re-try the PATCH with an explicit replacement payload (read the full files array, mutate, PATCH the whole array), then re-verify. If verification still fails after 3 retries, surface to Tom in the publish summary — do NOT publish silently. Reference snippet:

   ```python
   opp = json.loads(urlopen(Request(f"https://api.notion.com/v1/pages/{OPP_ID}", headers=HDR)).read())
   urls = {f.get("external",{}).get("url") for f in opp["properties"]["Diligence Materials"]["files"]}
   assert drive_url in urls, f"Property write did NOT take — URL {drive_url} not in {urls}"
   ```

**Fire publish-progress alert (3 of 3).** After the Diligence Materials
property write is verified, ping:

```bash
COMPANY="<subject company name>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
🔗 **${COMPANY}** Diligence Materials property updated. Sending completion alert.
EOF
```

### 6b. Send the alert

Read the `send-alert` skill (`**/send-alert/SKILL.md`) for delivery and format conventions.
Send the alert as **exactly 3 lines**. No header line, no blank lines between lines, no
trailing flags. The third line is the feedback prompt that closes the learning loop —
Tom's thread replies are routed through `claude-alerts-listener` and appended to
`FEEDBACK_PATTERNS.md` (loaded in Step 1f of future runs).

**Alert body — exactly 3 lines. Use GFM `[text](url)` link syntax, NEVER Slack mrkdwn `<url|text>` syntax** (the converter passes Slack mrkdwn through as literal text inside the underline+bold wrapper, which is what produced broken alerts in the past):

Template (substitute the three values directly — do not keep angle brackets around the placeholders):

```
🪏 <u>**First Pass Diligence: [COMPANY_NAME](OPP_URL) ([PDF](PDF_URL))**</u>
ONE_LINER_SUMMARY
💬 Reply in thread with any takeaways for next time.
```

Concrete example of the rendered body that should be piped into `send.sh`:

```
🪏 <u>**First Pass Diligence: [Shine](https://www.notion.so/35700beff4aa8147b93ede0f63694110) ([PDF](https://drive.google.com/file/d/1mjbovMWmrjdYWCqc6RNnlbhHfuxFxdHW/view))**</u>
Moderate founder pair; wedge is real but UserEvidence and Listen Labs are 10-50x better-capitalized.
💬 Reply in thread with any takeaways for next time.
```

Conventions:
- **Line 1 — bolded title with two links.** `🪏` (pickaxe emoji) outside the wrapper, then `<u>**...**</u>` wrapping the title `First Pass Diligence: COMPANY (PDF)`. The company name is hyperlinked to the Notion Opportunity URL (NOT the analysis page URL). The literal text `PDF` (in parens) is hyperlinked to the Drive PDF URL returned from step 6a.
- **Line 2 — one-liner summary.** Plain text, no bold, no bullets, no links. 1–2 sentences max — what Tom needs to know before clicking through. Lead with the most important signal (e.g., "Strong founder fit + obvious market, but $3M seed is later than my typical entry — fund-fit pass.").
- **Line 3 — feedback prompt.** Literal text `💬 Reply in thread with any takeaways for next time.` Do not modify or personalize. The static prompt is the trigger `claude-alerts-listener` keys on for routing first-pass feedback to `FEEDBACK_PATTERNS.md`.

If the lint or audit surfaced findings that publish-proceeded with caveats, append a fourth line (after the feedback prompt) starting with `⚠️ ` and naming the count and category — e.g. `⚠️ Audit flagged 2 untraced claims; see Notion page note for details.`

If for any reason the PDF upload (step 6a) failed and only the Notion analysis exists, replace `([PDF](PDF_URL))` on line 1 with `([analysis](NOTION_ANALYSIS_URL))` so the user always has one click-through. Do NOT skip the alert.

---

## Quality Bar

A good first-pass diligence analysis:

- Cites specific evidence from the diligence record for every major claim
- Flags every instance where founder representations diverge from independent findings
- Includes data tables that make information-dense sections scannable
- Grounds open questions in research rather than leaving them as generic placeholders
- Makes the case for and against the investment with equal rigor
- Is explicit about where the evidence base is thin and what additional context would help
- Uses substantive, multi-paragraph prose throughout — not bulleted lists of observations
- Cross-references Inverted portfolio memos in the Framework Mapping section
- Follows the exact Risk Summary table format (Failure Mode | Key Evidence | Kill Shot?)
- Includes all four Path to Next Round subsections with the prescribed analytical content

A bad first-pass diligence analysis reads like a pitch deck summary or a generic VC memo.
If you find yourself restating the company's own narrative without interrogating it, stop
and go back to the source materials.

**Reference output:** The Lex Financial First-Pass Diligence (March 27, 2026) represents the
quality bar for this skill. When in doubt about formatting, depth, or analytical voice, use
that output as the benchmark.
