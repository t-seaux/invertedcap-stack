---
name: memo-workshop
description: >-
  Bootstrap a live co-writing session on an EXISTING investment memo Google Doc: load every layer of
  context (the memo doc, Notion Opp, Master Diligence final assessment, founder meetings + transcripts,
  feedback + reference notes, Drive diligence materials, all reference memos, LP letters + the LPAC
  framework deck, letters-and-memos stylebook) and the Docs editing harness, then iterate on prose
  with Tom turn by turn. Trigger on "work on the
  investment memo for [X]", "let's work on the [X] memo", "iterate on the [X] memo", "memo session for
  [X]", "workshop the [X] memo", "co-write the [X] memo", "load up context for the [X] memo", or a
  Google Doc memo link plus intent to work on it. NOT the initial-draft skill — if no memo Doc exists
  yet, route to draft-investment-memo. Manual-only.
---

# Memo Workshop

Load full deal + voice context and enter an iterative co-writing loop on an existing investment
memo Doc. The deliverable of each turn is sharpened prose in Tom's voice, written to the Doc in
batched edits. This skill is the session bootstrap + iteration conventions; the memo skeleton,
formatting spec, and initial-draft pipeline live in `draft-investment-memo`.

---

## Step 0: Resolve the Opportunity and the memo Doc

1. Find the memo Doc in the canonical Drive folder (`1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`):
   `search_files` with `parentId = '1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO'` and match
   `[Company] - Investment Memo` — match without the `[WIP]` prefix, since drafts are
   named `[WIP] [Company] - Investment Memo` and finalized memos drop it. If Tom pasted
   a Doc URL, use that fileId directly.
2. Find the Notion Opportunity via `notion-search` on the company name.
3. **If no memo Doc exists**, stop and offer to run `draft-investment-memo` instead — that skill
   owns initial drafts (and conversely refuses when a Doc already exists). Never create a new
   memo Doc from this skill.

## Step 1: Load capabilities (before content)

- **Stylebook**: read `~/.claude/skills/writing-style/letters-and-memos/STYLE.md` in full, plus the
  INVESTMENT MEMO entries of `VOICE_EXAMPLES.md`. (The corpus was backfilled in a single pass, so
  entries share a date — "most recent" is moot; load the memo-type annotations, and note any memo
  newer than the backfill (e.g., Factir) is absent from `VOICE_EXAMPLES` and must be read from the
  canonical folder.) All voice, structure, anti-pattern, and self-citation rules live there — do not
  restate them; apply them.
- **Editing harness**: `~/.claude/scripts/google_docs_edit.py` (markdown always from a file path):
  - `replace_section <docId> <headingText> <md>` — swap a section by heading (runs post-write lint)
  - `replace_range <docId> <startMarker> <endMarker> <md>` — markers are paragraph-level and
    **inclusive on both ends**; everything from the start-marker paragraph through the end-marker
    paragraph is deleted and replaced
  - `append_section <docId> <afterHeading> <md>` / `read_preview <docId>`
  - Pre-write lint (em dashes, escapes, banned transitions, vague adverbs, wall-of-quote links)
    hard-fails; fix the draft, don't `--no-lint` memo edits.
- **Inline sentence swaps**: `~/.claude/scripts/gdocs_replace_text.py` — structure-preserving
  find→replace for single-sentence tweaks that shouldn't disturb paragraph formatting.
- **Formatting values** come from `draft-investment-memo/canonical_spec.py` via the harness —
  never hand-format the Doc.

## Step 2: Load the record

Load directly (full fidelity, in this order):

1. **The memo Doc** (`read_file_content`) — note which sections are drafted vs. placeholder, and
   flag any orphaned draft fragments for cleanup on the first write.
2. **The Notion Opp page** — properties and sourcing history. Build the note roster from the Opp's
   **live `✍️ Notes` relation, NOT the memo's Appendix** — the Appendix is a point-in-time snapshot
   that drifts (notes added after the last publish, or pending / no-reply notes, silently go
   missing). Enumerate every entry in the relation, dedup against the Appendix, and load the delta.
3. **The Master Diligence Doc** (latest `vFinal` / Final Assessment note on the Opp). The
  `notion-fetch` result usually oversizes to a saved file: parse the JSON `text` field, map the
   heading structure first, then read the **Final Assessment** (top section) and the first-pass
   **Framework Mapping / Need to Believe** sections in full. The dated updates are historical
   layers — slice on demand, don't bulk-load.

Fan out **background subagents** (they return digests; you keep the synthesis).

Deal-record loaders:

- **Founder meetings**: fetch every founder-meeting note on the Opp with `include_transcript: true`.
- **Feedback + references**: every backchannel feedback note and founder reference note.
- **Drive materials**: reconcile against the Opp's **`Diligence Materials` AND `Deal Docs` property
  lists** (the source of truth — not the Appendix); read every file (founder memo, decks incl.
  deprecated ones, DDQs, models, discovery-call records). Identify each bare fileId before deciding
  it's irrelevant — a stray chip can be a side-letter redline, or a mislinked research PDF that a
  memo hyperlink wrongly points at (flag the latter as doc hygiene).

