---
name: founder-outreach
description: >-
  Drafting primitive for pre-founder (-1) candidates. Reads an already-evaluated -1 Scanner
  row (Eval Score + Eval Breakdown + Claude Rec populated by neg1-enricher) and generates a
  Gmail draft using the canonical TEMPLATE.md cold email format, personalization anchored on
  the spike signal. Sets Status to "Draft Ready" when complete. Idempotent, never sends.
  Scoring + evaluation lives in neg1-enricher — this skill refuses to run if the row is
  unscored. Trigger phrases: "draft outreach for [name]", "draft cold email for [name]",
  "draft for [name]", "create outreach draft", "run founder-outreach". Auto-invoked as
  the terminal step of neg1-enricher on MANUAL -1 scan requests (runs regardless of
  Claude Rec — even Pass verdicts get a draft so Tom sees what the outreach would look
  like before deciding). Not auto-invoked on pipeline-agent Task 6 batch enrichment.
  Part of pipeline management.
---

# Founder Outreach

Single-mode drafting primitive for pre-founder candidates. Reads a fully-evaluated -1 Scanner row and generates a Gmail draft. Scoring, rubric application, and recommendation all live in `neg1-enricher` — this skill is just the email.

## Notion Targets

- **-1 Sourcing Database** — `collection://32c00bef-f4aa-80a5-923b-000b83921fa3`
- **Companies Database** — `collection://7d50b286-c431-49f5-b96a-6a6390691309`

## Reference file (must read at invocation)

- `~/.claude/skills/founder-outreach/TEMPLATE.md` — cold email template (subject, body scaffolding, hyperlinks, formatting rules, personalization-paragraph structure, anti-patterns)

## Precondition check — refuse if unscored

Before doing anything, verify the target row is fully evaluated. Required populated fields on the -1 Scanner row:

- `Eval Score (Spike)` — the peak signal score
- `Eval Summary` — the rationale (includes the bold peak sentence)
- `Claude Rec` — the verdict (`Reach Out ✅` or `Pass ❌`)
- `Eval Breakdown` — per-signal breakdown (needed for personalization anchor)
- `Email` — recipient required

**If any of those are missing**, refuse to draft. Tell Tom: "This row hasn't been evaluated yet. Run `neg1-enricher` on it first — that skill now owns enrichment + scoring + the Claude Rec verdict. Then come back to draft." Do not fall back into running the rubric here. Scoring is out of scope for this skill.

**If `Claude Rec = Pass ❌`**, also refuse. Tell Tom the row was auto-passed by neg1-enricher; if he wants to override, he should flip Claude Rec to ✅ in Notion and re-invoke this skill.

**If `Status` is already `Reached Out` or `Passed`**, also refuse. Don't redraft once Tom has acted.

---

## Trigger phrases

"draft outreach for [name]", "draft cold email for [name]", "draft for [name]", "create outreach draft", "run founder-outreach".

Explicitly NOT triggered by: "score [name]", "score the new -1 entries", "run founder scoring", "score this profile", or any enrichment-intent phrase. Those route to `neg1-enricher`.

## Input

One of: person's name, LinkedIn URL, or Notion page URL.

## Steps

**1. Resolve target.** Search -1 Scanner by Name or LI. If not found, tell Tom the row doesn't exist and suggest he run `neg1-enricher` on the LinkedIn URL first. Do NOT auto-invoke neg1-enricher — the user should see the enrichment + scoring output before any draft is generated.

**2. Run the precondition check** (above). Halt with a clear refusal message if anything is missing.

**3. Delete-then-create pattern (handle existing drafts).**
Gmail's `create_draft` does NOT dedupe — each call creates a new draft. To prevent stacking on iteration (e.g., Tom asks for wording changes), always check for and delete any existing draft for this recipient first:
- Call `mcp__claude_ai_Gmail__list_drafts` with `query: "to:{email}"`
- For each returned draft whose subject is `"Introducing Inverted Capital"`, run `/Users/tomseo/.claude/skills/shared-references/delete-gmail-draft.sh <hex_id>` using the draft's hex ID (from `list_drafts`, NOT the `r-XXXX` transaction ID).
- Requires Chrome running + authenticated to Gmail + "Allow JavaScript from Apple Events" enabled.
- See memory `feedback_gmail_draft_persistent_id.md` for full pattern.

