---
name: note-classifier
description: Classify a new or existing note in the Notion Notes database into the correct Category (Research, Diligence, Portfolio, Artifact, or Other). Trigger automatically whenever a new note is created in the Notes DB, or when Tom asks to classify or re-classify a note. Also used as a subroutine by other skills (add-conversation-to-notion, investor-update, log-transcript-to-notion, deal-digest, log-investor-letter-to-notion) to set the Category field on pages they create.
---

# Note Classifier

Set the `Category` field on a Notes DB page to one of five values based on the note's content, title, linked Opportunity, and — critically — the close date of that Opportunity.

**Notes DB data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`  
**Opportunities DB data_source_id:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`

---

## Operating Modes

This skill runs in two modes:

- **Mode A** — Scheduled daily sweep (reconciliation pass)
- **Mode C** — Manual (batch or single note) — also used as a subroutine by other skills

### Mode A — Scheduled daily sweep (reconciler)

Runs automatically once per day (scheduled task: `note-classifier-sweep`). **Reconciler role:** the primary path for meeting-note classification + enrichment is now the `notion-webhook` → `meeting-note-processor` pair (which fires on `page.created` in the Notes DB and processes ~45 min later). This sweep is a backstop that catches anything the webhook missed: meeting notes whose webhook delivery failed, non-meeting Notes rows the webhook doesn't gate through, and any other categorization gaps in the past 48 hours.

