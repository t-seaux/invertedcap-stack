---
name: founder-outreach
description: >-
  Drafting primitive for pre-founder (-1) candidates. Reads an already-evaluated -1 Scanner
  row (Eval Score + Eval Breakdown + Claude Rec populated by neg1-enricher) and generates a
  Gmail draft using the canonical writing-style/outreach/STYLE.md cold email format, personalization anchored on
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

- `~/.claude/skills/writing-style/outreach/STYLE.md` — cold email template (subject, body scaffolding, hyperlinks, formatting rules, personalization-paragraph structure, anti-patterns)

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

**3. Handle existing drafts (delete-old-after-Notion-points-to-new).**
Gmail's `create_draft` does NOT dedupe. **Whenever a NEW draft is created (i.e., not editing an existing draft in place), the old draft MUST be deleted** so Tom's drafts folder shows one canonical draft per recipient. The webhook smartening shipped to `gmail-webhook` v73+ (2026-04-28) makes this safe: the pass-detection pipeline now compares the deleted draft's hex against the Notion row's current `Gmail Draft URL` — if they don't match, the discard event is recognized as orphan cleanup and ignored.

Sequencing matters — this is the safe ordering:
1. Call `mcp__claude_ai_Gmail__list_drafts` with `query: "to:{email}"` to detect existing drafts.
2. Create the NEW draft first (Step 7) and capture its persistent hex (Step 8).
3. **Update Notion's `Gmail Draft URL` to the NEW hex (Step 10).** This MUST happen before the deletion. The webhook's URL-match check reads Notion at the time the deletion event is processed; if Notion still points to the OLD hex when the delete fires, the smartening sees a "match" and treats it as a real Tom-discard.
4. Only THEN delete the prior draft via `/Users/tomseo/.claude/skills/shared-references/delete-gmail-draft.sh <old_hex>`. The webhook fires `messageDeleted(old_hex)`, looks up Notion, sees URL contains NEW hex (not OLD) → mismatch → skip-not-active-draft → no spurious pass.

**Edit-in-place exception (future):** if the skill ever uses Gmail's `users.drafts.update` API to mutate an existing draft in place, no deletion is needed (and the webhook smartening also handles the same-batch `messagesAdded(DRAFT)` companion event). Today the skill uses `create_draft` only, so this path is not yet active.

See memories `feedback_workshop_iteration_not_pass.md` and `feedback_gmail_draft_persistent_id.md`.

**4. Read the outreach stylebook.**
- `~/.claude/skills/writing-style/outreach/STYLE.md` — canonical scaffold + hyperlinks + formatting rules + personalization principle + anti-patterns. Read in full.
- `~/.claude/skills/writing-style/outreach/EDIT_PATTERNS.md` — two sections: **Canonical Principles** (durable, foundational rules — apply as hard rules) and **Recent Edits** (append-only log of how Tom edits Claude-drafted outreach — apply as priors, then check the final draft against them). Read both sections in full.
- `~/.claude/skills/writing-style/outreach/VOICE_EXAMPLES.md` — full sent emails Tom wrote from scratch (no Claude draft involved). Scan the 2–3 most recent for canonical voice. Use as ground truth — if your draft sounds nothing like these, recalibrate.

Both files are auto-maintained by the `draft-feedback` pipeline (FOUNDER_EVAL_FRAMEWORK.md §13). Patterns are observations, not commands.

**5. Build the personalization paragraph** (per writing-style/outreach/STYLE.md "Personalization paragraph — structure"). The anchor is the **spike signal** already identified in `Eval Breakdown` — do not re-derive it. Pull the specific evidence cited in the breakdown for that peak signal, rewrite in Tom's voice.

Sketches by peak signal:
- Peak Non-Linearity: `"Your path caught my eye – [specific function-to-function crossing and why it's unusual]."`
- Peak Earned Reps: `"Your [specific tenure at hypergrowth company] stood out – [what they saw/built during that window]."`
- Peak Intellectual Rigor: `"Your [specific writing/project/talk] caught my attention – [specific quality — self-correction, framework, depth]."`
- Peak Anticipation: `"You were building [specific project/thesis] well before it became consensus – [pre-consensus timing detail]."`
- Peak Intentionality: `"The way you [specific counter-consensus move — demotion to switch, walked past a kingmaker, turned down accelerator] stood out."`
- Peak Range: `"The [technical-to-commercial / commercial-to-technical] arc at [company] caught my eye – [specific cross-fertilization detail]."`

Follow the career-signal-sentence + no-agenda-frame structure in writing-style/outreach/STYLE.md exactly.

**6. Build full email per writing-style/outreach/STYLE.md** (exact scaffolding, hyperlinks, formatting rules). See writing-style/outreach/STYLE.md for the template block, hyperlink map, signature whitespace, and quote rules.

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

**Both writes are MANDATORY on every invocation, including the 2nd/3rd/Nth iteration during a workshop session.** Defensive belt-and-suspenders — primary fix is Step 3's iteration-aware skip-deletion, but re-asserting Status here protects against any other event that might have flipped it. See memory `feedback_workshop_iteration_not_pass.md`.

**Leave unchanged**: everything else (Eval Score, Eval Breakdown, Signals, Claude Rec, Eval Summary, Working Description — all of which were written by `neg1-enricher` and are out of scope here). The legacy `Email Draft` Notion property is no longer used — the snapshot lives in Drive instead.

**11. Report back.** Per person: name, spike signal, 1-line personalization preview, Gmail draft URL. Summary table for batches.

---

## Important Rules

- **Never score, never apply the rubric.** If the row is unscored, refuse and point Tom at `neg1-enricher`. This skill has ONE job: write the email.
- **Never send.** Only create drafts. `create_draft` is a hard boundary.
- **Never redraft if `Status` is `Reached Out` or `Passed`.** Idempotence prevents clobbering Tom's actions.
- **Never draft without `Claude Rec = "Reach Out ✅"`.** If auto-rec is ❌ and Tom wants to override, he must flip it first.
- **Pull personalization from `Eval Breakdown`, not from scratch.** The breakdown already captures the spike signal and evidence — the draft's job is to render that in Tom's voice.
- **No pattern-match declarations.** Per writing-style/outreach/STYLE.md anti-patterns.
- **En dashes in all prose.** Per Tom's voice preference (memory: feedback_use_en_dash).
- **Report concisely.** Summary table for batches. 3-4 lines max for singletons.

---

## Interaction with neg1-enricher scheduled auto-draft

The `pipeline-agent` Task 6 runs the full pipeline on rows with `Status = Pending Enrichment` — it invokes `neg1-enricher` (which handles enrichment + scoring) and then this skill (for drafting) when Claude Rec comes out as ✅. Both paths — manual via this skill, scheduled via pipeline-agent — use the same writing-style/outreach/STYLE.md scaffolding and the same Notion update logic.

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
