---
name: founder-outreach
description: >-
  Drafting primitive for pre-founder (-1) candidates — store mode is the default: invoked inline by
  neg1-sourcing-listener when Tom replies draft to a #neg1-sourcing card. Reads the candidate-store row
  (candidates.py get), generates a Gmail draft per writing-style/outreach/STYLE.md anchored on the spike signal,
  writes the draft URL back to the store, and writes the Drive snapshot for draft-feedback voice learning.
  Never sends. NO Notion reads/writes in this skill — the listener owns Opportunity + eval-note creation at
  draft time. Refuses unscored rows (missing eval_summary/signals_line/email → run neg1-enricher first).
  Manual triggers: "draft outreach for [name]", "draft for [name]". The -1 Scanner DB and Request Draft button
  are RETIRED (2026-07-16).
---

# Founder Outreach

Drafting primitive for pre-founder candidates. Reads a fully-evaluated -1 Scanner row and generates a Gmail draft. Scoring, rubric application, and recommendation all live in `neg1-enricher` — this skill is just the email.

## Invocation modes

Trigger gate values shared with the webhook producer live in `~/.claude/skills/shared-references/triggers/founder-outreach.json` — read it at invocation and use its values; never inline them here (see the triggers README for policy).

**Manual mode** — Tom (or the neg1-enricher manual chain) invokes by name/URL. The default; everything below describes it unless noted.

**Webhook mode** — args contain `mode: "webhook"` and `page_id`. Fired by the notion-webhook Worker when a -1 Scanner row's Status flips to `Draft Requested` (Tom pressing the **Request Draft** button, or flipping the dropdown by hand). Differences from manual mode:

- **Resolve by `page_id`** directly — skip the name/LI search.
- **The button press is an explicit Tom request.** Draft regardless of `Claude Rec` — including `Pass ❌`. Tom flipped the status while looking at the verdict; that IS the override. The Pass-refusal rule below applies only to bare manual trigger phrases.
- **Unscored row** (missing Eval Summary, `Signals` (+ body Eval Rationale), or Email): cannot draft. Set Status back to `Pending Enrichment` (so pipeline-agent Task 6 enriches it on the next sweep), then post a Slack alert via `send-alert` telling Tom the row wasn't scored yet and will re-enter the enrichment queue — the button press must not be silently lost. Exit without drafting.
- **Terminal row** (`Status` already `Reached Out` or `Passed` at read time): exit without drafting, one-line Slack note.
- **On success**: same Step 7–8 as manual (draft + snapshot + Notion writes, Status → `Draft Ready`), then post a one-line Slack alert via `send-alert` with name, spike signal, and the Gmail draft URL (webhook runs are headless — the Step 9 chat report has no reader).


**Store mode (v2, 2026-07-16)** — invoked inline by `neg1-sourcing-listener` on a `draft` reply, with a candidate-store row instead of a -1 Scanner row (`python3 ~/.claude/scripts/decision-ledger/candidates.py get --li <url>`). Differences from manual mode:
- Precondition fields come from the store: `eval_summary`, `signals_line`, `email` must be populated; the personalization anchor is the spike evidence in `eval_summary` / `eval_rationale`.
- NO Notion reads or writes in this skill — the caller handles the Opportunity + eval note. Write the draft URL back with `set-state --gmail-draft-url` instead of a Notion property.
- Drive snapshot write (Step 9) unchanged — draft-feedback voice learning must keep working.
- Never sends, same as always.

**RETIRED 2026-07-16:** the -1 Scanner DB was deleted after full decommission — data archived at `~/.claude/data/neg1_scanner_archive.json` + migrated into the candidate store. Everything below referencing the Scanner is historical documentation only; do NOT query the DB.

## Notion Targets (RETIRED — historical)

- **-1 Sourcing Database** — `collection://32c00bef-f4aa-80a5-923b-000b83921fa3`
- **Companies Database** — `collection://7d50b286-c431-49f5-b96a-6a6390691309`

## Reference file (must read at invocation)

- `~/.claude/skills/writing-style/outreach/STYLE.md` — cold email template (subject, body scaffolding, hyperlinks, formatting rules, personalization-paragraph structure, anti-patterns)

## Precondition check — refuse if unscored

