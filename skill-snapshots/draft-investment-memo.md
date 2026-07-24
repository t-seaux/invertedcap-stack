---
name: draft-investment-memo
description: >-
  Draft the initial investment memo for a portfolio company Tom is investing in. Pulls every diligence artifact
  for the Opp (call notes + transcripts, deck, one-pager, first-pass diligence, pre-mortem, feedback), reads the
  reference memos in the canonical Drive folder (recency-weighted), and drafts a Google Doc in Tom's voice
  matching the modern memo skeleton — declarative opening, Round Overview table, Team table, numbered Thesis
  pillars from the first-pass analysis, Appendix links table. Audits every factual/quantitative claim against
  the source bundle via research-artifact-audit before publishing. Refuses if a memo Doc already exists.
  Trigger: "draft investment memo for [X]", "draft deal memo for [X]", "memo for [X]", "initial memo on [X]",
  "draft the memo on [X]", "first draft of the [X] memo". Manual-only.
---

# Draft Investment Memo

Draft the initial Google Doc investment memo for a company Tom is investing in.
The artifact is a long-form analytical document in Tom's voice that consolidates
the full diligence record into a publishable argument. The first-pass diligence
note is the primary analysis layer for the thesis — this skill turns that
framework-driven analysis into a memo written in Tom's voice, structurally
matching the most recent reference memos in the canonical Drive folder.

---

## Step 0: Idempotency check — refuse if a memo already exists

Before any other work, check whether an investment memo already exists for this
Opportunity in the canonical Drive folder
(`1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`).

Use `mcp__claude_ai_Google_Drive__search_files` with a query like
`"[Company] - Investment Memo" parent:1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO` and
also scan the Opportunity's `Diligence Materials` property for any URL
matching the memo Drive folder.

If a memo Drive doc is found, STOP. Reply to Tom with:

> A memo already exists at `<Drive URL>`. This skill only drafts initial memos.
> Want me to start a `memo-workshop` session on the existing doc instead?

Then, if Tom says yes, invoke the `memo-workshop` skill (context-loading + iterative
co-writing on an existing memo Doc).

Do NOT auto-version (`v2`), do NOT create a side-by-side draft, do NOT proceed.

---

## Step 1: Resolve the Opportunity & gather all sources

