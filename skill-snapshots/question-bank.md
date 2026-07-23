---
name: question-bank
description: >
  Interactively aggregate Tom's OWN diligence questions for a pipeline
  Opportunity into a running, sharpened, de-duped question bank, then build the
  formal Diligence Q&A Google Doc on command by handing off to the diligence-qa
  builder. The signature behavior: load all Opp + diligence context from Notion
  FIRST, then run a conversational capture loop where Tom dumps questions and
  Claude sharpens each, flags overlaps with existing diligence open-questions,
  and maps them to the four canonical sections. Manual / conversational only.
  Trigger phrases: "maintain a question bank", "create a question bank", "start
  a question bank for [Opp]", "question bank for [company]", "I want to dump
  questions for [company]", "aggregate my questions for [X]", "help me prep
  questions for [founder/company]". Distinct from diligence-qa, which
  auto-drafts questions by reading materials — here the questions come from
  Tom's live dump, and this skill DELEGATES doc creation to diligence-qa's
  build / guard / link / alert pipeline. Always load context before dumping.
---

# Question Bank

Tom-initiated, conversational tool for aggregating **his own** diligence
questions ahead of a founder conversation (a walk, a call, a dinner), then
publishing them as the formal Diligence Q&A Doc when he says so.

**This is the interactive front-end to `diligence-qa`.** That skill auto-drafts
a question set by reading materials; this skill's questions come from Tom's live
dump. The two share ONE artifact and ONE builder — so this skill owns the
**context-load + capture loop** and DELEGATES the **build / guard / link /
alert** to `diligence-qa`. Never restate the builder mechanics here.

**Mode:** Manual / conversational only. Runs in Tom's interactive session
(inherits Opus). No scheduled run, no webhook, no `run.sh`, no `SKILL_MODEL`
entry.

---

## Step 1: Load context FIRST — before any dumping

The thing Tom values most: he starts dumping into a session that already knows
the deal. Do this whole step before inviting the first question.

### 1a. Resolve the Opp