1. **Query the Notes DB** for all notes created in the last 48 hours (the 48-hour window provides overlap to catch notes created late in the previous day's cycle or any that the webhook missed).
2. **Filter to notes that need attention** — any note where ANY of the following are true:
   - `Category` is NULL/blank
   - The note has the meeting-note shape (title matches `<Person> (<Company>) / Tom (Inverted Capital) @<time>`) AND `Opportunity` relation is empty
   - The note has the meeting-note shape AND `Opportunity` is set AND the linked Opp has empty `Round Details` AND the Opp Status is in-pipeline (not Pass / Active Portfolio / Committed / Portfolio: Follow-On / Exited)

   If none, stop silently — no alert needed.
3. **For each note that needs attention:**
   - **Meeting-note rows**: invoke the `meeting-note-processor` skill (Mode C, treat as `phase: process`) — it handles opportunity match, classification, and Round Details on its own. Read `~/.claude/skills/meeting-note-processor/SKILL.md` and follow Mode B-process logic against that page. The processor's idempotency (skip-if-already-set on every step) means re-running it against partially-processed pages is safe.
   - **Non-meeting rows missing Category**: apply the standard classification logic (Steps 0–5 below). Group by Opportunity presence, dedup Opportunity fetches.
4. **Apply all updates** via `notion-update-page`.
5. **If any work was done**, send a brief summary via the `send-alert` skill. **Frame the alert as a reconciler** — make clear this is catching what the webhook missed:

   ```
   📂 **Note Classifier — sweep (reconciler)**
   **Webhook caught:** <N> meeting notes processed live · **Sweep caught:** <M>
   <details on M items>
   ```

   - Header line + counts line (2 lines for the summary)
   - Then per-item lines if M > 0, formatted compactly:
     - **Meeting notes** (processed via `meeting-note-processor`): use the per-note line
       format returned by the processor:
       `📓 <Note Title> → <Opp> ([Category]) · Intros: [N qualified, N drafts | none found]`
       If any intros were not in the People DB, append: `⚠️ N missing from People DB`
       If no Opportunity was matched: `⚠️ no Opp match`
     - **Non-meeting rows**: `📂 <Note Title> → classified <Category>`
   - If M = 0 (sweep found nothing the webhook hadn't already handled), send a one-liner: `📂 Note Classifier sweep — clean (webhook caught all <N>)` so Tom knows the system is healthy
   - The "Webhook caught" count comes from a quick query: count of meeting-note rows in the last 24h where Category=Diligence AND Opportunity is set AND Round Details on the linked Opp is non-empty (heuristic — these were almost certainly handled by the webhook)

### Mode C — Manual (batch or single note)

Triggered when Tom asks to classify a specific note, a range of notes (e.g. "run note classifier for November and December"), or as a subroutine call from another skill at page-creation time. All three entry points run the same classification logic; the only difference is scope.

**Single note** (subroutine or direct Tom ask): apply the Classification Logic steps below to that single page.

**Batch / retroactive:**

1. **Query the Notes DB** for all notes in the target date range using `notion-search` with a `created_date_range` filter. Fetch up to 25 at a time; paginate if needed.
2. **Filter to unclassified notes only** — notes where `Category` is NULL/blank. Skip notes that already have a category set, unless Tom explicitly asks to reclassify everything.
3. **Group notes by Opportunity presence** to leverage the fast-path (Step 0):
   - Notes **with** an Opportunity relation → these take the fast-path. Collect all unique Opportunity IDs first, then fetch each Opportunity **once** and cache the result (`Status` + `Close Date`). Classify each note directly as Diligence or Portfolio using Step 3 logic — no title or content inspection needed.
   - Notes **without** an Opportunity relation → classify via Steps 1–2, then Step 4–5 (Artifact/Research/Other checks, plus unlinked company matching).
4. **Apply all classifications** in batch using `notion-update-page` calls.
5. **Report a summary** at the end: how many notes were classified, broken down by category, and flag any that required manual judgment.

**Efficiency rule:** Deduplicate Opportunity fetches — if any two or more notes share the same linked Opportunity, fetch that Opportunity exactly once and reuse its `Status` and `Close Date` across all of them. Never fetch the same Opportunity page more than once in a batch run.

---

## Category Definitions

| Category | Description |
|---|---|
| **Research** | Market-wide, non-company-specific knowledge. Investor letters, LP letters, transcripts of public figures, market reports, deal digests, fund quarterly updates, benchmarks. Key test: would this note be useful even if every company in your portfolio ceased to exist? |
| **Diligence** | Work product tied to the evaluation of a *specific opportunity that has not yet been invested in* (or whose note predates the close date). Includes founder calls, feedback notes, reference checks, Claude research/analysis threads, pre-mortems, follow-up notes, and investor decks — all scoped to a single company during the diligence phase. |
| **Portfolio** | Any note created *on or after the Close Date* of the linked Opportunity. This includes portfolio syncs, founder catch-ups, investor updates, board prep, and any other engagement with a company Tom has already committed to. |
| **Artifact** | A Claude-generated output saved as a deliverable — HTML, PPTX, DOCX, structured document. The note is a *thing produced*, not a thing researched or discussed. |
| **Other** | Network calls with no linked Opportunity, LP feedback notes, fund administration documents (auditor financials, changelogs), or anything that doesn't cleanly fit the above four. |

---

## Classification Logic

### Step 0: Opportunity fast-path (check first)

If the note has an **Opportunity relation set**, skip Steps 1–2 entirely and go directly to **Step 3**. A note linked to a specific Opportunity is by definition company-scoped — it cannot be Research (which is market-wide, non-company-specific) or Artifact (which is a standalone deliverable). The only valid outcomes for an Opportunity-linked note are **Diligence** or **Portfolio**, and Step 3 determines which one via the Status short-circuit and Close Date comparison.

This fast-path matters for efficiency: in batch mode it avoids unnecessary title-pattern matching and content inspection for notes whose classification is fully determined by the Opportunity's Status and Close Date. If the note has **no Opportunity relation**, proceed to Step 1.

### Step 1: Check for an Artifact signal (no Opportunity only)

If the note title contains any of the following patterns, classify as **Artifact** immediately:
- "Source HTML", "Skill Map", file type references (e.g. "PPTX", "Deck", "Memo — Draft")
- Or the note was explicitly described by the user as a saved artifact/output

### Step 2: Check for a Research signal (no Opportunity only)

If the note has **no Opportunity relation** AND any of the following apply, classify as **Research**:
- Title starts with "Transcript:", "Letter:", "Deal Digest", "Deals" (deal digest format)
- Title references a fund letter (e.g. "Sequoia Global Equities", "Primary IV: Q", "Counterpoint Global", "Cambridge Associates", "SVB State of", "ICONIQ", "Avenir:", "Carta:")
- Content is clearly a market report, benchmark study, or third-party investor communication not tied to a specific deal

### Step 3: Check the Opportunity relation — Status short-circuit + Close Date comparison

If the note has an **Opportunity relation set**, fetch the linked Opportunity to read its `Status` and `Close Date` fields. Cache the result if processing multiple notes linked to the same Opportunity.

```
Tool: notion-fetch
ID: <opportunity page URL from the note's Opportunity field>
Fields needed: Status, date:Close Date:start
```

**Status short-circuit (check this first, before the date comparison):**

If the Opportunity's `Status` is a passed/dead status — `Pass (Met)`, `Pass (DNM)`, `Pass`, `No Response`, or any variant indicating the deal did not close — then regardless of the Close Date field, classify the note as **Diligence**. These are notes from the evaluation process on companies that were never invested in. Do not apply the Portfolio date logic to passed opportunities.

If the Opportunity's `Status` is a portfolio status — `Active Portfolio`, `Committed`, `Portfolio: Follow-On`, or `Exited` — then apply the Close Date comparison:

- **If Close Date is NULL or blank** → classify as **Diligence** (investment not formally closed yet; treat as in-progress)
- **If note `Created` < `Close Date`** → classify as **Diligence** (note predates the close; it was part of the evaluation process)
- **If note `Created` >= `Close Date`** → classify as **Portfolio** (note was created after the investment closed)

For all other statuses (pipeline stages like `Qualified`, `Outreach`, `Connected`, `Scheduled`, `Diligence`) → classify as **Diligence**.

> **Important:** "Close Date" means the date the investment was formally closed/wired — not the date it was committed. If a note is created the same day as the close date, treat it as Portfolio (>= comparison).

### Step 4: Unlinked company-specific notes (no Opportunity relation but clearly about one company)

If the note has no Opportunity relation but the title or content clearly refers to a single specific company:
- If that company is a known portfolio company (Status = "Active Portfolio", "Committed", "Portfolio: Follow-On", or "Exited"), find the matching Opportunity via `notion-search`, then apply the Status short-circuit and date comparison logic from Step 3.
- If the company is not in the portfolio or can't be matched confidently, classify as **Diligence**.

### Step 5: Other signals

If none of the above apply:
- Unlinked network/relationship calls with no clear opportunity → **Other**
- LP feedback notes → **Other**
- Fund administration documents (auditor financials, changelogs, capital call materials) → **Other**
- Pass note follow-ups (e.g. "Inverted follow up", "Dash follow up") that are linked to a passed opportunity → **Diligence** (these are part of the diligence closure process, not portfolio management)

---

## Step 6: Apply the Classification

Once the category is determined, update the note's Category property:

```
Tool: notion-update-page
Command: update_properties
Page ID: <note page ID>
Properties: { "Category": "<Research|Diligence|Portfolio|Artifact|Other>" }
```

In batch mode, apply all updates sequentially after resolving all classifications. Do not interleave fetches and updates.

---

## Integration with Other Skills

When this skill is used as a subroutine (called by another skill at note creation time), it should run **after** the page is created and the Opportunity relation is set. The classification cannot be determined reliably until the Opportunity link is in place.

Skills that should call this classifier after page creation:
- `add-conversation-to-notion`
- `investor-update`
- `log-transcript-to-notion`
- `log-investor-letter-to-notion`
- `deal-digest`
- `pre-mortem`
- `backchannel-drafter` (for feedback notes it creates)

---

## Ambiguity Rules

| Scenario | Classification |
|---|---|
| Claude thread that does market research *and* deal-specific analysis (e.g. TAM sizing for Tuor) | **Diligence** — company scope wins |
| Founder call note on a committed/portfolio company, note created before close date | **Diligence** — date wins |
| Founder call note on a portfolio company, note created after close date | **Portfolio** |
| Pitch deck or investor deck for a company Tom has not yet invested in | **Diligence** |
| Pitch deck for a portfolio company (e.g. board deck, Series A deck post-close) | **Portfolio** |
| Pass note follow-up (e.g. "Clerical - Inverted follow up") | **Diligence** |
| Recurring portfolio sync (e.g. "Recurring: Oun Homes x DRF x Inverted") after close | **Portfolio** |
| Investor update from a portfolio company | **Portfolio** (handled by investor-update skill; close date check still applies) |
| Note about a company with no Opportunity match found | **Diligence** (default for company-specific content) |
| Network call, no Opportunity linked, no company match | **Other** |
| Note linked to a passed opportunity (any Pass status) | **Diligence** — Status short-circuit applies; do not treat as Portfolio even if Close Date is set |
| Note linked to Active Portfolio opportunity with no Close Date set | **Diligence** — investment not formally closed yet; treat as in-progress regardless of Status |

---

## Key DB IDs and Field References

| Item | Value |
|---|---|
| Notes DB data_source_id | `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb` |
| Opportunities DB data_source_id | `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6` |
| Notes `Category` field | `Category` — SELECT: `Research`, `Diligence`, `Portfolio`, `Artifact`, `Other` |
| Notes `Opportunity` relation | `Opportunity` — links to Opportunities DB |
| Notes `Created` field | `Created` — ISO-8601 datetime, auto-set |
| Opportunities `Close Date` field | `date:Close Date:start` — ISO-8601 date |
| Opportunities `Status` field | `Status` — pass statuses: `Pass (Met)`, `Pass (DNM)`, `Pass`, `No Response`; portfolio statuses: `Committed`, `Active Portfolio`, `Portfolio: Follow-On`, `Exited` |