Before doing anything, verify the target row is fully evaluated. Required populated fields on the -1 Scanner row:

- `Eval Summary` — the rationale (includes the bold peak sentence)
- `Claude Rec` — the verdict (`Reach Out ✅` or `Pass ❌`)
- `Signals` — compact per-signal line (the page-body Eval Rationale section carries the evidence for the personalization anchor)
- `Email` — recipient required

**If any of those are missing**, refuse to draft. Tell Tom: "This row hasn't been evaluated yet. Run `neg1-enricher` on it first — that skill now owns enrichment + scoring + the Claude Rec verdict. Then come back to draft." Do not fall back into running the rubric here. Scoring is out of scope for this skill.

**If `Claude Rec = Pass ❌`**, also refuse — UNLESS this run is webhook mode (button press) or the neg1-enricher manual chain, both of which draft regardless of the verdict (the button press / manual scan request is itself the override). For a bare "draft for [name]" trigger, tell Tom the row was auto-passed by neg1-enricher; he can override by pressing Request Draft on the row (or flipping Claude Rec to ✅ and re-invoking).

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

Both files are auto-maintained by the `draft-feedback` pipeline (founder-taste/SYSTEM.md §15). Patterns are observations, not commands.

**5. Build the personalization paragraph** (per writing-style/outreach/STYLE.md "Personalization paragraph — structure"). The anchor is the **spike signal** already identified in `Signals` (+ body Eval Rationale) — do not re-derive it. Pull the specific evidence cited in the breakdown for that peak signal, rewrite in Tom's voice.

Sketches by peak signal:
- Peak Non-Linearity: `"Your path caught my eye – [specific function-to-function crossing and why it's unusual]."`
- Peak Earned Reps: `"Your [specific tenure at hypergrowth company] stood out – [what they saw/built during that window]."`
- Peak Intellectual Rigor: `"Your [specific writing/project/talk] caught my attention – [specific quality — self-correction, framework, depth]."`
- Peak Anticipation: `"You were building [specific project/thesis] well before it became consensus – [pre-consensus timing detail]."`
- Peak Intentionality: `"The way you [specific counter-consensus move — demotion to switch, walked past a kingmaker, turned down accelerator] stood out."`
- Peak Range: `"The [technical-to-commercial / commercial-to-technical] arc at [company] caught my eye – [specific cross-fertilization detail]."`

Follow the career-signal-sentence + no-agenda-frame structure in writing-style/outreach/STYLE.md exactly.

**6. Build full email per writing-style/outreach/STYLE.md** (exact scaffolding, hyperlinks, formatting rules). See writing-style/outreach/STYLE.md for the template block, hyperlink map, signature whitespace, and quote rules.

**7. Create the Gmail draft AND write the snapshot atomically** via `~/.claude/scripts/gmail-create-draft.py`. This single helper replaces the prior three steps (mcp create_draft → list_drafts → manual snapshot Write). It posts to the gmail-webhook `createDraft` endpoint, returns the persistent hex `messageId`, and writes the snapshot to `_system/draft-snapshots/<hex>.json` in one shot. The two writes can no longer decouple — if the snapshot fails, the helper exits non-zero and you MUST treat the whole step as failed.

Write two scratch files first:
- HTML body (rendered email with hyperlinks, includes signature block)
- Plain-text snapshot body (strip HTML tags, preserve line breaks, EXCLUDE the signature block from `–` onward since Gmail auto-appends — this is what the headless `draft-feedback` pipeline diffs against the sent message)

Then invoke:

```
~/.claude/scripts/gmail-create-draft.py \
  --to <person email from Notion> \
  --subject "Introducing Inverted Capital" \
  --html-body-file /tmp/<scratch>.html \
  --snapshot-text-file /tmp/<scratch>.txt \
  --skill founder-outreach
```

Stdout is one JSON line: `{"ok": true, "messageId": "19dd...", "threadId": "...", "draftUrl": "https://mail.google.com/mail/u/0/#drafts/19dd...", "snapshotPath": "..."}`. Use `messageId` and `draftUrl` in Step 8.

Exit code 0 = both writes succeeded. Exit codes 1/2/3 = failure — abort the row, do NOT advance to Step 8, surface the error to Tom.