**4. Read TEMPLATE.md AND the voice corpora.**
- `TEMPLATE.md` — scaffolding, hyperlinks, formatting rules, anti-patterns.
- `EDIT_PATTERNS.md` (same directory) — record of how Tom edits Claude-drafted outreach before sending. Scan the last 20–50 entries; identify 3–5 most frequent patterns (e.g., "cuts filler closers," "tightens opener," "removes hedging adverbs"). Apply as priors when drafting, then check the final draft against them.
- `VOICE_EXAMPLES.md` (same directory) — full sent emails Tom wrote from scratch (no Claude draft involved). Scan the 2–3 most recent for canonical voice. Use as ground truth — if your draft sounds nothing like these, recalibrate.

Both files are auto-maintained by the `draft-feedback` pipeline (FOUNDER_EVAL_FRAMEWORK.md §13). Patterns are observations, not commands.

**5. Build the personalization paragraph** (per TEMPLATE.md "Personalization paragraph — structure"). The anchor is the **spike signal** already identified in `Eval Breakdown` — do not re-derive it. Pull the specific evidence cited in the breakdown for that peak signal, rewrite in Tom's voice.

Sketches by peak signal:
- Peak Non-Linearity: `"Your path caught my eye – [specific function-to-function crossing and why it's unusual]."`
- Peak Earned Reps: `"Your [specific tenure at hypergrowth company] stood out – [what they saw/built during that window]."`
- Peak Intellectual Rigor: `"Your [specific writing/project/talk] caught my attention – [specific quality — self-correction, framework, depth]."`
- Peak Anticipation: `"You were building [specific project/thesis] well before it became consensus – [pre-consensus timing detail]."`
- Peak Intentionality: `"The way you [specific counter-consensus move — demotion to switch, walked past a kingmaker, turned down accelerator] stood out."`
- Peak Range: `"The [technical-to-commercial / commercial-to-technical] arc at [company] caught my eye – [specific cross-fertilization detail]."`

Follow the career-signal-sentence + no-agenda-frame structure in TEMPLATE.md exactly.

**6. Build full email per TEMPLATE.md** (exact scaffolding, hyperlinks, formatting rules). See TEMPLATE.md for the template block, hyperlink map, signature whitespace, and quote rules.

**7. Create Gmail draft** via `mcp__claude_ai_Gmail__create_draft` with:
- `to`: person's Email from Notion
- `subject`: `"Introducing Inverted Capital"`
- `body`: rendered email (HTML with hyperlinks)

**8. Retrieve the persistent draft ID.**
`create_draft` returns an `r-XXXX` transaction ID, NOT the persistent hex ID. Call `mcp__claude_ai_Gmail__list_drafts` with `query: "to:{email}"` immediately after creating; the most recent returned entry's `id` (hex format, e.g., `19da8bae7d10166e`) is the persistent ID. Use this for the Notion URL — the `r-XXXX` value doesn't resolve in browser URLs.

**9. Write the draft snapshot to Drive (for the headless `draft-feedback` pipeline).**

Build a snapshot JSON capturing the initial draft state and write it to:

```
~/Library/CloudStorage/GoogleDrive-tom@invertedcap.com/My Drive/draft-snapshots/<hex_id>.json
```

Where `<hex_id>` is the persistent Gmail message ID from Step 8 (e.g., `19da8bae7d10166e`). File contents:

