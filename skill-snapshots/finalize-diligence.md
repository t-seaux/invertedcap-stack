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

**Subagent C₁ — Preview Builder (Sonnet OK).** Owns Step 7 (PDF build — operates on a
local consolidated markdown snapshot built from the new FA draft + the existing page
content, NOT a post-Notion-prepend re-fetch) + Step 7.5 (Preview Gate — Slack ping +
HALT). Inputs: verified draft + audit JSON + link map. Outputs: preview PDF in iCloud
Downloads + Slack approval request. Does NOT touch Notion or canonical Drive yet.

**HARD GATE.** Subagent C₁ halts after posting the preview. The parent waits for Tom's
explicit `publish <company>` reply in chat (or equivalent). On `revise <company>
[feedback]`, parent routes back to Subagent A with the feedback. On `publish`, parent
dispatches Subagent C₂.

**Subagent C₂ — Publisher (Sonnet OK).** Owns Step 4 (Notion prepend — uses REST
canonical per memory `feedback_mcp_insert_content_ordering_bug`) + Step 8 (canonical
Drive upload + retention sweep + Notion link patches) + Step 9 (final Slack alert).
Inputs: the approved preview PDF + verified draft. Pure mechanical publish, no
re-drafting. The canonical Drive upload REUSES the same PDF bytes Subagent C₁
generated — no rebuild needed.

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

### MANDATORY — drafting runs on Opus tier

Per memory `feedback_model_tier_framework`: Opus = no-rubric + final-artifact tasks.
The Final Assessment is exactly that — a synthesis call with no scoring rubric and a
delivered artifact. The audit-judge gate in Step 6 is Sonnet (rubric-based review), but
drafting itself must be Opus.

- **If you're in an interactive Opus session** (this file's parent context is Opus 4.7
  or later): proceed inline; you're already on the right tier.
- **If you delegate to a subagent** for bundle assembly + drafting (the recommended
  pattern for context budget reasons), pass `model: "opus"` to the Agent tool
  explicitly. `subagent_type: "general-purpose"` inherits the parent's model — if the
  parent is somehow Sonnet, the subagent silently degrades.
- **If you're in a Sonnet session and can't promote to Opus**: stop, tell Tom the
  drafting tier is wrong, and ask him to re-invoke from an Opus session. Don't
  silently produce a Sonnet-quality Final Assessment.

### Format

The Final Assessment is a standalone artifact. A reader who lands directly on this block
(LP, future Tom, a coinvestor, a partner) must be able to read it cold and understand the
company, the bet, how conviction got built, and what residual risks the investment is
accepting. No section assumes the reader has read the rest of the Master Diligence Doc.

```markdown
# Final Assessment — [Month Day, Year]

### Overview

[1-2 paragraphs of pure context, no thesis, no commentary. Cover:
- What the company does (one-sentence positioning)
- The product — wedge today and the roadmap to the end state where applicable
- The articulated end state for the business (what does this look like at scale)
- The founder (one-line who, with the operator-grade reputation anchor)
- The round — stage, size, terms, Inverted check size + ownership

Lift from the original first-pass `Context > Company Overview` as the starting frame, but
refresh for current state: ICP narrowing surfaced across updates, traction milestones,
final round structure (the original first-pass round structure is often stale by finalize
time). This section is factual setup — every claim in it is in scope for the audit gate.]

### Thesis

[Opens with the underlying rationale for investing in this business — the why. This is
the bet articulated as a forward-looking statement: what makes this company worth
backing if it works. Read it as standalone prose that a coinvestor could excerpt
verbatim. No reinforcement language ("our prior was confirmed"), no diligence meta
("the May 15 call moved us") — those belong in the Diligence Journey. Pure forward bet.

**Use multiple paragraphs if needed.** 1-3 short paragraphs is the right shape — never
one long dense paragraph cramming every clause together. If a single sentence would run
past ~30 words to fit everything in, split it. If two distinct ideas (e.g., the wedge
mechanics + the moat compounding mechanic) need their own beats, give each its own
paragraph. The multi-paragraph option is the natural defense against the
one-mega-sentence anti-pattern; use it.

Then break out into the load-bearing claims as a numbered list under a `**Need to
Believes**` standalone bold label (no colon, plural — this is the canonical shape; do
NOT emit "Need to Believe:" or "Needs to Believe"):

**Need to Believes**

1. [NTB-1 — one or two declarative sentences stating a claim the investment is
   underwriting. Single sentence is the default; split into two when a single
   sentence would compound multiple independent ideas with conjunctions and
   become run-on. This is a judgment call — no character/word limit. Prefer
   crisp over comprehensive: each NTB should be skimmable in one read.]
2. [NTB-2 — same shape]
3. [NTB-3 — ...]

Each NTB is a "this must be true for the venture-scale outcome to materialize" statement.
Anchor to the original first-pass NTBs as the spine, but refine where the diligence
record has sharpened or recast them. The NTBs ARE the thesis in structured form —
the Thesis prose is the synthesized version, the NTBs are the testable claims.]

### Diligence Journey

[Multi-paragraph narrative. NOT a chronological regurgitation. This is the COMMENTARY
on how the diligence record fed into the Thesis + NTBs above. Organize by NTB. The
preamble paragraph introduces the section in 1-2 sentences — DO NOT include qualifying
language like "rather than by Update date" or other meta-comparisons. Just say the
record is organized by NTB and what the commentary covers.

For each NTB, emit a `#### NTB-N — [Short Title]` H4 anchor, followed by 1-3 short
bullet points (NOT prose paragraphs) that visually break up each block:

