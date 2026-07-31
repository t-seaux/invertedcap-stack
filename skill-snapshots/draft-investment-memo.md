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
`"[Company] - Investment Memo" parent:1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`
(search without the `[WIP]` prefix so both draft and finalized memos match) and
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
   spot-check. **Black bullet glyphs in the Appendix (AgentBay lesson,
   2026-07-28):** when a bulleted list item is entirely a hyperlink, the
   bullet glyph inherits the link's blue into `paragraph.bullet.textStyle`
   and renders blue. Prevent it by keeping the leading zero-width space (or
   any unlinked char) as the paragraph's first run; if blue glyphs appear
   anyway, `createParagraphBullets` alone will NOT reset them — you must
   `deleteParagraphBullets` then `createParagraphBullets` over the same
   ranges (two separate batches; bullet ops don't shift text indices).
   Verify post-publish: walk table cells, check no `bullet.textStyle`
   carries link-blue foregroundColor.
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

Write the memo in markdown, following the modern skeleton. Save the
draft to `/tmp/memo_draft.[Company].md`. The structure is non-negotiable;
the substance inside it is where the work lives.

### Memo skeleton, Round Overview harness, Appendix + Thesis sourcing

The canonical memo skeleton and section order, the Round Overview
deterministic-facts harness (run `round_overview.py` — do NOT hand-derive the
7-value table), the Appendix sourcing rules, and the Thesis section sourcing
rules are the full procedure for this step — in
`references/memo-skeleton-and-sourcing.md`; **read it now before proceeding.**

### Format-consistency checklist (run BEFORE the audit in Step 5)

MANDATORY self-check gate. Self-check the draft against the full checklist —
full procedure in `references/format-consistency-checklist.md`; **read it now
before proceeding.** Failures must be fixed in the draft before the audit
invocation. This same checklist runs again in Step 5 on the post-audit draft.

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
Re-run the Step 2 format-consistency checklist against the final draft path
(full checklist in `references/format-consistency-checklist.md` — **read it
now before proceeding**). Fix any drift introduced by audit edits before
publishing.

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

Publish the final draft as a Google Doc by copying the canonical
`[TEMPLATE] Investment Memo` and populating it via Docs API `batchUpdate`
(the HTML→Doc upload path is deprecated for this skill). The entire Docs API
publish harness — cross-memo canonical shape, placeholder substitution map
(6a), template-copy + `replaceAllText` (6b), the canonical formatting harness
(6c), the inline styling walker spec (6d), the post-publish spot-check (6d),
and template maintenance (6e) — is the full procedure for this step, in
`references/step-6-docs-publish.md`; **read it now before proceeding.**

The publish MUST end with the 6d post-publish spot-check (no `{{PLACEHOLDER}}`
strings remain, all links present, Inverted logo + CONFIDENTIAL header + page
numbers present) before moving to Step 7. Do NOT re-implement formatting rules
inline — the sibling `formatting_harness.py` / `canonical_spec.py` /
`insert_drive_chips.py` are canonical.

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
`[WIP] [Company] - Investment Memo v2`, do not create a side-by-side draft, do
not append to the existing doc. Tom will route edits through a future
`update-investment-memo` skill.

### Never use the audit-failed escape hatch

If `first_pass_audit.py` exits with code 2 (judge crash, parse failure,
timeout on all chunks), STOP and investigate. Do NOT publish the draft with
a `⚠️ Audit invocation failed` line in the Slack alert — Tom explicitly
rejected that pattern on 2026-05-15 (per research-artifact-audit Step D).