Resolve the company name to a Notion Opportunity row in the Opportunities DB
(`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Use `notion-search` first; if more
than one row matches, surface the candidates to Tom and stop until he picks
one. Never auto-pick.

### 1a. Fetch the Opportunity

`notion-fetch` the Opp page. Extract every memo-relevant property:

- Identity: `Name`, `Description`, `Website`, `HQ`, `Stage`, `Round Details`,
  `OS% @ Round`, `Inv @ Round`, `Close Date`, `Fund`, `Status`, `Contact`
- Relations: `🏁 Founder(s)`, `Source(s)`, `Support`, `Coinvestors`, `Angels`,
  `👓 Existing Backers`, `✍️ Notes`, `🗄️ Investor Updates`,
  `🕰️ Funding History`
- Files: `Diligence Materials`, `Deal Docs`
- Rollups: `📝 Founder Description`

If the Opp's `Status` is `Portfolio: Follow-On` or the name contains `(FO)`,
STOP. Memos are written for the original (non-FO) Opp; route Tom to the
underlying entry per the FO-routes-to-original memory rule.

### 1b. Fetch every linked Note with transcripts

For each page in the `✍️ Notes` relation, `notion-fetch` with
`include_transcript: true` (mandatory — the meeting-notes widget hides the
transcript otherwise; per memory `feedback_summarize_call_use_full_transcript`).
Categorize each note in working notes as one of:

- **First-pass diligence** — title pattern `[Claude] [Company] Master Diligence Doc` —
  this is the **primary analysis layer for the Thesis section**. Treat as
  highest-priority source.
- **Pre-mortem** — title pattern `Claude Pre-Mortem: [Company]` or similar.
  Stress-test layer; useful for sharpening pillars against failure modes.
- **Founder meetings / call notes** — direct founder signal.
- **Backchannel / expert feedback** — third-party perspective.
- **Other Claude analysis** — market research, model behavior research,
  pre-memo questions.

If no first-pass diligence note exists, STOP and tell Tom — the memo cannot be
drafted without that analysis layer.

### 1c. Fetch every Diligence Material

Iterate `Diligence Materials`. For each entry use the right access method:

- **Google Docs** → `mcp__claude_ai_Google_Drive__read_file_content` by file ID
- **Google Drive PDFs** → read with `read_pdf_bytes` via
  `https://drive.google.com/uc?export=download&id={FILE_ID}`
- **Notion-hosted attachments** → reuse the Chrome-based pipeline from
  `first-pass-diligence` Step 1c Type 3
- **DocSend** → invoke `docsend-to-pdf` to convert, then read the local PDF
- **Video / Loom** → reuse the Whisper / Loom transcript paths from
  `first-pass-diligence` Step 1c Type 5/6

Note that the first-pass diligence PDF will also typically be linked here —
read it once via whichever access path resolves it.

### 1d. Read all 5 reference memos in the Drive folder (weighted by recency)

List and read every memo in `1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`. As of 2026-07
the canonical set is:

1. **Factir** (May 2026) — newest, and THE canonical formatting exemplar
   (Tom-confirmed 2026-07-17): match its margins, logo placement, bullet
   indents, table chrome, and section layout exactly. `canonical_spec.py`
   and the template were baked from its publish run, so the harness output
   IS Factir formatting — verify against the doc itself on the Step 6d
   spot-check.
   [Doc](https://docs.google.com/document/d/1uDluLfFs7Qc7vONpIrwOptnDCARTXQF-23G4Ui6Vt90/edit)
2. **Tuor** (Feb 2026) — heaviest structural weight alongside Factir
3. **Signal7** (Nov 2025)
4. **Rengo** (Sep 2025)
5. **Quiet AI** (May 2025)
6. **Oun Homes** (Apr 2025) — oldest, drift exists (4-column Team table,
   plain-bold title, no `Appendix` H1 wrapper) — **do not replicate** the Oun
   formatting

The structure and formatting have evolved. When the most recent two memos
(Tuor, Signal7) diverge from the older three, **follow the recent pattern**.
Specifically: 3 thesis pillars is the modern norm, but do not constrain
yourself to 3 — if the first-pass surfaces more (Oun has 5), use what the
analysis warrants. Numbering is optional. Bolded lead-ins are mandatory.

The memos are also the source of framework vocabulary and cross-portfolio
cross-references. Build a `{company → file URL → frameworks expressed}`
manifest as you read — Tom's modern memos lean heavily on naming and
hyperlinking prior portfolio companies when invoking a recurring framework
("the same common language that shaped my thinking on Rengo, Quiet, and Oun").
The manifest is the only set of portfolio companies you may name as memo
analogs in this draft.

### 1e. Read the canonical style guide

Read `~/.claude/skills/writing-style/letters-and-memos/STYLE.md` in full.
This is the single source of truth for voice, register, argumentation pattern,
recurring framings, and anti-patterns. **Re-read it again before the audit
pass in Step 5** — it is designed to be consulted at iteration time, not
just up-front.

---

## Step 2: Draft the memo

Write the memo in markdown, following the modern skeleton below. Save the
draft to `/tmp/memo_draft.[Company].md`. The structure is non-negotiable;
the substance inside it is where the work lives.

### Canonical skeleton (modern — anchored on Tuor + Signal7)

```
*Month Year*

# [Company] Investment Memo

[Bold one-sentence declarative positioning statement.] [Optional 1-3
follow-up sentences elaborating what the company does and what it's
betting on. Top-down, declarative, no scene-setting.]

[Optional: bulleted "long-range hypotheses" list — used by Tuor; skip if
not warranted by the deal's structure]

## Round Overview

[ALL 7 values come from the deterministic harness — run
`python3 ~/.claude/skills/draft-investment-memo/round_overview.py
--company "[Company]" --lead "[Lead fund]"` and paste its table verbatim.
Do NOT hand-derive. See "Round Overview — deterministic facts harness"
below for the field mapping. Example output (AgentBay, 2026-07-17):]

| | |
|---|---|
| Fund | Inverted Capital I, LP |
| Round | Pre-Seed (Pre-Product, Pre-Revenue) |
| Round Size & Cap | $2.5m on $12.5m post (SAFE) |
| Inverted Check & Ownership | $1.0m (8.0%) |
| Investor Syndicate | Moonfire Ventures (Lead), QED Investors, Inverted Capital |
| Prior Funding | N/A |
| Deal Source | Laura Bock (QED Investors) |

## Team

| Name | Role | Previous |
|---|---|---|
| [Founder name with inline LinkedIn link] | [Role — CEO, CTO, CPO etc.] | [Career string: `Company (Function), Company (Function), School (Degree)`] |
| ... | ... | ... |

## Thesis

**[Pillar 1 named framework — bold lead-in, proper-noun framing, not a topic slug.]**
[2-4 paragraphs of continuous prose. Top-down assertion first, evidence second.
Multi-part logic runs as inline (a)/(b)/(c) or semicolon-chained causality —
NEVER as a bulleted list inside thesis prose. Cross-reference portfolio companies
by name with hyperlinks to their Drive memos where the framework recurs.]

**[Pillar 2 named framework.]**
[Same shape.]

**[Pillar 3 named framework — almost always some variant of "intentional,
rigorous, and intellectually honest company-building style" per STYLE.md.]**
[Same shape. When this canonical framing appears, prefer a verbatim block-quote
self-citation from a prior memo over a paraphrase — Tom's pattern.]

[Additional pillars as warranted by the first-pass diligence analysis. Do not
cap at 3 — use as many as the evidence supports. Oun used 5.]

# Appendix

## Diligence Materials & Notes

| Artifact | Links (Restricted) |
|---|---|
| **External** | |
| Investor Deck | [the company's fundraise deck from Opp.Diligence Materials, as a Drive chiclet; `N/A` if none] |
| Investor Memo | [the COMPANY's own memo (not Tom's), from Opp.Diligence Materials, Drive chiclet] |
| Other | [ALL remaining company-provided materials from Opp.Diligence Materials — one chiclet per line] |
| **Internal** | |
| Internal Workspace | [`[Company] Opportunity (Notion)` — hyperlink to the Notion Opp page] |
| Master Diligence Doc | [the Master Diligence PDF from Opp.Diligence Materials, Drive chiclet] |
| Diligence Q&A | [the ORIGINAL Q&A Google Doc(s) — may be one or more; never the PDF export] |
| Founder Meetings | [date-only bullets (`May 5, 2026`), each hyperlinked to the Notion note — ONLY calls Tom had with the founders, from the Opp's ✍️ Notes relation] |
| Feedback | [`Name @ Company` bullets, hyperlinked — customers, design partners, potential customers/partners, investors Tom asked for a take, from ✍️ Notes; never `[PENDING]` notes; FULL names always] |
| References | [grouped per founder: unbulleted full-name header line (`Nipun Jasuja`), that founder's reference bullets beneath (`Name @ Company (ex-X)`), blank line between groups — founder reference calls from ✍️ Notes] |
```

### Round Overview — deterministic facts harness

Run `round_overview.py` (sibling file) instead of hand-deriving the table.
Read-only Notion REST; token via `$NOTION_API_TOKEN` or the SOPS fallback.

```bash
python3 ~/.claude/skills/draft-investment-memo/round_overview.py \
  --company "[Company]" --lead "[Lead fund]"   # add --json for dict output
```

Field mapping (confirmed by Tom 2026-07-17):

- **Fund** — `Fund` select mapped to the formal legal name:
  `Inverted 1️⃣` → `Inverted Capital I, LP`. Unmapped funds warn loudly;
  extend `FUND_FORMAL_NAMES` in the script.
- **Round** — `Stage` select, emoji stripped, plus the default qualifier
  `(Pre-Product, Pre-Revenue)` (override with `--round-qualifier`).
- **Round Size & Cap** — parsed from `Round Details`. A "cap" round is a
  SAFE and renders `$2.5m on $12.5m post (SAFE)` — always say `post`; the
  `(SAFE)` suffix is what marks it, and its absence implies a priced round.
- **Inverted Check & Ownership** — `Inv @ Round` + `OS% @ Round`, rendered
  `$1.0m (8.0%)`. When OS% is blank and the round is a SAFE, ownership is
  computed as invested ÷ cap from Round Details (the only permitted
  no-cap-table math). Priced rounds with no OS% stay TBC — never
  approximate without the pro-forma cap table.
- **Investor Syndicate** — every fund in `Coinvestors` (page titles), the
  lead marked `(Lead)` and listed first (`--lead`; ask Tom if not
  specified — he designates the lead), `Inverted Capital` appended last.
- **Prior Funding** — `N/A` default; warns if the Funding History relation
  is non-empty so the value gets filled by hand.
- **Deal Source** — `Source(s)` relation rendered `Person (Company)` with
  the firm spelled out (e.g., `Laura Bock (QED Investors)`).

### Appendix sourcing rules (confirmed by Tom 2026-07-17, Factir example)

Every External row comes from the Opp's `Diligence Materials` property
field. `Investor Memo` is the memo the COMPANY put together — never
Inverted's memo. `Other` is a catch-all listing everything else the company
provided during diligence. Internal note rows are partitioned from the
Opp's `✍️ Notes` relation: Founder Meetings = Tom's calls with the founders
only, rendered as date-only links (`May 5, 2026`); References = founder
reference calls; Feedback = everyone whose take Tom solicited (customers,
design partners, potential customers/partners, investors) — EXCLUDE
`[PENDING]` feedback notes; a note only earns a row once the feedback has
actually landed (Tom 2026-07-17). References are GROUPED PER FOUNDER: an
unbulleted full-name header line per founder, that founder's reference
bullets beneath, blank line between groups — attribution comes from the
note title (`[Ref name]: [Founder] Reference`). FULL NAMES always in
Feedback and References, regardless of what the Notion note title uses
(note titles often shorthand — "Bukie" → "Bukie Adebo Umeano"); resolve
via the People DB, ContactOut as fallback. Bullets appear ONLY in the
Founder Meetings / Feedback / References cells, only on link-carrying
lines, with BLACK glyphs — all enforced by the harness's appendix-bullets
phase (see canonical_spec `list_labels` / `plain_labels` /
`bullet_glyph_black`). Drive-file cells render as smart chips (chiclets)
via `insert_drive_chips.py` at publish.

**PDF-snapshot policy (Tom 2026-07-17)**: company-provided Google Docs and
Slides/presentations are never chipped live — export each to PDF
(`files.export_media`, upload to the Opp's `Diligence/[Company]/` Drive
folder, named `[Company] - [Artifact].pdf`, preserving any `(Deprecated)`
tag) and chip the PDF snapshot instead. Spreadsheets/Excel models are the
exception: chip the live Sheet, since a frozen model loses its utility.
NOTE: the template's `{{APPX_*}}` placeholder rows predate this schema
(Overview Memo / One-Pager / Claude Artifacts etc.) — map old placeholders
to the new row set at populate time, and fold the renames into the next
template re-scaffold (Step 6e).

### Thesis section — sourcing rules

The Thesis is the heart of the memo and the section most exposed to
hallucination. Strict rules:

1. **Pillar selection comes from the first-pass diligence analysis.** Read
   the first-pass note's Framework Mapping section and identify the 3-N
   frameworks where the analysis is strongest. Those are the candidate
   pillars. The pre-mortem (if present) sharpens which pillars survive
   stress-testing.

2. **Every factual claim in a pillar must be citable.** Inline-hyperlink the
   source: `[March 12 call](Notion URL)`, `[Tuor memo](Drive URL)`,
   `[Carta 2024 retention report](https://...)`. Quantitative claims
   (market size, ARR, retention, comp valuations) must hyperlink to the
   primary source. If the source is a founder claim from a call, link to the
   Notion call note. If you cannot link a quantitative claim, cut it — do
   not approximate or hedge.

3. **Cross-portfolio cross-references are the signature move.** When a
   pillar invokes a framework that recurs across the portfolio, name and
   hyperlink the prior memos. Verbatim block-quote (indented `>`) when
   pulling from a prior memo — never paraphrase a framework Tom has already
   articulated. Use only memos in the Step 1d manifest. No biographical
   detail about analog founders unless the manifest memo states it verbatim
   and you're quoting verbatim — per the first-pass analog-claims rule.

4. **No bullets inside prose pillars.** Multi-part logic = inline (a)/(b)/(c)
   or semicolon chains.

5. **No synthesis paragraph at the close.** The memo ends mid-argument when
   the evidence is complete (per STYLE.md anti-patterns).

### Format-consistency checklist (run BEFORE the audit in Step 5)

Self-check the draft against this list. Failures must be fixed in the draft
before the audit invocation:

- [ ] Title is H1 (`# [Company] Investment Memo`), not bold-only
- [ ] Round Overview table has 2 columns and 7 rows in the canonical order
      (Fund / Round / Round Size & Cap / Inverted Check & Ownership / Investor
      Syndicate / Prior Funding / Deal Source)
- [ ] Team table has 3 columns (Name+LinkedIn / Role / Previous) — never 4
- [ ] Every thesis pillar has a bold lead-in
- [ ] No `[N]` numbered citation markers anywhere (memos use hyperlinks + blockquotes)
- [ ] No em dashes (`—`) in body prose — en dashes (`–`) only
- [ ] No escaped tildes, dollars, or brackets (`\~`, `\$`, `\[`, `\]`) — bare
      forms only. Notion-sourced markdown is the usual culprit; grep the
      draft for `\\[` and `\\$` literally and strip any matches before
      publishing. This has been a recurring failure mode on diligence
      artifacts — do not let it happen on memos.
- [ ] No "Furthermore", "Moreover", "In conclusion", "In addition", "With that in mind"
- [ ] No synthesis paragraph at the close of Thesis
- [ ] Appendix is wrapped under `# Appendix` H1 with `## Diligence Materials & Notes` H2
- [ ] All in-prose portfolio company mentions resolve to manifest memos with a hyperlink
- [ ] All call-note references hyperlink to the Notion page URL
- [ ] All quantitative claims hyperlink to a primary source
- [ ] **Header capitalization rule applied consistently.** All H1/H2/H3
      headers use Title Case (`## Round Overview`, `## Diligence Materials
      & Notes`, not `## Round overview` or `## diligence materials and
      notes`). The rule: capitalize the first letter of every word except
      articles, conjunctions, and short prepositions (`a`, `an`, `the`,
      `and`, `or`, `but`, `of`, `in`, `on`, `for`, `to`, `with`). The
      reference memos are Title Case across the board — match them. Tom
      has flagged inconsistent header capitalization as a recurring formatting
      bug; the linter does not always catch it.
- [ ] **Header visual hierarchy is unambiguous.** Inside the Thesis section,
      pillar lead-ins are bolded (`**Pillar Name.**` followed by prose),
      not promoted to H3 — promoting them creates a flat hierarchy where
      the pillar header looks the same as the section header. Bold lead-ins
      preserve the Tuor/Signal7 shape.

If any check fails, edit the draft and re-run the checklist before proceeding.

---

## Step 3: Build the source bundle

Concatenate every source consumed into a single markdown file at
`/tmp/memo_sources.[Company].md`. Use `==== SECTION NAME ====` as the section
delimiter and `--- Item: <title> ---` for items inside each section.

Section structure:

```
==== NOTION OPPORTUNITY ====
--- Item: Opportunity page properties ---
[full property dump]
--- Item: Opportunity page body ---
[full body content]

==== LINKED NOTES ====
--- Item: [Claude] [Company] Master Diligence Doc ---
[full content]
--- Item: Claude Pre-Mortem: [Company] ---
[full content]
--- Item: Call with [Founder] — [Date] ---
[full content INCLUDING transcript]
...

==== DILIGENCE MATERIALS ====
--- Item: Investor Deck ---
[full text]
--- Item: One-Pager ---
[full text]
...

==== REFERENCE MEMOS (Inverted portfolio) ====
--- Item: Tuor Investment Memo ---
[full text]
--- Item: Signal7 Investment Memo ---
[full text]
...

==== EXTERNAL RESEARCH ====
--- Source: <URL> ---
[snippet + context]
...
```

The bundle must contain every artifact the draft drew on. If you cite it,
bundle it. Missing sources = audit false positives.

---

## Step 4: Audit gate via research-artifact-audit

Read `~/.claude/skills/research-artifact-audit/SKILL.md` in full and follow
its operational discipline verbatim — including the HARD EXIT GATE, the
Step 3.5 partial normalization, and the hard prohibitions.

Bind these caller-specific values:

- `DRAFT` = `/tmp/memo_draft.[Company].md`
- `SOURCES` = `/tmp/memo_sources.[Company].md`
- `AUDIT_JSON` = `/tmp/memo_audit.[Company].json`
- `JUDGE_PROMPT` = `~/.claude/skills/first-pass-diligence/first_pass_audit.prompt.md`
- `AUDIT_RUNNER` = `~/.claude/skills/first-pass-diligence/first_pass_audit.py`
- `MAX_ITER` = `3`
- `WEB_RESEARCH_CAP` = `6`
- `ITER_SNAPSHOT_PREFIX` = `/tmp/memo_draft.[Company].iter`
- `NORMALIZED_DRAFT` = `/tmp/memo_draft.[Company].normalized.md`

**MANDATORY: pass `--chunk-size 0` to force single-pass un-chunked audit.**
The default chunked audit is fundamentally broken for memo bundles. When the
source bundle (typically 400–600KB for a memo) gets split into 5 chunks, each
chunk's judge sees only its own slice of sources but extracts claims from the
full draft. The runner spec claims `traced` verdicts merge across chunks
("positive evidence trumps absence in any single chunk"), but in practice
this merge produces dozens of false-positive `untraced` claims because (a)
claim_text strings don't align across chunks for deduping and (b) cross-chunk
merging silently fails on the audit JSON output. Factir run 2026-05-22:
chunked produced **64 untraced** (mostly "source not in bundle" false
positives where the source was in another chunk); single-pass produced
**0 untraced**.

For memo-sized bundles (well within Sonnet 4.6's context), always run:

```bash
python3 "$AUDIT_RUNNER" \
  --draft   "$DRAFT" \
  --sources "$SOURCES" \
  --output  "$AUDIT_JSON" \
  --chunk-size 0
```

If a future bundle genuinely exceeds the single-judge context limit, that's a
bundle-shrinking problem (drop reference memo full text and bundle only the
quoted passages), not a chunking problem.

**Note on citation format vs. judge prompt.** The first-pass judge prompt was
written for `[N]`-style citations. Memos use markdown hyperlinks instead. The
judge's claim taxonomy (founder_name, numeric_claim, analog_business_fact,
etc.) and verdict logic (traced / partial / untraced against the source
bundle) are syntax-agnostic — a claim is `traced` if the bundle contains the
substance, regardless of how the visible citation in the draft is formatted.
The judge will work as-is. If you observe a systematic false-positive
untraced pattern tied to hyperlink-style citations across multiple runs,
that's a judge-prompt patch — not a workaround in this skill.

After the audit completes (gate exits to publish, normalization runs if
partials > 0), the final draft path is either `$NORMALIZED_DRAFT` (if Step
3.5 ran) or `$DRAFT`. Make the selection explicit before proceeding.

---

## Step 5: Re-check format consistency on the post-audit draft

After the audit and normalization, the draft may have lost or gained content.
Re-run the Step 2 format-consistency checklist against the final draft path.
Fix any drift introduced by audit edits before publishing.

Also re-read `~/.claude/skills/writing-style/letters-and-memos/STYLE.md` and
do one final voice pass — looking specifically for:

- Throat-clearing transitions ("Furthermore", etc.) introduced by audit rewrites
- Hedge language ("approximately", "it could be argued") that softened a claim
- Em dashes that crept in
- Bulleted lists inside thesis prose (the audit sometimes converts inline
  enumeration to bullets when adding citations)

This is the last gate before publishing. Tom does not want to receive a memo
that passes the audit but drifts from voice.

---

## Step 6: Publish to Google Drive (template-copy + Docs API populate)

The HTML→Doc upload path is **deprecated** for this skill — it inherits
Drive's default Doc styling (no Inverted logo, no CONFIDENTIAL stamp, wrong
header weights, no italic section labels, no table borders, no page numbers).
The first Factir run on 2026-05-22 surfaced the gap.

The canonical path is **copy the template doc, then populate via Docs API
batchUpdate**. The template is a one-time-scaffolded Doc that carries all of
Inverted's chrome — logo, CONFIDENTIAL header, page numbers, paragraph
spacing, header styles (italic H2 labels), bullet indents, table borders +
padding, font family. The skill copies it and find-replaces the placeholder
tokens with memo content.

**Template:** `[TEMPLATE] Investment Memo` at Drive ID
`1Izzs5lp0YaLx1Lx48WVQ8b2WZGhYEgBXpBztrVVCp_M` in the memos folder. Built
as a **universal skeleton** — only the elements every memo always has: date,
title, lede paragraph, founder bridge paragraph, Round Overview table, Team
table, Thesis heading, Diligence Materials & Notes table. Tuor-only modules
(long-range hypotheses bullets, solution components bullets, multi-paragraph
thesis setup, embedded product screenshots) were stripped on 2026-05-22 v2
re-scaffold. Optional sections (Quiet-style Product/Distribution/Market H2s,
blockquote self-cite 1×1 tables, multi-founder Team rows) are added by the
skill on demand, not baked into the template.

**Cross-memo canonical shape** (from comparing all 5 reference memos —
Tuor, Signal7, Rengo, Quiet, Oun):

- **Lede**: 1-paragraph declarative — what the company does + end-state
  ambition. First sentence MAY be bolded if it reads as a clean tagline
  (Tuor pattern); otherwise leave unbolded (Signal7/Rengo/Quiet/Oun pattern).
  No "long-range hypotheses" or "solution comprises" bulleted lists —
  Tuor-only flourish.
- **Founder bridge**: closing paragraph before Round Overview that ties the
  founder profile to the thesis difficulty. Universal in Tuor + Signal7;
  optional but recommended when team is a dominant signal.
- **Round Overview**: 7 rows in this exact order — Fund, Round, Round Size
  & Cap, Inverted Check & Ownership, Investor Syndicate, Prior Funding,
  Deal Source — produced verbatim by `round_overview.py` (see the
  deterministic facts harness section in Step 2). Fund is the formal legal
  name (`Inverted Capital I, LP` for Inverted 1). Round always includes
  the qualifier `Pre-Seed (Pre-Product, Pre-Revenue)` (or Seed/Series A
  equivalent). SAFEs render `$2.5m on $12.5m post (SAFE)`; no `(SAFE)`
  suffix implies priced. Check & Ownership uses parens, not slash:
  `$750k (6.0%)`. Syndicate lists the lead first marked `(Lead)` with
  `Inverted Capital` last. Prior Funding is `N/A` (no parenthetical
  elaboration).
- **Team**: 3 columns (Name+LinkedIn / Role / Previous). Previous is
  comma-separated `Company (Function), …, School (Degree)`. No arrows
  (`→`) inside Function — use comma-separated roles when listing multiple
  positions at one company.
- **Thesis**: 3–5 pillars, each opened by a **bolded named-framework**
  lead-in. Numbered when ≥4 pillars; un-numbered when ≤3. Setup line is
  0–1 sentence, not 4 paragraphs (Tuor is the outlier with 4). Last
  pillar is universally "intentional, rigorous, intellectually honest
  company-building style" (4 of 5 reference memos). Self-cite block-quotes
  go in 1×1 tables with italic prose inside.
- **Optional post-Thesis H2s** (skip when an external one-pager / business
  plan / Master Diligence Doc carries the narrative): Company Overview
  (Signal7, Rengo), Product / Distribution / Market (Quiet only).
- **Appendix**: H1 wrapper + `Diligence Materials & Notes` H2 + 3-column
  table with External/Internal row grouping (merged col-1 cells). Canonical
  row set is the **union** of all 5 memos: External (Investor Deck, Customer
  Deck, Overview Memo, Business Plan, One-Pager, Product Demo) / Internal
  (Internal Workspace, Claude Artifacts, Diligence Q&A, Founder Meetings,
  Expert Feedback, References, optionally Transcripts, Pre-[Company] Notes,
  Downstream VC Feedback). Skip rows whose link is `N/A`.

Do not edit the template casually — it's load-bearing.

### 6a. Compose the placeholder substitution map

The skill's drafting step (Step 2) should produce content in placeholder-keyed
form, not free-form markdown. Map each draft section to a placeholder:

| Placeholder | Content shape |
|---|---|
| `{{DATE}}` | `Month Year` (e.g., `May 2026`) |
| `{{COMPANY}}` | Company name (appears in title + intro sentences) |
| `{{LEDE}}` | Bold one-sentence positioning + 1–3 elaboration sentences. Ends with: `{{COMPANY}} is founded on several long-range hypotheses:` — that intro is part of the same paragraph in the template. |
| `{{HYPOTHESIS_1/2/3}}` | One paragraph per bullet (Tuor uses 3) |
| `{{SOLUTION_1/2/3}}` | One paragraph per solution component bullet, with a `**bold-noun phrase**` lead-in |
| `{{FOUNDER_BRIDGE}}` | Closing paragraph before Round Overview |
| `{{ROUND_FUND}}`, `{{ROUND_STAGE}}`, `{{ROUND_SIZE_CAP}}`, `{{ROUND_CHECK_OWN}}`, `{{ROUND_SYNDICATE}}`, `{{ROUND_PRIOR}}`, `{{ROUND_SOURCE}}` | Single-line table cell values |
| `{{FOUNDER_NAME_LINK}}`, `{{FOUNDER_ROLE}}`, `{{FOUNDER_PREV}}` | Team table data row cells (one founder; if multi-founder, use Docs API `insertTableRow` to duplicate the row pattern first) |
| `{{THESIS}}` | Full thesis section as a block of prose, bullets, and (optionally) a blockquote table. The skill must apply inline styling via `updateTextStyle` after substitution — see §6c |
| `{{APPX_DECK}}`, `{{APPX_OVERVIEW_MEMO}}`, `{{APPX_ONE_PAGER}}`, `{{APPX_WORKSPACE}}`, `{{APPX_CLAUDE_ARTIFACTS}}`, `{{APPX_DILIGENCE_QA}}`, `{{APPX_FOUNDER_MEETINGS}}`, `{{APPX_FEEDBACK}}`, `{{APPX_REFERENCES}}` | Appendix table link-column cells. Use `N/A` for empty rows. Multi-link cells should use line-separated values (the template's bulleted cells retain their bullet style) |

### 6b. Copy template + run replaceAllText

```bash
python3 - <<'PY'
from google.oauth2 import service_account
from googleapiclient.discovery import build

SCOPES = ["https://www.googleapis.com/auth/drive", "https://www.googleapis.com/auth/documents"]
SUBJECT = "tom@invertedcap.com"
SA_KEY = "/Users/tomseo/.claude/local-agents/gmail-webhook-history-reconciler/sa-key.json"
PARENT = "1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO"
TEMPLATE_ID = "1Izzs5lp0YaLx1Lx48WVQ8b2WZGhYEgBXpBztrVVCp_M"

creds = service_account.Credentials.from_service_account_file(
    SA_KEY, scopes=SCOPES
).with_subject(SUBJECT)
drive = build("drive", "v3", credentials=creds, cache_discovery=False)
docs  = build("docs",  "v1", credentials=creds, cache_discovery=False)

# 1. Copy template
copy = drive.files().copy(
    fileId=TEMPLATE_ID,
    body={"name": "[Company] - Investment Memo", "parents": [PARENT]},
    fields="id,webViewLink",
    supportsAllDrives=True,
).execute()
doc_id = copy['id']
doc_url = copy['webViewLink']

# 2. Build substitutions map
substitutions = {
    "{{DATE}}": "May 2026",
    "{{COMPANY}}": "Acme",
    "{{LEDE}}": "Acme is the AI-native...",
    # … all other placeholders
}

# 3. Batch replaceAllText
requests = [
    {"replaceAllText": {"containsText": {"text": k, "matchCase": True}, "replaceText": v}}
    for k, v in substitutions.items()
]
docs.documents().batchUpdate(documentId=doc_id, body={"requests": requests}).execute()
print(doc_url)
PY
```

### 6c. Apply canonical formatting via the deterministic harness

All formatting rules below are codified in three sibling files — DO NOT
re-derive them in prose at run time:

- `canonical_spec.py` — column widths, paddings, indents, border colors,
  font sizes (single source of truth, measured 2026-05-22 v3)
- `formatting_harness.py` — idempotent passes that apply / repair every
  canonical rule (pillar bullets, blockquote styling, lede first-sentence
  bold, body paragraph separators, etc.)
- `insert_drive_chips.py` — Drive smart-chip insertion via Docs API
  `insertRichLink` (requires Drive scope + Docs scope on the SA — both
  DWD'd as of 2026-05-22)

**Invocation pattern** (after `replaceAllText` populates the template):

```bash
# Insert Drive chips for the appendix file labels
python3 ~/.claude/skills/draft-investment-memo/insert_drive_chips.py \
  --doc-id "$DOC_ID" \
  --search-from "Diligence Materials & Notes" \
  --chip "Factir Memo (April 2026)|https://drive.google.com/file/d/13FK.../view" \
  --chip "Factir_Master_Diligence_05.22.2026_vFinal.pdf|https://drive.google.com/file/d/1V1I.../view" \
  --chip "Pre-Seed Plan|https://drive.google.com/file/d/1MWf.../view" \
  --chip "ACV Build|https://drive.google.com/file/d/1v1L.../view" \
  --chip "Founder Positioning Notes (May 2026)|https://drive.google.com/file/d/1R8W.../view"

# Apply all canonical formatting (idempotent)
# --company is required: it drives the deterministic H1 enforcement
# (`{company} Investment Memo`) AND the Drive filename
# (`{company} - Investment Memo`). Both formats are defined in
# canonical_spec.{TITLE_FORMAT, DRIVE_FILE_TITLE_FORMAT}.
python3 ~/.claude/skills/draft-investment-memo/formatting_harness.py \
  --doc-id "$DOC_ID" \
  --company "Factir" \
  --pillar "A compounding data asset, accreted from" \
  --pillar "Payment orchestration as the wedge." \
  --pillar "An unusually complete founder, earned at" \
  --pillar "An intentional, rigorous, and intellectually honest" \
  --blockquote "… businesses sitting atop large" \
  --blockquote "In an era of company building where so much"

# Apply hyperlinks (Notion URLs in prose) via the SKILL's own walker —
# see 6d below.
```

**Memo content schema** for the populate step: see `MEMO_CONTENT_SCHEMA.md`
in this skill's directory. The schema defines the structured input shape
the populate step expects, so future skill runs compose memo content into
a clean dict before driving `replaceAllText`.

**Do NOT re-implement formatting rules in inline batchUpdate calls in this
skill.** Each rule has been tuned from the cross-memo audit; the harness
is the canonical implementation. If a new rule is needed:
1. Add the constant to `canonical_spec.py`
2. Add the phase to `formatting_harness.py`
3. Document the spec change here

### 6d. Apply inline styling per the canonical formatting spec (legacy reference)

This section describes the rules the harness implements. Read for context
when extending the harness; do NOT manually apply via batchUpdate inline.

`replaceAllText` populates content but not structure. The thesis area in
particular needs a post-replace pass to match the cross-memo canonical
formatting. The full spec, measured from Tuor + Signal7 + Rengo + Quiet
on 2026-05-22 v3 audit:

**Pillar paragraphs** — every thesis pillar lead-in (`A compounding data
asset…`, `Payment orchestration as the wedge.`, etc.) must be a
**bulleted paragraph**:

```python
{"createParagraphBullets": {
    "range": {"startIndex": pillar_start, "endIndex": pillar_start + 1},
    "bulletPreset": "BULLET_DISC_CIRCLE_SQUARE"
}}
{"updateParagraphStyle": {
    "range": {"startIndex": pillar_start, "endIndex": pillar_start + 1},
    "paragraphStyle": {"indentStart": {"magnitude": 18, "unit": "PT"}},
    "fields": "indentStart,indentFirstLine"
}}  # clears indentFirstLine; bullet marker sits at left margin, text at 18pt
```

Pillar first text-run gets `bold=True` (applies to lead sentence ending
in a period; rest of pillar paragraph stays non-bold).

**Body paragraphs between pillars**: NORMAL_TEXT, `indentStart=18pt,
indentFirstLine=18pt, bullet=False`. Inherited from the `{{THESIS}}`
placeholder paragraph's style after the SKILL applies a base style of
indent=18 to the whole thesis content range immediately after the
replaceAllText pass.

**Empty-separator paragraphs inside the thesis range** (the `\n\n`
spacers between pillar blocks): must be reset to `indentStart=None,
indentFirstLine=None`. Without this, the spacers carry the bullet's
indent and the visual rhythm breaks.

**Blockquote 1×1 tables** for self-cite verbatim quotes (Rengo, Quiet,
prior-memo pulls). Per Tuor's spec:

| Attribute | Value |
|---|---|
| Column width | `429.75 pt` (content width 468pt minus 18pt indent + 20.25pt right margin) |
| Cell padding (all sides) | `0` (effectively unset) |
| Border Right / Top / Bottom | `width=1pt, dashStyle=SOLID, color=RGB(1,1,1)` |
| Border Left | unset (visual: open on the left) |
| Cell paragraph indent | `indentStart=9pt, indentFirstLine=9pt` |
| Cell text first run | `italic=True` (Tuor + Rengo). Signal7 uses italic+bold |

**Critical Docs API note**: cell-level borders only apply via
`updateTableCellStyle` with **explicit `tableRange`** targeting cell
`(0,0)` with `rowSpan=1, columnSpan=1`. Using `tableStartLocation` only
silently drops the border requests for 1×1 tables. Use this shape:

```python
{"updateTableCellStyle": {
    "tableRange": {
        "tableCellLocation": {
            "tableStartLocation": {"index": tbl_start},
            "rowIndex": 0,
            "columnIndex": 0
        },
        "rowSpan": 1, "columnSpan": 1
    },
    "tableCellStyle": {"borderRight": border, "borderTop": border, "borderBottom": border},
    "fields": "borderRight,borderTop,borderBottom"
}}
```

**Hyperlinks** — apply via linear text walk through the rendered doc.
Extract `[anchor](url)` pairs from the markdown source in order, then
search the doc's flat text for each anchor starting from the previous
link's end position. Apply `updateTextStyle` with
`link={"url": url}, foregroundColor=blue, underline=True` to the matched
range. Missing anchors usually mean the cleanup pass dropped the
surrounding cell content (e.g., Round Overview cell values that got
shortened to `N/A`) — confirm absence and skip.

**Lede first sentence bolding** (Tuor pattern, optional): after
`{{LEDE}}` substitution, locate the first sentence and apply
`updateTextStyle` with `bold=True` on that range. Critically, the
replaceAllText pass inherits the source character's style, so the entire
LEDE paragraph may come back fully bold (because Tuor's source first
character was bold). Apply a follow-up `updateTextStyle` with
`bold=False` to the range AFTER the first sentence to un-bold the rest.

**Lede ↔ Founder-bridge separator**: the template puts `{{LEDE}}` and
`{{FOUNDER_BRIDGE}}` on consecutive paragraphs with NO blank line
between. After `replaceAllText`, locate the founder-bridge paragraph's
startIndex and insert `\n` to create a blank separator paragraph.

**Team table founder name cell**: split into two stacked paragraphs —
line 1 `Name`, line 2 `(LinkedIn)`. The hyperlink applies to the word
`LinkedIn` only (NOT to the name). Per Tuor / Signal7 / Rengo / Quiet
majority. Round Overview cells stay plain text — **no hyperlinks** in
Round Overview (strip them with `updateTextStyle` clearing `link,
foregroundColor, underline` fields if any survived the populate step).

**Team `Previous` cell format**: comma-separated `Company (Function),
Company (Function), …, School (Degree)`. Multi-role at one company gets
collapsed into a single entry with comma-separated roles in parens. No
arrows (`→`). Mirror to the founder's actual LinkedIn profile via
`mcp__contactout__contactout_enrich_linkedin_profile`.

For V1 / first runs of a new memo: ship the simple `replaceAllText` pass
first, then apply this styling pass in a second `batchUpdate`. The
2026-05-22 v3 Factir run baked this end-to-end and the script is
preserved at `/tmp/factir_*` for reuse as the template for future
populate-then-style runs.

### 6d. Post-publish spot-check

Export the populated Doc as HTML via Drive v3 `files.export_media` and
confirm:
- No `{{PLACEHOLDER}}` strings remaining (grep for `{{`)
- All `<a href>` links present (count should match the link count in the
  source content)
- Inverted logo present (check for image elements in the HTML)
- CONFIDENTIAL header present
- Page numbers in the footer

If any check fails, fix the upstream substitution map and re-copy from
template (cheap operation — the broken artifact can be trashed via
`drive_rename.py --trash`).

### 6e. Template maintenance

The template was scaffolded once on 2026-05-22 by copying Tuor and running a
Docs API `batchUpdate` that replaced body content with placeholder tokens.
If Tom updates the canonical visual language (new logo, different header
style, new section), re-scaffold the template:

1. Copy the new canonical memo (e.g., the next Tuor-equivalent) to
   `[TEMPLATE] Investment Memo v2`
2. Run the scaffold script (preserved in this skill's git history; the
   placeholder names and cell mapping logic is what matters)
3. Trash the old template
4. Update the `TEMPLATE_ID` constant in Step 6b above

Do not edit the template by hand in the browser unless you're confident the
edit doesn't break the placeholder match strings the skill relies on.

---

## Step 7: Link the memo back into the Opp

Append the Drive URL to the Opportunity's `Diligence Materials` property
field (per memory `feedback_deck_url_in_diligence_materials_field`: artifact
links live in the property field as-is, not duplicated in the page body).

Use the Notion file-property helper at
`~/.claude/scripts/notion_files_property.py` to add the Drive URL as an
external file entry — do NOT call `notion-update-page` with a Files property
in the inline schema; per memory `reference_notion_files_property_helper`
that path is brittle.

Do NOT create a separate Notes-DB entry for the memo. The Drive doc is the
canonical artifact. Notes DB is reserved for Claude analytical artifacts
(first-pass, pre-mortem, market research, etc.).

---

## Step 8: Slack alert via send-alert

Read `~/.claude/skills/send-alert/SKILL.md` and post a summary to
`#claude-alerts` with:

- Company name + Drive URL of the memo
- Word count of the body (target ~2,000–2,500 per the reference memos)
- Audit summary: final `untraced` count, `partial` count, iteration count
- Any normalized partials as a before → after diff (mandatory per
  research-artifact-audit Step D)
- Any remaining `untraced` claims with the judge's notes (if iteration cap
  hit before `untraced==0`)

GFM links (`[text](url)`) — never Slack mrkdwn `<url|text>` — per memory
`feedback_send_alert_gfm_not_mrkdwn`.

---

## Behavior rules

### Never ask permission inside this flow

This skill is end-to-end Tom-initiated. Do not prompt for confirmation on:

- Reading any Notion page or Drive file
- Building the source bundle
- Invoking the audit
- Creating the Drive Doc
- Linking the URL into Diligence Materials
- Posting the Slack alert

The only stops are (a) Step 0 idempotency check (memo exists), (b) Step 1
ambiguous Opp resolution, (c) Step 1b missing first-pass diligence, and (d)
audit script exit 2 (per research-artifact-audit hard prohibition: never
publish with an "audit invocation failed" caveat — fix and re-run).

Per memory `feedback_first_pass_no_permission_prompts` — applies in full.

### Never auto-version

If a memo already exists in the Drive folder, refuse. Do not create
`[Company] - Investment Memo v2`, do not create a side-by-side draft, do
not append to the existing doc. Tom will route edits through a future
`update-investment-memo` skill.

### Never use the audit-failed escape hatch

If `first_pass_audit.py` exits with code 2 (judge crash, parse failure,
timeout on all chunks), STOP and investigate. Do NOT publish the draft with
a `⚠️ Audit invocation failed` line in the Slack alert — Tom explicitly
rejected that pattern on 2026-05-15 (per research-artifact-audit Step D).