- **Going-In View** — the initial framing from the deck, first call, intro context;
  what we thought before diligence touched it.
- **Diligence Findings** — the specific calls / references / materials / Drive docs
  that bore on this NTB, cited inline with `[N]` footnotes. The label is direction-
  neutral: findings can support, weaken, sharpen, or contradict the going-in view —
  the bullet text describes which. Use the verbatim phrase the source delivered when
  it carried weight ("`Tushar's 'either a COO equivalent or a CPO equivalent'`" beats
  paraphrase).
- **Net Take** — the synthesis after diligence: confirmed / sharpened / softened
  / recast / refuted, with one-line justification. The Thesis prose above is the
  long-form articulation of the same read.

The bulleted shape is required, not optional. It makes each NTB's arc skimmable and
prevents the section from collapsing into one long block of prose. Each bullet can be
1-3 sentences; collectively a single NTB's commentary should run 100-300 words.

The point is to make the reader understand the arc of conviction-building, not to recap.
A claim like "the AtoB reference cohort verified an operator-grade reputation that the
deck and intro could only assert" belongs here — it's commentary on what the references
DID for the underwrite, not a chronological log of who said what.

Quote evidence specifically where it carried weight. Tables are encouraged for
quantitative arcs (e.g., before/after on ACV math, ICP narrowing, revenue build). Every
factual claim about who said what or what a Note contains must be cited and is in scope
for the audit gate; the synthesis commentary about how that evidence shaped the thesis
is excluded from audit (it's interpretation, not assertion).]

### Standing Open Questions

- [Bulleted residual unresolved questions at the time of finalization]
- [Each question is a real risk the investment is accepting, not framework boilerplate]
- [3-5 sharp ones beats ten cosmetic ones]

### Footnotes for this Section

^N Source Label — Brief description / specific quote
^N+1 ...
```

**No Decision / Disposition section.** The investment call is Tom's — flipped via the
Opp Status in the Opportunities DB, not stated inside the Final Assessment. This skill
documents the thesis, the journey, and the residual risks; it never makes or restates
the call. Memory `project_committed_is_pipeline_not_portfolio` applies: Status flips
stay with Tom.

### Header / formatting contract — MANDATORY, DETERMINISTIC

The Final Assessment block MUST emit headers exactly as below. The PDF builder
(`shared-references/pdf_builder_template.py`) renders these per the canonical
formatting in `shared-references/label-hierarchy.md`. Deviations break the
visual hierarchy Tom workshopped.

| Markdown emitted | Renders as | Use for |
|---|---|---|
| `# Final Assessment — Month DD, YYYY` | H0: 13pt bold UNDERLINED | The single H1 anchor at top |
| `### Overview` / `### Thesis` / `### Diligence Journey` / `### Standing Open Questions` / `### Footnotes for this Section` | H2: 11pt bold UNDERLINED | All five subsection headers |
| `#### NTB-N — [Short Title]` (under Diligence Journey) | H3: 10.5pt bold-italic UNDERLINED | Per-NTB anchors inside Diligence Journey |
| `**Need to Believes**` (standalone line inside Thesis section) | h2-style bold (no underline) | The NTB sub-label inside Thesis prose |
| `***Open Questions:***` inline | italic-only (no bold) | First-pass §X-style callouts — N/A in Final Assessment block usually |
| `**Label.**` paragraph-leader (e.g. `**Killshot 1 — …**`) | body-bold inline | Leaf labels inside body prose |

**Don't emit:**
- `**bold standalone line**` as a section header — use `###` instead (legacy h2-style without underline collapses against `**Need to Believes**`).
- `## ` (H1 underlined) inside the Final Assessment block — reserved for `## Update — …` and other top-level anchors elsewhere on the page.
- Trailing periods on headers (`### Thesis.` is wrong; `### Thesis` is right).
- Em dashes (`—`) inside any header except the `# Final Assessment — Month DD, YYYY` H1 anchor. Use en dashes (`–`) elsewhere.

### Voice & citation rules

**Canonical voice source: `~/.claude/skills/writing-style/letters-and-memos/STYLE.md`
+ `~/.claude/skills/writing-style/letters-and-memos/VOICE_EXAMPLES.md`.** Read both at
the start of every drafting session. The Final Assessment is long-form analytical
prose — same voice that the LP letters and investment memos use. The stylebook is the
single source of truth on tone, sentence shape, hedge avoidance, and concrete-language
expectations. Memory `feedback_be_opinionated_thought_partner` reinforces it.

- **Direct prose, no hedging.** "This NTB is confirmed by …" beats "There is some evidence
  suggesting this NTB may be confirmed by …". Call hard reads plainly; well-rounded mediocrity is a negative, not neutral.
- **No empty qualifiers.** Banned phrasings include but are not limited to:
  *structurally honest*, *venture-scale outcome*, *load-bearing*, *category-defining*,
  *first-mover advantage*, *in-flow-of-funds position*, *underwrite the bet*, *flywheel*.
  Test: if you can delete the qualifier and the sentence still makes sense, the
  qualifier was empty — delete it. The reader gets nothing from abstract scaffolding
  that doesn't communicate a specific fact, mechanism, or threshold. When in doubt,
  name the actor (the founder by name), the threshold (50-100 customers), or the
  competitor (Flexport / Wise / Airwallex) instead of reaching for a Latinate
  abstraction. The writing-style stylebook above governs; this list is the active
  watch-list of phrasings Tom has rejected in prior runs.
- **Citations are digit-only `[N]`, multi-source `[N,M]` with NO space after the comma.**
  Forbidden: `[N, M]`, `[N] [M]`, `[N-M]`, `[uN]`/letter-prefix. New footnotes in the Final
  Assessment continue numbering from the highest existing `^N` on the page (do NOT restart
  at `^1`). If the page's existing high-water-mark is `^14`, the Final Assessment's first new
  footnote is `^15`.
- **Refer to sections by name, never `§N` shorthand.** "The original Product section" / "the
  May 15 update's Prior Assessment on regulatory risk" — never `§2.1` or `§U2`.
- **Em dashes only in the H1 anchor.** Prose uses en dashes `–`. The only allowed em dash is
  in `# Final Assessment — Month DD, YYYY`.
- **Numerical ranges use hyphens.** `$30K-$80K`, not `$30K–$80K`.
- **Quantitative claims must be inline-cited.** Per memory
  `feedback_quant_claim_citation_discipline`: founder docs > Inverted internal refs > external
  public > Claude estimate. Tag every cell.
- **Speaker attribution in transcripts must trace to the actual speaker.** When Tom raises
  a framework, analogy, push-back, or industry parallel in a call transcript and the founder
  agrees (or extends the point), the framework/analogy belongs to *Tom*, not the founder.
  Misattributing Tom's reframings to the founder inflates the founder-evaluation signal and
  misleads the next reader (often Tom himself) who treats the framing as founder-originated.
  Mandatory pre-submit check: for every sentence in the draft attributing an analogy,
  framework, or strategic reframing to a named person, verify against the transcript who
  *introduced* the framing vs. who *agreed with* it — the introducer owns the attribution.
  Tom-as-questioner is a high-frequency analogy-introducer in his transcripts; default
  toward Tom-introduction unless the transcript clearly shows the founder bringing it
  first. Memory: `feedback_transcript_speaker_attribution_to_tom`.
  Concrete prior incident: Alongside Medical first-pass 2026-06-06 attributed the
  Brex-default-expense-policy analogy to Sonia ("Sonia's Brex analogy", "she described it
  without prompting") when in fact Tom introduced the analogy after Manish described the
  underlying baseline-plus-gold content layer, and Sonia simply agreed.

### What this block must NOT do

- Include a Decision / Disposition section. Tom flips Status separately.
- Re-narrate the original first-pass in the Overview. The Overview is 1-2 paragraphs of
  refreshed context, not a synopsis of the prior diligence prose.
- Make the Diligence Journey a chronological recap. It's commentary on how the record
  shaped the thesis. If you find yourself writing "Update #1 covered X, then Update #2
  covered Y", you're writing a log, not a journey — restructure by NTB or signal
  cluster, not by Update date.
- Pad the Standing Open Questions with hypothetical risks that no one is actually worried
  about. Three real residual questions beats ten cosmetic ones.

---

## Step 4: Replace Existing Final Assessment (if any) + Prepend New (Subagent C)

Tooling choice — REST is canonical for both delete and prepend. MCP is a narrow fallback only.

- **REST API direct (PATCH/POST/DELETE via `urllib`)** — canonical for delete + prepend + title
  PATCH + listing children. Token resolution via SOPS, per the snippet in
  `update-diligence-priors/SKILL.md` Step 5.
- **MCP `notion-update-page` with `insert_content`** — fallback ONLY when the new Final
  Assessment markdown embeds non-text block types that REST would require hand-building (rich
  tables with merged cells, callouts, image embeds). Plain markdown — headings, paragraphs,
  bullets, plain tables, footnotes, bold/italic — goes through REST.

### 4.1 Delete the existing Final Assessment block range (REST API, parallel)

If Step 1 found an existing `# Final Assessment — …` H1 anchor:

1. List all top-level blocks (paginated, `page_size=100`).
2. Identify the block range: from the `# Final Assessment — …` heading_1 block to (but not
   including) the next H1/H2 anchor (`## Update — …` or `# First-Pass Diligence — …`).
   Include any leading/trailing divider blocks that bracket the Final Assessment.
3. DELETE each block via `DELETE /v1/blocks/{block_id}` in parallel via
   `ThreadPoolExecutor(max_workers=8)`. The prior serial-with-0.15s-sleep pattern took ~11s for
   a 74-block delete; parallel collapses that to <2s.

```python
from concurrent.futures import ThreadPoolExecutor
from urllib.request import Request, urlopen

def _delete_block(block_id):
    urlopen(Request(f'https://api.notion.com/v1/blocks/{block_id}',
                    headers=HDR, method='DELETE'))

with ThreadPoolExecutor(max_workers=8) as ex:
    list(ex.map(_delete_block, [b["id"] for b in to_delete]))
```

**Rate-limit fallback.** If the parallel run hits HTTP 429, retry the failing IDs with a 0.05s
sleep between calls (still 3x faster than the original 0.15s). Notion REST tolerates 8-wide
parallelism on block deletes in observation; tighten only if rate-limit errors surface.

**Destructive op acknowledgment.** Per memory `feedback_always_confirm_before_delete`, the
default for delete operations is to confirm. The exception is when the destructive op is the
prescribed step of an explicitly-invoked skill — this is one of those (Tom said "finalize
diligence", which means "replace the prior Final Assessment block"). Proceed autonomously, but
state the scope in the run summary: "Deleted N blocks from the prior Final Assessment dated
MM.DD.YYYY."

### 4.2 Prepend the new Final Assessment (REST API canonical)

REST is the canonical prepend path. The MCP `insert_content` + `position:{type:"start"}`
combination is **non-deterministic on multi-block markdown ordering** — observed on Factir
2026-05-22 to place 118 blocks in REVERSE order on 2 of 3 attempts. Memory
`feedback_mcp_insert_content_ordering_bug` documents the bug; do not use MCP for prepend.

**Canonical REST prepend pattern.** Notion's `PATCH /v1/blocks/{page_id}/children` endpoint
accepts a `children:[...]` array and an `after` parameter. To prepend (insert as the new first
children), set `after` to the empty string `""` — Notion places the new blocks at the start of
the parent's children list and preserves the order of the `children` array exactly as sent.

```python
import json
from urllib.request import Request, urlopen

# blocks_payload is the list of Notion block JSON objects derived from the
# Final Assessment markdown. Build via your preferred markdown→blocks converter
# (the same logic the MCP wraps, but invoked directly so we control ordering).
payload = {
    "children": blocks_payload,
    "after": "",  # empty string → prepend; ordering of `children` is preserved
}
urlopen(Request(
    f'https://api.notion.com/v1/blocks/{PAGE_ID}/children',
    headers=HDR, method='PATCH',
    data=json.dumps(payload).encode(),
))
```

If a single PATCH would exceed Notion's per-request block limit (100 children per call), batch
in groups of 100 and reverse the iteration order so each batch's blocks land in front of the
previously-prepended ones — net result preserves the original markdown order at the top of the
page.

**MCP fallback (only for complex embeds).** If the Final Assessment markdown contains tables
with merged cells, image embeds, or callouts that the REST hand-builder doesn't handle, use
`notion-update-page` with `insert_content` for those specific blocks only. **Do not use it for
multi-block prepend of plain markdown** — the ordering bug fires on every multi-block payload.

**MCP timeout handling — MANDATORY (only relevant if MCP fallback in use).** Same as
`update-diligence-priors` Step 5.1: if MCP returns `notionhq_client_request_timeout`, do NOT
retry. Wait 5–10s, list children via REST API, verify the new H1 anchor is present. Only retry
MCP if verification confirms the content is absent.

**Post-prepend ordering verification — MANDATORY.** After the prepend lands, list the page's
first 10 top-level blocks via REST and confirm the order matches the FA markdown's H1 → ###
Overview → ### Thesis → ### Diligence Journey → ### Standing Open Questions → ### Footnotes
sequence. If reversed or scrambled, treat as a failed prepend: delete the misplaced blocks
(parallel pattern from 4.1) and re-run the REST prepend. Never silently publish a scrambled FA.

### 4.3 Update the page title (REST API)

PATCH the page title to:

```
[Claude] [Company Name] Master Diligence Doc — MM.DD.YYYY Final
```

(Keep the `Master Diligence Doc` base for Notion search continuity; the suffix flips from
`Update` to `Final`. Do NOT stack date suffixes.)

```python
new_title = '[Claude] <Company> Master Diligence Doc — <MM.DD.YYYY> Final'
payload = {'properties': {'Name': {'title': [
    {'type': 'text', 'text': {'content': new_title}}]}}}
urlopen(Request(f'https://api.notion.com/v1/pages/{PAGE_ID}',
                headers=HDR, method='PATCH',
                data=json.dumps(payload).encode()))
```

### 4.4 Confirm the icon (MCP, only if missing)

If the page icon is not already set to `:claude-color:`, set it now via
`notion-update-page` with `icon: ":claude-color:"`. If already set (likely — first-pass
sets it on creation), skip.

**Fire publish-progress alert (1 of 3) — once the Final Assessment is prepended
and the icon is confirmed.** The publish phase is silent for ~15 min between
the audit and the final completion alert; these three pings make it legible.

```bash
COMPANY="<subject company name>"
NOTION_URL="<diligence page URL>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
📝 **${COMPANY}** Final Assessment prepended — [diligence page]($NOTION_URL). Building consolidated PDF next.
EOF
```

---

## Step 5: Pre-PDF Lint — MANDATORY GATE (Subagent B)

After the Notion prepend but BEFORE PDF build, run the deterministic shape-rule lint over the
FULL updated markdown (new Final Assessment + every Update + the inner First-Pass Diligence section).

Write the rebuilt full markdown to `/tmp/<company>_finalized_full.md` and run:

```bash
python3 /Users/tomseo/.claude/skills/finalize-diligence/finalize_lint.py \
    --draft /tmp/<company>_finalized_full.md
```

**Hard gate.** Exit code 0 = proceed. Exit code 1 = STOP, fix the underlying issue, re-run the
lint. The rules it catches all manifest in the published PDF.

The lint covers (F1–F9):

- **F1** — Final Assessment H1 well-formed: `# Final Assessment — Month DD, YYYY` (em dash,
  spelled-out month). Exactly one such anchor on the page.
- **F2** — `### Overview` subsection present inside the Final Assessment block.
- **F3** — `### Thesis` subsection present inside the Final Assessment block.
- **F4** — `### Diligence Journey` subsection present inside the Final Assessment block.
- **F5** — `### Standing Open Questions` subsection present inside the Final Assessment block.
- **F6** — No `§N` / `§N.X` inline section shorthand within the Final Assessment block.
- **F7** — No prose em dashes within the Final Assessment block, except in the H1 anchor.
- **F8** — Numerical ranges use hyphen, not en dash.
- **F9** — Multi-citation form `[N,M]` no-space; no `[N][M]` / `[N] [M]` / `[N, M]` / `[N-M]` /
  letter-prefix.
- **F10** — Final Assessment footnotes `^N` do not collide with `^N` definitions elsewhere in
  the doc (continue numbering, don't restart at `^1`).
- **F11** — No `### Disposition` / `### Decision` subsection present (decision stays with Tom,
  outside the doc — Status flips via the Opportunities DB, not the Final Assessment block).

See `finalize_lint.py --self-test` for the fixture set.

---

## Step 6: LLM Audit Gate — MANDATORY (Subagent B)

Per memory `feedback_research_artifact_self_audit`, long-form diligence artifacts get the
research-artifact-audit discipline before delivery. The Final Assessment is squarely in scope.

Reuse the first-pass audit runner with the finalize-specific judge prompt:

```bash
python3 /Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.py \
    --draft   /tmp/<company>_final_assessment_only.md \
    --sources /tmp/<company>_finalize_sources.md \
    --prompt  /Users/tomseo/.claude/skills/finalize-diligence/finalize_audit.prompt.md \
    --output  /tmp/<company>_finalize_audit.json
```

### Source bundle

Build `/tmp/<company>_finalize_sources.md` with the same `==== SECTION ====` delimiters the
first-pass audit expects. Sections to include:

```
==== ORIGINAL FIRST-PASS MEMO ====
[Full verbatim text of the # First-Pass Diligence — ... block as it currently stands on the page]

==== UPDATE BLOCKS ====
[Each ## Update — ... block in chronological order, separated by --- delimiters; full verbatim text]

==== OPPORTUNITY PAGE ====
[Current Notion Opp page properties + full verbatim page body]

==== LINKED NOTES ====
[For EVERY Notion note in the Opp's ✍️ Notes relation that the Final Assessment cites
in any footnote: emit the note's full verbatim body, including the Notion ID anchor
("Notion ID: <32hex>"), the page URL, the full call transcript if present (always
fetch with include_transcript: true for call/meeting notes), the AI-summary block,
and any structured fields. NOT a pointer manifest. NOT a summary. NOT "see Update #3
for what this call covered." The audit judge compares draft claims against this
section LITERALLY — if the call body isn't here verbatim, every claim citing the call
will be flagged untraced and the audit gate will fail spuriously.]

==== DILIGENCE MATERIALS ====
[Each Drive PDF / Google Doc / DocSend etc. cited in the Final Assessment — full
text extraction, not a pointer or summary. Same standard as LINKED NOTES.]

==== INVESTMENT MEMO MANIFEST ====
[The Step 2.5 manifest used by first-pass — JSON list of every memo title + Drive URL + first
~500 chars of each, so the judge can verify "as Tom wrote in the Tuor memo" type claims]
```

**The pointer-manifest failure mode (caught Factir 2026-05-23).** Subagent A built a
LINKED NOTES section that listed `^N = Person Name (call date)` pointers with prose like
"Each note's contents are characterized in detail within the Update block's prose."
The audit then flagged 16 claims as untraced because the actual call-note text wasn't
in the bundle for the judge to find. The bug isn't the draft — it's the bundle. Memory
`feedback_finalize_diligence_bundle_verbatim_mandate` documents the failure mode.

### Source bundle completeness check — MANDATORY GATE before audit

The bundle completeness gate is canonical in `research-artifact-audit` Step
A.1 — read that for the full spec. The finalize-diligence binding is:

```bash
python3 ~/.claude/skills/shared-references/bundle_completeness_check.py \
    --draft  /tmp/<company>_final_assessment_only.md \
    --bundle /tmp/<company>_finalize_sources.md \
    --notes-section 'LINKED NOTES' \
    --required-sections 'LINKED NOTES,DILIGENCE MATERIALS,ORIGINAL FIRST-PASS MEMO,UPDATE BLOCKS' \
    --self-page-ids <Master Diligence Doc page ID, 32hex no hyphens>
```

The `--self-page-ids` argument is REQUIRED for finalize — the Final Assessment
often cites Update sections via the page's own URL (e.g., footnote ^N pointing
at `https://notion.so/<page_id>` for "Update #3 on this page"). Without
exempting the self-page-id, B2 will false-positive on those self-references.
Look up the Master Diligence Doc page ID from Step 1's notion-search result.

Exit 0 = proceed to audit. Exit 1 = bundle is missing verbatim content. STOP,
rebuild, re-run. The audit verdict on a broken bundle is structurally
meaningless.

### Draft to audit

Build `/tmp/<company>_final_assessment_only.md` with JUST the new Final Assessment block (not
the consolidated full doc — the audit's scope is the new prepended content only; the original
+ updates have already been audited by their own runs).

### Iteration loop

Same as first-pass Step 4b: exit code 0 = proceed. Exit code 1 = at least one `untraced` claim
— fix the offending claim in the Notion block (re-prepend if needed) and re-run. Max 3
iterations. If after 3 iterations untraced claims persist, append a publish-summary surface
warning: `⚠️ Audit: <N> untraced after 3 iterations`.

### MANDATORY — verify the audit JSON before declaring this step done

After every audit invocation, **before** proceeding to Step 7 or reporting anything about
the audit result, run:

```bash
jq '.summary' /tmp/<company>_finalize_audit.json
```

Show the literal output (`{"total_claims_audited": N, "traced": X, "partial": Y, "untraced": Z}`)
in the run trace. If `untraced > 0`, also dump the offending claims:

```bash
jq -r '.claims[] | select(.verdict == "untraced") | "[" + .claim_type + "] " + (.claim_text | .[0:160]) + "\n  -> " + (.notes // "")' /tmp/<company>_finalize_audit.json
```

**Never self-report "audit clean" without the jq summary in hand.** Caught after Factir
2026-05-22: subagent claimed `0 untraced, 47 claims, 1 iteration` while the JSON on disk
showed `8 untraced + 12 partial out of 48`. Verify, don't trust. Memory
`feedback_skill_self_report_diverges_from_actual_write` applies in full.

### Chunking gotcha

The audit script auto-chunks bundles over 350KB (see `AUTO_CHUNK_THRESHOLD` in
`first_pass_audit.py`). For typical diligence runs (50-300KB), it runs un-chunked in one
Sonnet pass — accurate and avoids the cross-chunk merge gap that produced false-positive
untraceds on Factir's first finalize run. If your bundle is over 350KB and the audit
returns suspicious untraceds, sanity-check by re-running with `--chunk-size 0` (forces
single un-chunked pass); if untraceds disappear, the chunking merged poorly and the
un-chunked verdict is ground truth.

The judge prompt at `finalize_audit.prompt.md` has the same claim-type inclusion list as
first-pass but with these specific scope rules for the Final Assessment surface:

- **Overview** is IN SCOPE — every factual claim about what the company does, the
  product, the founder, the round terms must trace to the source bundle.
- **Thesis prose** is EXCLUDED (synthesis statement, not assertion).
- **Need-to-Believe items** are EXCLUDED (they're forward-looking "must be true"
  claims, not factual assertions to verify).
- **Diligence Journey** is MIXED — factual claims about who said what / what a Note
  contains / what an Update found ARE in scope; commentary on how that evidence
  shaped the thesis is excluded (it's interpretation, not assertion).
- **Standing Open Questions** are EXCLUDED (forward-looking risks, not factual
  claims).

---

## Step 7: Build the Consolidated PDF (Subagent C — runs in parallel with Step 6)

**Parallelism.** Step 7 (PDF build) operates on the consolidated page state established by Step
4 (Notion prepend), but Step 6 (audit) operates on the FA draft only. Both are downstream of
Step 4 / Step 5, but neither depends on the other. Run them concurrently:

- Subagent B fires Step 6 audit on `/tmp/<company>_final_assessment.md` + sources.
- Subagent C fires Step 7 PDF build on the post-prepend consolidated page in parallel.

If Step 6 fails iteration (untraced > 0 after max 3), discard the in-progress (or completed)
PDF; once the FA fix lands, rebuild. Net savings: ~3-5 min on typical runs where the audit
passes first iteration and the PDF was being built concurrently.


Generate `[Company]_Master_Diligence_MM.DD.YYYY_vFinal.pdf` containing:

1. The new Final Assessment block (top of the document).
2. Every `## Update — …` block, in chronological order (newest first), each on its own page.
3. The `# First-Pass Diligence — …` H1 anchor and the full original analysis below it.

### Filename

The finalized PDF is always written as
`[Company]_Master_Diligence_MM.DD.YYYY_vFinal.pdf`, where `MM.DD.YYYY` is today's date
(the finalize date). There is no version number — there is only ever ONE `_vFinal.pdf`
per company in Drive at a time. Re-finalize overwrites the prior `_vFinal.pdf` (handled
by the retention sweep in Step 8 — trash the old, upload the new).

Examples:
- First-ever finalize: `Tuor_Master_Diligence_05.22.2026_vFinal.pdf`.
- Re-finalize 2 weeks later (after another `update-diligence-priors` run that produced
  `_v4.pdf`): the new finalize produces `Tuor_Master_Diligence_06.05.2026_vFinal.pdf` and
  trashes the prior `_v4.pdf` + prior dated `_vFinal.pdf` files.

The original first-pass date is dropped from the filename entirely (consistent with
`update-diligence-priors`).

### Content + page breaks

Use the canonical PDF templates at
`~/.claude/skills/shared-references/pdf_builder_template.py` or
`pdf_builder_from_md_template.py` — both implement automatic page breaks via `parse_content()`.
The template inserts `PageBreak()` before every:

- `# Final Assessment — …` heading (top of doc — no break needed there, but template handles
  the no-leading-break case).
- `## Update — …` heading.
- `# First-Pass Diligence — …` heading.

Per memory `feedback_pdf_builder_multicitation_artifacts` and
`feedback_pdf_builder_formatting_watertight`: confirm the template's regex handles
multi-citation `\s?\[(\d+(?:,\s*\d+)*)\]`, applies `\[N\]` / `\$` / `\^N` cleanup, uses
strict-YAML frontmatter regex, and runs `_strip_unrenderable_chars()`. If you're calling
`pdf_builder_from_md_template.py`, these MUST be active or the output ships with literal
`[13,14]` and `\$620B` artifacts.

### PDF header

- **Title (14pt bold):** `[Company Name] — Master Diligence Doc`
- **Subtitle (9pt italic, dark gray):** `Latest Update: [Today's Date] | First-Pass:
  [Original First-Pass Date] | Notion` (where "Notion" is a clickable hyperlink to the
  page URL)

The "Latest Update" date is today's date — when finalize runs, the Final Assessment IS
the latest update on the doc, so there's no need to distinguish them. Drop the
"Initially Published" field; "First-Pass" covers it more clearly.

### Page numbers + clean_text

Same as `update-diligence-priors` Step 5b: centered 8pt page numbers via canvas callback;
all prose through `clean_text()` for `***`/`**`/`*`/`[text](url)` → reportlab XML conversion.
Read the formatting spec at
`/Users/tomseo/.claude/skills/shared-references/long-form-pdf-spec.md`.

Save the PDF to:
`/sessions/loving-modest-fermat/Users/tomseo/Downloads/[Company]_Master_Diligence_MM.DD.YYYY_vFinal.pdf`

### Deterministic publish gates — MANDATORY before Drive upload

Two guards must pass between extraction and Drive upload. Both target failure
modes observed on Factir 2026-05-23 (extractor leaked `Top-level blocks: 531`
debug text into the rendered PDF body; subtitle shipped in the old verbose
`Final Assessment | Latest Update | Initially Published` format that confused
readers).

**Gate 1 — markdown sanity** (runs after extractor, before PDF build):

```bash
python3 ~/.claude/skills/shared-references/markdown_sanity_check.py \
    --markdown /tmp/<company>_full.md \
    --expected-h1-prefix '# Final Assessment'
```

Exit 0 = proceed to PDF build. Exit 1 = the extractor emitted debug output
(`Top-level blocks:`, `DEBUG:`, `Found N notes`, etc.) into the markdown body
OR the first line is not the expected H1. Fix the extractor's `print()`
statements (move to `sys.stderr` or remove) and re-extract.

**Gate 2 — PDF header check** (runs after PDF build, before Drive upload):

```bash
python3 ~/.claude/skills/shared-references/pdf_header_check.py \
    --pdf ~/Library/Mobile\ Documents/com~apple~CloudDocs/Downloads/<Company>_Master_Diligence_<MM.DD.YYYY>_vFinal.pdf
```

Exit 0 = title and subtitle match canonical shape. Exit 1 = subtitle drift
(wrong field set or wrong ordering — e.g., shipping with `Final Assessment:` or
`Initially Published:` in the subtitle) OR debug-pattern leak detected on the
first page. Fix the build script's subtitle string and re-run; do NOT upload
the PDF until this gate passes.

Both gates use defaults aligned with the canonical Master Diligence Doc shape
(title `^[A-Z].+ — Master Diligence Doc$`, subtitle `^Latest Update: .+ \|
First-Pass: .+ \| Notion$`). For other artifact families, pass
`--title-pattern` / `--subtitle-pattern` overrides.

**Gate 3 — PDF format guard** (runs after PDF build, before Drive upload):

```bash
python3 ~/.claude/skills/shared-references/pdf_format_guard.py \
    --pdf ~/Library/Mobile\ Documents/com~apple~CloudDocs/Downloads/<Company>_Master_Diligence_<MM.DD.YYYY>_vFinal.pdf \
    --company "<Company>"
```

Re-extracts the rendered PDF with PyMuPDF and asserts every text-span's
`(font, size, color)` signature against the MEASURED Factir reference profile
(`templates/master-diligence-doc/reference_profile.json`). Catches what the
header check can't: a heading rendered at the wrong size, a body paragraph
that lost its font, a blue/colored hyperlink (spec is black-only), a wrong
page size, and — critically — a `[N]`/`[N,M]` citation rendered as literal
body text instead of a superscript (the Upskill 2026-05-20 regression).
Checks F1-F10 (adds F8 table-cell fill + stroke colors, F9 heading underlines,
F10 centered page numbers on top of the typography/anchor checks; paragraph
spacing + indents are deliberately not gated — they false-positive in a rendered
PDF); exit 0 = publishable, exit 1 = STOP, do not upload. The profile
is MEASURED from the approved Factir vFinal PDF, not hand-typed, so the gate's
expectation cannot drift from the real reference. Regenerate the profile only
when Tom re-approves a changed reference artifact (see
`templates/master-diligence-doc/reference.json` → `profiler.regenerate`).

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
