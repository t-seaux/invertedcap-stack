---
name: memo-workshop
description: >-
  Bootstrap a live co-writing session on an EXISTING investment memo Google Doc: load every layer of
  context (the memo doc, Notion Opp, Master Diligence final assessment, founder meetings + transcripts,
  feedback + reference notes, Drive diligence materials, reference memos, letters-and-memos stylebook)
  and the Docs editing harness, then iterate on prose with Tom turn by turn. Trigger on "work on the
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
   `[Company] - Investment Memo`. If Tom pasted a Doc URL, use that fileId directly.
2. Find the Notion Opportunity via `notion-search` on the company name.
3. **If no memo Doc exists**, stop and offer to run `draft-investment-memo` instead — that skill
   owns initial drafts (and conversely refuses when a Doc already exists). Never create a new
   memo Doc from this skill.

## Step 1: Load capabilities (before content)

- **Stylebook**: read `~/.claude/skills/writing-style/letters-and-memos/STYLE.md` in full, plus the
  most recent 1-2 entries of `VOICE_EXAMPLES.md` (recency-weighted). All voice, structure,
  anti-pattern, and self-citation rules live there — do not restate them; apply them.
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
2. **The Notion Opp page** — properties, sourcing history, and the linked-notes roster.
3. **The Master Diligence Doc** (latest `vFinal` / Final Assessment note on the Opp). The
  `notion-fetch` result usually oversizes to a saved file: parse the JSON `text` field, map the
   heading structure first, then read the **Final Assessment** (top section) and the first-pass
   **Framework Mapping / Need to Believe** sections in full. The dated updates are historical
   layers — slice on demand, don't bulk-load.
4. **Reference memos** — list the canonical memos folder; read the most recent memo in full (it is
   the structural template and freshest voice sample). STYLE.md names the canonical verbatim
   self-citation sources; pull those specific memos only when a pillar calls for the quote.

Fan out as **three parallel background subagents** (they return digests; you keep the synthesis):

- **Founder meetings**: fetch every founder-meeting note on the Opp with `include_transcript: true`.
- **Feedback + references**: every backchannel feedback note and founder reference note.
- **Drive materials**: every file in the Opp's `Deal Docs/[Company]/` folder + Diligence Materials
  chips (founder memo, decks incl. deprecated ones, DDQs, models, discovery-call records).

Digest prompt spec (all three): raw data for memo drafting, not a human-facing message; per-item
coverage; every concrete number, term, and commitment; short verbatim quotes attributed to speaker
and date; founder-character evidence; open questions; no editorializing; 2,000-4,000 words.
Oversized fetches must be chunk-read to 100% coverage. **Report unreadable files as gaps** (e.g.,
Drive's server-side "ineligible in generative AI contexts" block on founder-owned sheets) — never
silently skip; Tom may need to export an unrestricted copy.

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
- Support `log to writing style` checkpoints via the `writing-style` skill.

## What NOT to do

- Don't create a new memo Doc, rebuild the skeleton, or run the draft-investment-memo pipeline.
- Don't build a PDF or share the Doc unless Tom asks.
- Don't touch Round Overview economics beyond what the diligence record states (priced-round math
  requires the pro-forma cap table).
- Don't re-derive voice rules inline — STYLE.md is the single source of truth.
