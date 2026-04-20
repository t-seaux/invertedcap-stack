---
name: founder-outreach
description: >-
  Scoring + drafting primitive for pre-founder (-1) candidates. Two modes:
  (1) Score — reads an enriched -1 Scanner entry (invokes neg1-enricher to enrich first if
  missing), runs online research per ONLINE_SOURCES.md, writes Founder Eval Score +
  rationale + Contrarian Signals + auto-recommendation to Notion.
  (2) Draft — generates a Gmail draft using the canonical TEMPLATE.md cold email format,
  personalization anchored on the spike signal. Sets Status to "Draft Ready" when complete.
  Idempotent, never sends.
  Trigger phrases: "score [name]", "score the new -1 entries", "run founder scoring",
  "score this profile", "draft outreach for [name]", "draft cold email for [name]",
  "draft for [name]", "create outreach draft", "run founder-outreach", or a raw LinkedIn URL
  paired with draft/score intent. If given a raw profile without a -1 Scanner row, this skill
  invokes neg1-enricher first to enrich, then proceeds. Commonly chained after neg1-enricher
  to complete the -1 sourcing pipeline end-to-end. Part of pipeline management, not diligence.
---

# Founder Outreach

Standalone skill for scoring and drafting outreach for pre-founder candidates. Two modes:

1. **Score** — structured + online-research scoring against the 6-signal framework
2. **Draft** — generate Gmail outreach using the canonical template, personalization anchored on spike

Tom's manual `Reach Out? = ✅` flip is the boundary between Mode 1 and Mode 2. Scoring never flips the field on its own; drafting never runs without the flip.

## Notion Targets

- **-1 Sourcing Database** — `collection://32c00bef-f4aa-80a5-923b-000b83921fa3`
- **Companies Database** — `collection://7d50b286-c431-49f5-b96a-6a6390691309`

## Reference files (must read at mode invocation)

- `~/.claude/skills/founder-outreach/TEMPLATE.md` — cold email template, rubric, Reach Out thresholds, portfolio-calibrated priors (Mode 1 + Mode 2)
- `~/.claude/skills/founder-outreach/CALIBRATION.md` — 6 invest-calibration founder entries; anchor point for signal scoring (Mode 1)
- `~/.claude/skills/founder-outreach/ONLINE_SOURCES.md` — online research taxonomy per signal (Mode 1)

## Enrichment prerequisite

Both modes require the target to already exist in the -1 Scanner database (enriched via neg1-enricher). Required populated fields:
- `Name`, `LI`, `Profile Experience Summary`, `Company` relation
- Related Companies row's `Last Enriched` populated (so hypergrowth data is available)

**If the user feeds a raw LinkedIn URL and the person isn't in the -1 Scanner yet**, invoke `neg1-enricher` first (as a sub-step) to enrich, then proceed. Do not require the user to run enrichment separately.

---

## Mode 1: Score

**Purpose**: populate signal scoring on an enriched -1 Scanner entry. Leaves `Reach Out?` at `Pending` — the skill's auto-recommendation goes into `Reach Out Rationale` as text. Tom's manual flip is the action trigger for Mode 2.

### Trigger phrases

"score [name]", "score the new -1 entries", "run founder scoring", "score this -1 scanner entry", "score this profile".

### Input

One of: person's name, LinkedIn URL, or direct Notion page URL. For "score the new -1 entries", query for rows where `Founder Eval Score` is null AND a Company relation exists with `Last Enriched` populated.

### Steps

**1. Resolve target and verify enrichment.** Search -1 Scanner by Name or LI. If not found but a LinkedIn URL was given, invoke `neg1-enricher` to enrich first, then continue. Confirm `Profile Experience Summary`, `Company` relation, and related Companies row's `Last Enriched` are populated.

**2. Read full Notion context.**
- The -1 Scanner row's properties (all of them)
- The `Company` (primary employer) Companies row — especially `Funding Rounds JSON`, `Hypergrowth Windows`, `Industry`, `Founded Year`, `Employee Count`
- Each `Work History` Companies row with the same enrichment fields

