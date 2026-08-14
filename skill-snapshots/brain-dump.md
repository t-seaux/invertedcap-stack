---
name: brain-dump
description: "Live capture of Tom's free-form thinking on a pipeline opportunity into a single living Notes DB row — verbatim-first, structured later — as source material for the investment memo. Trigger on \"brain dump on [company]\", \"capture my thinking on [X]\", \"I want to dump thoughts on [X]\", \"start a brain dump\", or when Tom resumes dumping thoughts on a company that already has a Brain Dump note. Manual-only (Mode C); no scheduled or webhook path. NOT note-logging of a discrete artifact (that's the Notes DB entry rules in feedback_notes_and_updates) and NOT memo drafting (that's draft-investment-memo / memo-workshop — this skill feeds them)."
---

# Brain Dump

Capture Tom's stream-of-consciousness thinking on an opportunity as it comes — spoken dictation, pasted fragments, half-formed takes — into one living Notion note. The capture is verbatim-first; structure is a later, separate layer. The end use is the investment memo: the dump is the primary source of thesis and voice for `draft-investment-memo` (no memo Doc yet) or `memo-workshop` (memo Doc exists).

**What the dump is (Tom, 2026-08-13):** ORIGINAL thoughts — Tom reflecting on his interactions with the founder/company and on the research and diligence materials, deliberately NOT regurgitating facts from the record. That is exactly why it is a key (albeit raw) memo input. Division of labor: the **dump supplies the original argument** (framings, hypotheses, convictions, tensions); the **diligence record supplies the evidence**. So never judge a dump batch by whether it restates the record's facts — its job is to say something the record doesn't. Grounding (Step 3.3) means attaching the record's evidence to his original thought or surfacing a tension with it, never suggesting the thought should be more factual. In the memo, his original thinking is the payload; the record is the citation base beneath it.

## Step 1: Resolve the opportunity and the note

1. Resolve the company to its Opportunities DB row (data source `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). If semantic search comes up empty, run a direct `Name LIKE` query before concluding it doesn't exist.
2. Check the Opp's `✍️ Notes` relation for an existing row titled `[Company]: Brain Dump`. **One brain-dump row per company, ever** — if it exists, append to it; never create a second.
3. If none exists, create a Notes DB row (data source `collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`):
   - **Name**: `[Company]: Brain Dump` — **no date in the title** (deliberate exception to the usual `Subject: Type (YYYY-MM-DD)` note-title shape; the row is a living doc, not a point-in-time artifact)
   - **Icon**: 🧠
   - **Category**: `Diligence`
   - **Opportunity**: relation → the Opp
   - **Body**: opens directly with the first `## Raw Dump — <date>` heading — no preamble, purpose blurb, or boilerplate lines (Tom removed one 2026-08-13)

## Step 2: Load the diligence record

Before or shortly after capture begins, load the full diligence record so each thought can be contextualized against the evidence Tom is relying on:

- The Opp page (round details, status, source email)
- Every linked note: founder call notes, reference calls, feedback notes, prior updates
- The Master Diligence doc's **Final Assessment** section (and Need-to-Believes) if one exists — for very large pages, fetch and read the Final Assessment portion rather than the whole page

This powers the grounding notes in Step 3 and any synthesis in Step 4. If Tom starts dumping before the load finishes, capture first — never make him wait.

## Step 3: Capture loop

For each batch Tom delivers:

1. **Append a section** to the note: `### <short thematic title you write>` under a date-only `## <Mon DD, YYYY>` heading — no "Raw Dump" label, that's implied by the note itself (add the dated heading the first time capture happens on a given day; subsequent sections that day nest under it). A dump is expected to span multiple days or weeks of revisits — the dated headings are the journal spine, and later-day thoughts may revise earlier ones; capture the revision as a new section, don't edit the old one (except transcription fixes).
2. **Verbatim-first transcription repair.** Tom usually dictates; fix obvious speech-to-text artifacts (e.g. "pre seat" → "pre-seed", "own wall" → "OwnWell") and light disfluencies ("um", repeated words), but preserve his phrasing, rhythm, and hedges exactly. Never compress, reorder, or synthesize.
   - Uncertain repair → keep your best reading and bracket the raw: `[transcription unclear: "off Cambridge"]`, then ask Tom in chat.
   - Dictation cut off mid-sentence → mark `[dictation cut off mid-thought]` and prompt him to resume from there.
   - When Tom corrects a word ("I meant X"), fix the note in place and remove the bracket.
3. **Reply in chat with a short confirmation** plus at most one grounding observation from the diligence record — an alignment ("three Box references converge on this"), a correction the record forces ("'solo founder' isn't literally true — Scott Sperling is Head of Eng"), or a counterweight ("Eric Rosenfeld's overthinking read is the other side of this trait"). Cite specifics (names, numbers) from the record, not vibes. No unsolicited multi-point synthesis between batches — stay out of the way.
4. Never block capture on questions. Flag ambiguities after logging, not before.

## Step 4: Synthesis and organization (on ask only)

- **"Summarize my take" / "what's the read"** → apply Tom's synthesis shape: abstract to a few load-bearing points grounded in his verbatim quotes, name the unifying thesis explicitly, close with the most important gap in the dump so far (what a memo would still be missing). Do not re-list the sections.
- **"Organize the dump"** → add a thematic organization layer (e.g. a top-of-note map grouping sections into thesis pillars / risks / open questions). The raw dated sections always stay intact beneath — organization is additive, never a rewrite.

## Step 5: Hand-off to the memo

When Tom says he's done (or asks to move to the memo): remind him of the route — `memo-workshop` if a memo Google Doc already exists for the company, `draft-investment-memo` if not. Both pull Opp-linked notes automatically, so the brain dump rides along; no export step needed.