**On re-drafts** (Tom asked for a tweak — Step 3 deleted the prior Gmail draft): re-run this helper. A new hex is minted, a new snapshot is written. Stale snapshots auto-purge after 30 days via `purgeOldSnapshots` in the Apps Script.

**8. Update Notion**:
- `Gmail Draft URL` → `draftUrl` from Step 7 stdout
- `Status` → `"Draft Ready"`

**Both writes are MANDATORY on every invocation, including the 2nd/3rd/Nth iteration during a workshop session.** Defensive belt-and-suspenders — primary fix is Step 3's iteration-aware skip-deletion, but re-asserting Status here protects against any other event that might have flipped it. See memory `feedback_workshop_iteration_not_pass.md`.

**Leave unchanged**: everything else (`Signals` (+ body Eval Rationale), Claude Rec, Eval Summary, Working Description — all of which were written by `neg1-enricher` and are out of scope here). The legacy `Email Draft` Notion property is no longer used — the snapshot lives in Drive instead.

**9. Report back.** Per person: name, spike signal, 1-line personalization preview, Gmail draft URL. Summary table for batches.

---

## Important Rules

- **Never score, never apply the rubric.** If the row is unscored, refuse and point Tom at `neg1-enricher`. This skill has ONE job: write the email.
- **Never send.** Only create drafts. `create_draft` is a hard boundary.
- **Never redraft if `Status` is `Reached Out` or `Passed`.** Idempotence prevents clobbering Tom's actions.
- **Never draft an auto-passed row from a bare manual trigger.** Webhook mode (Request Draft button) and the neg1-enricher manual chain are the two sanctioned overrides — both draft regardless of Claude Rec.
- **Pull personalization from `Signals` (+ body Eval Rationale), not from scratch.** The breakdown already captures the spike signal and evidence — the draft's job is to render that in Tom's voice.
- **No pattern-match declarations.** Per writing-style/outreach/STYLE.md anti-patterns.
- **En dashes in all prose.** Per Tom's voice preference (memory: feedback_use_en_dash).
- **Report concisely.** Summary table for batches. 3-4 lines max for singletons.

---

## Interaction with the -1 Scanner status flow

The -1 Scanner status pipeline: `Pending Enrichment` → `Enriched` → `Draft Requested` → `Draft Ready` → `Reached Out` / `Passed`.

`pipeline-agent` Task 6 enriches + scores pending rows via `neg1-enricher` and leaves them at `Enriched` — it does NOT draft. Drafting is Tom-initiated: he reviews the scored row and presses the **Request Draft** button (Status → `Draft Requested`), which fires this skill in webhook mode within ~1–2 min. Task 6's sweep also picks up any `Draft Requested` rows the webhook missed. All paths — manual, webhook, sweep — use the same writing-style/outreach/STYLE.md scaffolding and the same Notion update logic.

Manual invocation via trigger phrase is for:
- On-demand drafting when Tom is already in a chat session (skips the button round-trip)
- The neg1-enricher manual-scan chain (drafts regardless of verdict)

---

## Examples

### Standard draft-only flow

**User:** (after reviewing Greg Reiner's scored -1 Scanner row in Notion) `"draft for greg"`

**Claude:**
1. Finds Greg → Earned Reps: High (peak signal), Claude Rec = "Reach Out ✅", Gmail Draft URL empty.
2. Precondition check passes.
3. Deletes any existing Gmail draft for the recipient.
4. Builds personalization anchored on peak Earned Reps (Meta cross-surface — pulled from `Signals` (+ body Eval Rationale)).
5. Creates Gmail draft.
6. Updates Notion: Gmail Draft URL + Email Draft + Status = Draft Ready.
7. Reports: drafted, spike, preview line, Gmail URL.

### Unscored row — refusal

**User:** `"draft for jane smith"`

**Claude:**
1. Finds Jane Smith → Claude Rec empty, `Signals` (+ body Eval Rationale) empty.
2. Refuses: "Jane Smith's -1 Scanner row hasn't been evaluated yet. Run `neg1-enricher` on her profile first — that skill handles enrichment + scoring + the Claude Rec verdict. Once the row shows Claude Rec = ✅, come back to draft."