Resolve the Opportunity exactly as `diligence-qa` Step 1a does — `notion-search`
the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`), stop and surface
candidates if ambiguous (never auto-pick), and apply the **FO-suffix gate**
(if the Opp name matches `/\([^)]*\bFO\b[^)]*\)/i`, route to the underlying
non-FO Opp). Fetch the Opp page and hold in working memory: Name, Description,
Round Details, Stage, Contact / founder, and **Next Meeting** (the conversation
this bank is prepping for).

### 1b. Pull the diligence open-questions (the overlap set)

Find the latest diligence artifact for the Opp — the Master Diligence Doc / PDF
and/or the first-pass Notion note (in the `✍️ Notes` relation or `Diligence
Materials`). Read it and extract into working memory:

- every **Open Questions** block (these are the questions diligence already
  flagged — the overlap set),
- the **Risks / Killshots** sections (unaddressed risks are the raw material
  for "what would you bring" in Step 2).

The Master Diligence Doc frequently exceeds the fetch token limit. When
`notion-fetch` spills to a saved file, slice it by character range (the fetch
tool result names the file and the technique) and grep for `Open Question`,
`Risk`, `Killshot`, and headings — do NOT try to re-read the whole thing into
context. A diligence artifact is NOT required; if none exists, the overlap set
is empty — say so in the Step 1c orientation and lean harder on Step 1b-alt.

### 1b-alt. No diligence doc ≠ no context — load everything else

Whether or not a diligence artifact exists, the Opp usually links other context.
When there is NO Master Diligence Doc / first-pass, this step is the context
load, not a fallback — do it thoroughly:

- **Opp page body** — original referral email, summary, founder background
  (often already on the page from add-to-crm).
- **`✍️ Notes` relation** — call notes / meeting notes; fetch each with
  `include_transcript: true` (per memory
  `feedback_summarize_call_use_full_transcript`).
- **`Diligence Materials`** — decks, memos, one-pagers, investor updates. Read
  what's readable (Drive PDFs, Docs); note anything skipped.
- **Sources / founder LinkedIn** — whatever the Opp links for founder
  background.

With no diligence doc, the Risks/Killshots raw material for Step 2's "what
would you bring" doesn't exist either — derive it yourself from the materials
just read (competitive threats, unproven assumptions, team gaps, round
mechanics) so the opinionated move still fires.

### 1c. Orient Tom, then open the floor

Give a tight orientation — one-paragraph deal summary, the conversation being
prepped for (date), that context + the diligence open-questions are loaded, and
that questions will be organized into the four sections (Product / Distribution
/ Market / Team) and flagged against what diligence already covers. Then invite
the first question. Keep it short; Tom wants to start dumping.

---

## Step 2: Capture loop (the running bank)

For each question Tom dumps, do all of the following — this is the value of the
skill, not just stenography:

- **Log it** into a running, section-mapped bank (Product / Distribution /
  Market / Team, plus an optional Other). Re-display the whole bank whenever
  Tom asks.
- **Sharpen it** — offer the crisper one-sentence version, the highest-leverage
  follow-up ("the payoff question"), and a one-line "why this matters" tie-in
  grounded in the specific deal.
- **Flag overlaps** with the Step 1b open-questions — tell Tom when diligence
  already flagged a question, so he decides whether it's worth asking the
  founder directly for the live read vs. already documented.
- **Tag fact-extraction vs. how-he-thinks** when it clarifies how to use the
  question (a walk is better for judgment-revealing questions than for fact
  lookups).
- **Split compound questions.** If one dump contains several questions, note
  the split — the doc renders one question per entry (builder requirement).

**When Tom asks "what am I missing / what would you bring":** be opinionated
(per memory `feedback_agent_behavior` — opinionated, not a pleaser). Propose
net-new questions drawn from the diligence Risks / Killshots and gaps his dump
hasn't touched, RANK them, name the must-asks explicitly, and map each to a
section. Do not just mirror his framing back.

Nothing is written outside the conversation during Step 2. The bank lives in
the session until Tom says build.

---

## Step 3: Build the Doc — on Tom's command

Triggers: "build the doc", "assemble it", "turn this into the Q&A", "create the
diligence Q&A", or any explicit build instruction.

### 3a. Close coverage gaps first

Every mandatory section needs ≥3 questions (`diligence-qa` coverage minimum /
G9). A founder-conversation dump commonly leaves **Team** empty — proactively
propose Team questions from the diligence's team risks (solo-founder execution,
critical hires) and let Tom keep / cut / rewrite. If a section is still thin
after that, offer to fill from the Step 1b open-questions or to drop the
section. **Confirm the final set with Tom before building** — especially any
questions you proposed rather than he dumped.

### 3b. Apply Tom's voice

Read `~/.claude/skills/writing-style/letters-and-memos/STYLE.md` and phrase
every question in his declarative, no-hedge register: plain interrogative, one
sentence, specific over generic, no preamble ("I'd love to understand…"), no
hedge stems ("could you maybe…").

### 3c. Assemble content + hand off to diligence-qa

Assemble the captured questions into the `diligence-qa` content schema
(`~/.claude/skills/diligence-qa/QA_CONTENT_SCHEMA.md`): a flat list of
`[label, question]` pairs per section key, 1–3 word labels (unbolded, no
trailing period — the builder normalizes), one question per entry. Write it to
`/tmp/qa_factir.<Company>.json`.

Then **execute `~/.claude/skills/diligence-qa/SKILL.md` Step 0 and Steps 5–8**
to build, guard, link, and alert. Do NOT restate or paraphrase those steps —
run them:

- **Step 0** — idempotency: if a Q&A Doc already exists for the Opp, STOP per
  that skill (no auto-version).
- **Step 5** — get-or-create the per-Opp Drive subfolder, run `build_qa_doc.py`.
- **Step 6** — `format_guard.py` HARD GATE (G1–G9); non-zero exit = stop and
  surface, do not publish.
- **Step 7** — link the Doc URL into the Opp's `Diligence Materials` via
  `notion_files_property.py`.
- **Step 8** — Slack alert via `send-alert`.

(Skip `diligence-qa` Steps 1–4 — the source-gather + auto-draft — entirely.
This skill replaced them with the Step 1 context-load + Step 2 capture loop.)

---

## Behavior rules

- **Context-load before dumping, always.** Step 1 is non-negotiable — it is the
  behavior Tom asked for by name. Never open the floor for questions before the
  Opp + diligence context is loaded.
- **Read-only until build.** Step 1 reads Notion; Step 2 writes nothing outside
  the conversation. The only writes (Drive folder, Doc, Notion file-property,
  Slack) happen in Step 3, after Tom says build. No permission prompts inside
  Step 3 (Tom-initiated), consistent with `diligence-qa`.
- **Be opinionated.** The "what am I missing" move is the differentiator — bring
  ranked net-new questions and name the must-asks, don't just organize his.
- **Never auto-version.** `diligence-qa` Step 0 refuses if a Q&A Doc exists;
  respect it. If Tom wants to extend an existing Doc, tell him this skill only
  drafts the initial version (edits are a manual pass until an
  `update-diligence-qa` skill exists).
- **No trailing offers** (memory `feedback_no_trailing_permission_offers`) —
  after the build reports done, stop.