**3. Read the 3 reference files** (TEMPLATE.md, CALIBRATION.md, ONLINE_SOURCES.md) to anchor scoring.

**4. Compute structured scores (ContactOut-sourced signals).**

- **Non-Linearity (0-10)** — count distinct function crossings across time-ordered Work History, mapping each role to the Function taxonomy:
  - 0-2: single-function career
  - 3-5: one clear crossing
  - 7-9: multi-hop (2+ crossings)
  - 10: Hardik archetype (3+ crossings, reinvented across disciplines)

- **Earned Reps (0-10)** — sum person-tenure years that OVERLAP with each employer's `Hypergrowth Windows`:
  - Pure tenure without evolution → 0 (drift)
  - Evolution within one company → 5-7
  - Hypergrowth overlap during tenure → 8-10
  - 10: Erik archetype (progressively smaller, progressively faster — Microsoft → Blend → Maybern)

- **Range (0-10)** — breadth across Technical + Commercial + Domain triple, weighted by "pulled-out-of-silo" markers in `experience[].summary` narratives. Follow the 0-10 table in TEMPLATE.md.

**5. Run Phase 2 online research for primary-source signals.**

For each of **Intellectual Rigor**, **Anticipation**, **Intentionality**, consult ONLINE_SOURCES.md and run time-boxed research (cap ~5-10 minutes per source). Record the specific URL + finding for citation.

- **Intellectual Rigor**: personal blog, GitHub READMEs/issues, Twitter threads, HN comment history, podcast transcripts, Manifold/Metaculus, Wikipedia contributions
- **Anticipation**: company blog launch posts, Product Hunt (as Maker), Show HN threads, Twitter Advanced Search for pre-project discussion, star-history.com, USPTO patents, angel investments in category
- **Intentionality**: personal bio essays, Twitter role-change announcements, podcast career-origin stories, commencement addresses, alumni magazine interviews, LinkedIn posts on transitions

Score each 0-10. If no observable online trace but other signals are strong, mark as "unobservable from public data" in the rationale rather than scoring 0.

**6. Apply portfolio-calibrated priors.**
- **Intentionality gate**: if peak signal ≠ Intentionality, require Intentionality ≥ 5 for an ✅ auto-recommendation
- **Don't downgrade based on thin public output alone**
- **Hypergrowth exposure dominates Earned Reps 10** — don't award 10 without it

**7. Compute Founder Eval Score** = MAX of 6 signal scores (not sum).

**8. Apply Reach Out auto-recommendation thresholds** per TEMPLATE.md:
- Peak ≥ 7 AND (peak is Intentionality OR Intentionality ≥ 5) → `✅`
- Peak ≥ 7 AND Intentionality < 5 → `🤔`
- Peak 4-6 → `🤔`
- Peak 0-3 → `❌`

**9. Write to Notion** (-1 Scanner row):

| Field | Value |
|---|---|
| `Founder Eval Score` | MAX score (number) |
| `Founder Eval Rationale` | Per-signal breakdown, 6 lines. Each: `• **Signal** (X/10): one-sentence rationale + [evidence URL if online]`. **Bold the signal name** in each bullet for scannability. End with: `Driver of score: **<Peak Signal>** (X/10). <one-sentence framing>`. |
| `Contrarian Signals` | Multi-select — tag all signals scoring ≥ 5. |
| `Working Description` | 2-3 sentence prose summary anchored on peak signal. Tom's voice. The TL;DR Tom reads first when triaging. |
| `Reach Out Rationale` | Auto-recommendation: `"Auto-rec: [✅/🤔/❌] – [reasoning tied to peak + Intentionality]."` |