```json
{
  "skill": "founder-outreach",
  "messageId": "<hex_id>",
  "threadId": "<gmail thread id from list_drafts response>",
  "recipient": "<email>",
  "subject": "Introducing Inverted Capital",
  "draftText": "<plain-text body of the draft — strip HTML tags, preserve line breaks, EXCLUDE the signature block from `–` onward since Gmail auto-appends>",
  "createdAt": "<ISO 8601 timestamp>"
}
```

Use the `Write` tool — Drive Desktop syncs the file to the cloud automatically (within seconds). The webhook handler picks it up on send. If Tom never sends, the snapshot auto-purges after 30 days via `purgeOldSnapshots` in the Apps Script.

**On re-drafts** (Tom asked for a tweak — Step 3 deleted the prior Gmail draft, Step 7 created a new one with a new `<hex_id>`): write a NEW snapshot file at the new hex_id path. Stale snapshots auto-purge.

**10. Update Notion**:
- `Gmail Draft URL` → `https://mail.google.com/mail/u/0/#drafts/<hex_id>`
- `Status` → `"Draft Ready"`

**Leave unchanged**: everything else (Eval Score, Eval Breakdown, Signals, Claude Rec, Eval Summary, Working Description — all of which were written by `neg1-enricher` and are out of scope here). The legacy `Email Draft` Notion property is no longer used — the snapshot lives in Drive instead.

**11. Report back.** Per person: name, spike signal, 1-line personalization preview, Gmail draft URL. Summary table for batches.

---

## Important Rules

- **Never score, never apply the rubric.** If the row is unscored, refuse and point Tom at `neg1-enricher`. This skill has ONE job: write the email.
- **Never send.** Only create drafts. `create_draft` is a hard boundary.
- **Never redraft if `Status` is `Reached Out` or `Passed`.** Idempotence prevents clobbering Tom's actions.
- **Never draft without `Claude Rec = "Reach Out ✅"`.** If auto-rec is ❌ and Tom wants to override, he must flip it first.
- **Pull personalization from `Eval Breakdown`, not from scratch.** The breakdown already captures the spike signal and evidence — the draft's job is to render that in Tom's voice.
- **No pattern-match declarations.** Per TEMPLATE.md anti-patterns.
- **En dashes in all prose.** Per Tom's voice preference (memory: feedback_use_en_dash).
- **Report concisely.** Summary table for batches. 3-4 lines max for singletons.

---

## Interaction with neg1-enricher scheduled auto-draft

The `pipeline-agent` Task 6 runs the full pipeline on rows with `Status = Pending Enrichment` — it invokes `neg1-enricher` (which handles enrichment + scoring) and then this skill (for drafting) when Claude Rec comes out as ✅. Both paths — manual via this skill, scheduled via pipeline-agent — use the same TEMPLATE.md scaffolding and the same Notion update logic.

Manual invocation via this skill is for:
- On-demand drafting when Tom doesn't want to wait for the next scheduled run
- Drafting after Tom manually flips Claude Rec from ❌ to ✅ on an auto-passed row

---

## Examples

### Standard draft-only flow

**User:** (after reviewing Greg Reiner's scored -1 Scanner row in Notion) `"draft for greg"`

**Claude:**
1. Finds Greg → Eval Score = 8 (peak Earned Reps), Claude Rec = "Reach Out ✅", Gmail Draft URL empty.
2. Precondition check passes.
3. Deletes any existing Gmail draft for the recipient.
4. Builds personalization anchored on peak Earned Reps (Meta cross-surface — pulled from Eval Breakdown).
5. Creates Gmail draft.
6. Updates Notion: Gmail Draft URL + Email Draft + Status = Draft Ready.
7. Reports: drafted, spike, preview line, Gmail URL.

### Unscored row — refusal

**User:** `"draft for jane smith"`

**Claude:**
1. Finds Jane Smith → Eval Score empty, Claude Rec empty.
2. Refuses: "Jane Smith's -1 Scanner row hasn't been evaluated yet. Run `neg1-enricher` on her profile first — that skill handles enrichment + scoring + the Claude Rec verdict. Once the row shows Eval Score + Claude Rec = ✅, come back to draft."