Reference-corpus loaders (deal-independent voice + framework context — the modern memo maps its
pillars onto these by name, so load the **full set up front**, not lazily):

- **All investment memos**: read every memo in the canonical folder in full; return per-memo pillar
  labels + framework glosses AND the verbatim self-citation passages (Quiet company-building, Tuor
  completeness, Factir composite, Rengo data-asset + opportunity-cost, Signal7 data-gravity), ready
  to block-quote. The newest 1-2 memos double as the structural/formatting template.
- **LP letters + LPAC deck**: read every Inverted quarterly LP letter (`[PARTNERS] Inverted Capital
  LP Letters` folder) and the **latest LPAC deck**. The deck is where Tom **codifies his six
  investment frameworks** (Solution Shape, Self-Reinforcing Moat, Multi-Act Sequencing, Founder
  Shape, Company-Building Style, Opportunity Cost — each mapped to an exemplar company); the modern
  memo's pillars map onto them by name. The letters carry reusable coined framings (kingmaking,
  opportunity-cost / self-selection, the non-obvious→obvious flip, temporal consistency).

Digest prompt spec (all loaders): raw data for memo drafting, not a human-facing message; per-item
coverage; every concrete number, term, and commitment; short verbatim quotes attributed to speaker
and date; founder-character evidence; open questions; no editorializing; 2,000-4,000 words.
Oversized fetches must be chunk-read to 100% coverage. **Report unreadable files as gaps** (e.g., a
founder-owned sheet returning "entity not found", or a server-side "ineligible in generative AI
contexts" block) — never silently skip; Tom may need to re-share or export an unrestricted copy.

## Step 3: Synthesis checkpoint

Before drafting, post one message: what's loaded, gaps found (unreadable files, missing artifacts,
doc-hygiene issues), and — if the Thesis is undrafted — candidate pillars mapped to the Inverted
frameworks from the first-pass, each with its strongest supporting evidence and the applicable
verbatim self-citation source. Then ask where Tom wants to start.

## Step 4: Iteration loop

- Be opinionated: diagnose before offering options, give 2-3 candidates with a recommendation and
  the reasoning, and push back when a suggestion fights the stylebook (e.g., intensifier adjectives
  where a `(vs. X)` contrast parenthetical is the Tom-native move).
- After every accepted tweak, show the latest version of the affected span as a `>` blockquote.
- Keep coined terms stable doc-wide: when a term changes in one sentence (e.g., "risk-based" →
  "risk-led"), sweep every other occurrence in the same edit.
- **`read_preview` immediately before every write.** Tom edits the Doc live mid-session; markers
  must come from the live text, not from an earlier read. If a marker isn't found, re-read — don't
  retry blind.
- Batch related edits into one `replace_range` (e.g., new opening + fragment cleanup + typo fixes
  in a single call). Choose markers that bracket the full block, including scratch fragments.
- **List-item + image traps (locked 2026-08-05, AgentBay session)** — the harness's range markers
  only match plain PARAGRAPHs, so `replace_range` CANNOT target text inside bullets/numbered
  items ("Start marker not found"); edit list items via direct Docs API batchUpdate (SA auth per
  `gdocs_replace_text.py`). When doing so: (a) `insertText` AND `replaceAllText` inherit the
  character style at the insertion index / the match's FIRST character — after any insert,
  explicitly set bold true on lead-ins AND bold false on bodies, never assume; NEVER begin a
  find string on linked or specially-styled text (the replacement takes that style across its
  full length — a find starting on a hyperlink turns the whole replacement into one giant link);
  after any replace that overlaps a link, strip links from the affected sentence and re-apply
  them on their exact anchors; (b) consecutive numbered pillars must share ONE list via a single
  `createParagraphBullets` spanning all of them, then apply the canonical indents from
  `canonical_spec.py` `THESIS.pillar_paragraph` (`number_*` keys: 18pt start / 0pt first-line) —
  the Docs preset default (36/18) is wrong; (c) NEVER `replace_section` a section containing an
  embedded inline image (Tom pastes LPAC slides into the Thesis) — section replace deletes the
  image; use surgical ops only; (d) consecutive numbered pillars get a BLANK unbulleted separator
  paragraph between them (canonical_spec `THESIS.empty_separator`: indent 0/0, no bullet) — insert
  `\n` at the next item's start, then `deleteParagraphBullets` + zero indents on the new empty
  paragraph; numbering continues across the gap when the items share a listId. Never leave numbered
  items butted together.
- Support `log to writing style` checkpoints via the `writing-style` skill.

## What NOT to do

- Don't create a new memo Doc, rebuild the skeleton, or run the draft-investment-memo pipeline.
- Don't build a PDF or share the Doc unless Tom asks.
- Don't touch Round Overview economics beyond what the diligence record states (priced-round math
  requires the pro-forma cap table).
- Don't re-derive voice rules inline — STYLE.md is the single source of truth.