**Leave unchanged**: `Reach Out?` (Pending — Tom's manual flip is the Mode 2 trigger), `Outreach Status` (Not Reached), `Gmail Draft URL` (empty).

**10. Report back.** Per person: name, peak signal, score, auto-recommendation, Notion link. Summary table for batches.

---

## Mode 2: Draft

**Purpose**: generate a Gmail draft for a scored -1 Scanner entry and set `Status = Draft Ready`. Idempotent (never overwrites existing Gmail draft). Never sends.

### Trigger phrases

"draft outreach for [name]", "draft cold email for [name]", "draft for [name]", "create outreach draft".

### Input

One of: person's name, LinkedIn URL, or Notion page URL.

### Steps

**1. Resolve target.**
- Search by Name or LI. If not found but a LinkedIn URL was given, invoke `neg1-enricher` to enrich first, then `founder-outreach` Mode 1 to score, then continue with Mode 2.
- Mode 2 is usually chained after Mode 1 (either by user invocation or by pipeline-agent Task 6). Standalone Mode 2 requires scoring to have already happened.

**2. Verify preconditions.**
- `Email` is populated (recipient required)
- `Founder Eval Score` + `Founder Eval Rationale` populated (need the spike signal + evidence for personalization)
- `Status` NOT IN (`Reached Out`, `Passed`) — don't re-draft if Tom has already acted

**2a. Delete-then-create pattern (handle existing drafts).**
Gmail's `create_draft` does NOT dedupe — each call creates a new draft. To prevent stacking on iteration (e.g., Tom asks for wording changes), always check for and delete any existing draft for this recipient first:
- Call `mcp__claude_ai_Gmail__list_drafts` with `query: "to:{email}"`
- For each returned draft whose subject is `"Introducing Inverted Capital"`, run `/Users/tomseo/.claude/skills/shared-references/delete-gmail-draft.sh <hex_id>` using the draft's hex ID (from `list_drafts`, NOT the `r-XXXX` transaction ID).
- Requires Chrome running + authenticated to Gmail + "Allow JavaScript from Apple Events" enabled.
- See memory `feedback_gmail_draft_persistent_id.md` for full pattern.

**3. Read TEMPLATE.md** for scaffolding, hyperlinks, formatting rules, and anti-patterns.

**4. Build the personalization paragraph** (3 sentences, per TEMPLATE.md "Personalization paragraph — structure"):

- **Radar hook sentence** — anchored on the spike signal from `Founder Eval Rationale`. Specific evidence, not pattern-match declarations. Sketches by peak:
  - Peak Non-Linearity: `"Your path caught my eye – [specific function-to-function crossing and why it's unusual]."`
  - Peak Earned Reps: `"Your [specific tenure at hypergrowth company] stood out – [what they saw/built during that window]."`
  - Peak Intellectual Rigor: `"Your [specific writing/project/talk] caught my attention – [specific quality — self-correction, framework, depth]."`
  - Peak Anticipation: `"You were building [specific project/thesis] well before it became consensus – [pre-consensus timing detail]."`
  - Peak Intentionality: `"The way you [specific counter-consensus move — demotion to switch, walked past a kingmaker, turned down accelerator] stood out."`
  - Peak Range: `"The [technical-to-commercial / commercial-to-technical] arc at [company] caught my eye – [specific cross-fertilization detail]."`

- **Career signal sentence** — one additional specific observation pulled from the rationale.

- **No-pressure frame** — always ends with: `No agenda, no ask. A low-pressure way to get to know each other.`

**5. Build full email per TEMPLATE.md** (exact scaffolding, hyperlinks, formatting rules). See TEMPLATE.md for the template block, hyperlink map, signature whitespace, and quote rules.

**6. Create Gmail draft** via `mcp__claude_ai_Gmail__create_draft` with:
- `to`: person's Email from Notion
- `subject`: `"Introducing Inverted Capital"`
- `body`: rendered email (HTML with hyperlinks)

**7. Retrieve the persistent draft ID.**
`create_draft` returns an `r-XXXX` transaction ID, NOT the persistent hex ID. Call `mcp__claude_ai_Gmail__list_drafts` with `query: "to:{email}"` immediately after creating; the most recent returned entry's `id` (hex format, e.g., `19da8bae7d10166e`) is the persistent ID. Use this for the Notion URL — the `r-XXXX` value doesn't resolve in browser URLs.

**8. Update Notion**:
- `Gmail Draft URL` → `https://mail.google.com/mail/u/0/#drafts/<hex_id>`
- `Status` → `"Draft Ready"`

**Leave unchanged**: everything else.

**8. Report back.** Per person: name, spike signal, 1-line personalization preview, Gmail draft URL. Summary table for batches.

---

## Important Rules

### Mode 1 (Score)

- **Cite evidence in the rationale.** Every online-sourced score should include the specific URL.
- **Unobservable ≠ 0.** Flag as "unobservable from public data" rather than defaulting to 0 when other signals are spiking.
- **Respect the Intentionality gate.** Don't auto-✅ a candidate whose peak isn't Intentionality unless Intentionality ≥ 5.
- **Time-box online research.** Cap ~5-10 minutes per source.
- **MAX, not SUM.** Founder Eval Score is the single highest signal.
- **Never flip Reach Out? to ✅ automatically.** Tom's manual action is the trigger for Mode 2.

### Mode 2 (Draft)

- **Never send.** Only create drafts. `create_draft` is a hard boundary.
- **Never overwrite an existing Gmail Draft URL.** Idempotence prevents clobbering Tom's edits.
- **Never draft without `Reach Out? = ✅`.**
- **Pull personalization from the rationale, not from scratch.** Rationale already captures the spike signal and evidence.
- **No pattern-match declarations.** Per TEMPLATE.md anti-patterns.
- **En dashes in all prose.** Per Tom's voice preference (memory: feedback_use_en_dash).

### Both modes

- **Invoke neg1-enricher for enrichment when needed.** Don't require the user to run it separately if they feed a raw profile.
- **Report concisely.** Summary table for batches. 3-4 lines max for singletons.

---

## Interaction with neg1-enricher scheduled auto-draft

The `neg1-enricher` skill contains a separate **scheduled sub-task** that auto-drafts for any `Reach Out? = ✅` rows without a Gmail draft, integrated into `pipeline-agent`'s runs. That scheduled sub-task does Mode 2 equivalently. Both paths — manual via this skill, scheduled via neg1-enricher sub-task — use the same TEMPLATE.md scaffolding and the same Notion update logic.

Manual invocation via this skill is for:
- End-to-end flows where you feed a raw profile ("score + draft outreach for [LI URL]")
- On-demand drafting when you don't want to wait for the next scheduled run
- Scoring flows (which the scheduled sub-task doesn't do — scoring is always manual in V1)

---

## Examples

### End-to-end manual flow

**User:** `"score and draft outreach for https://linkedin.com/in/janesmith"`

**Claude:**
1. Searches -1 Scanner → Jane Smith not found.
2. Invokes `neg1-enricher` to enrich from the LinkedIn URL → creates -1 Scanner row + Companies relations.
3. Runs Mode 1 scoring: reads reference files, computes structured + online scores, applies Intentionality gate, writes signal data + auto-rec to Notion. Reach Out? stays Pending.
4. Reports scoring result, notes that draft requires Tom's ✅ flip first.
5. (Tom reviews Notion, flips Reach Out? to ✅, runs `"draft for jane smith"`)
6. Runs Mode 2 drafting: reads TEMPLATE.md, builds personalization, creates Gmail draft, updates Notion.
7. Reports: drafted, Gmail URL.

### Score-only

**User:** `"score the new -1 entries"`

**Claude:**
1. Queries -1 Scanner for rows with null `Founder Eval Score` + populated `Company.Last Enriched`.
2. For each, runs Mode 1 end-to-end.
3. Returns summary table: names, peak signals, scores, auto-recs, Notion links.

### Draft-only

**User:** (after flipping ✅ on Greg Reiner's Notion row) `"draft for greg"`

**Claude:**
1. Finds Greg → Reach Out? = ✅, Gmail Draft URL empty, score populated.
2. Builds personalization anchored on peak Earned Reps (Meta cross-surface).
3. Creates Gmail draft.
4. Updates Notion: Gmail Draft URL + Status = Draft Ready.
5. Reports: drafted, spike, preview line, Gmail URL.
