---
name: meeting-note-processor
description: "Process auto-generated Notion AI meeting notes (Zoom/Granola call summaries) that land in the Notes DB. Two webhook phases — `link` runs immediately on page.created and best-effort sets the Opportunity relation from the title, `process` runs ~45 min later (after the meeting wraps and Notion AI fills in the summary) and finishes the job: opportunity match (if not yet set), category classification via note-classifier, Round Details extraction onto the linked Opportunity, AND for portfolio-tagged notes, upsert of a Live entry in the Company Updates DB (one row per Company × Month, rolling Summary/Traction with prior-month freeze). Also supports a manual Mode C for retroactive runs against existing notes. Not user-facing in webhook modes — invoked via the `claude-job-queue` primitive by `notion-webhook/notion-meeting-note.js`."
---

# Meeting Note Processor

Handles new Notion AI meeting-note pages that land in the `✏️ Notes` database (auto-created by Notion's meeting feature when Tom is on a Zoom call). Four jobs:

1. **Tag the note against an existing Opportunity** (when the company is identifiable from the title or the call body)
2. **Set the Category** via `note-classifier` (Diligence for live deals, Portfolio for committed/active companies)
3. **Populate Round Details on the linked Opportunity** when extractable from the call body and the Opp's existing Round Details field is empty
4. **Upsert a Live entry in the Company Updates DB** when `Category=Portfolio` — appends the call as a dated subsection to a `Company × Month` rollup, regenerates rolling `Summary` + `Traction` (with a prior-month freeze rule). See `~/.claude/skills/shared-references/company-updates-db.md` for the full design.

**Notes DB data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`
**Opportunities DB data_source_id:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
**Company Updates DB data_source_id:** `bf491fb9-214f-456e-921b-5194b8187f2a`

---

## Operating Modes

- **Mode B-link** — Webhook, phase=`link`, runs immediately on Notion `page.created`. Cheap, title-only opportunity match.
- **Mode B-process** — Webhook, phase=`process`, runs ~45 min after page creation (gated on `not_before` in the queue payload). Heavy lift: opportunity match (if still empty), classify, Round Details.
- **Mode C** — Manual. Tom passes a Notes page URL or ID; runs the full B-process logic against that page. Used for retroactive runs and validation.

The two webhook phases are dispatched as separate queued jobs by `notion-webhook/notion-meeting-note.js`. Each job is keyed by `meeting-note-link:<page_id>` and `meeting-note-process:<page_id>` respectively, so re-deliveries collapse and Phase B firing later does not skip Phase A.

---

## Args

The skill accepts these args in webhook modes:

- `pageId` (required) — Notion page ID of the meeting note (also accepts a full Notion URL; extract the ID).
- `phase` (required for webhook) — `"link"` or `"process"`.
- `source` (optional) — defaults to `"notion-webhook"`.

Manual mode (Mode C): Tom passes a page URL/ID conversationally; treat as `phase: "process"`.

---

## Pre-flight: Skip Claude-generated artifact notes

Before any step runs in any mode, check if the note's title starts with `Claude:` (case-sensitive). These are LLM-generated artifacts (e.g. `Claude: Caplight First-Pass Diligence — 04.22.2026`), not real meeting notes — even when they end up in the Notes DB and get linked to a portfolio Opportunity. Skip the entire pipeline: no Opportunity link, no Category classification, no Round Details extraction, no Live Company Updates upsert. Log `claude-prefix-skip` and exit 0.

---

## Mode B-link — Title-based Opportunity match (fast, runs immediately)

### Step 1: Read the note

Fetch the page via `notion-fetch`. Extract:
- `Name` (title)
- `Opportunity` relation (current value)
- `Category` (current value)

If `Opportunity` is **already set** → log `already-linked-skip` and exit 0. Job B will still run later for category + Round Details.

### Step 2: Parse company from title

Apply title-parsing rules in this exact priority order. Pick the **first** rule that matches.

**Rule 1 — Meeting-note format** (`<Counterparty Name> (<Company>) / Tom (Inverted Capital) @<DateOrTime>`):

```python
import re
m = re.match(r'^[^(]+\(([^)]+)\)\s*/\s*Tom\s*\(Inverted Capital\)', title)
if m: company = m.group(1).strip()
```

In this format the paren company **IS** the subject (the call is about that company). Examples:
- `Emily (Inlets) / Tom (Inverted Capital) @Today 10:00 AM` → `Inlets`
- `Sarah Chen (Acme) / Tom (Inverted Capital) @Apr 27, 2026 2:00 PM` → `Acme`

**Rule 2 — `<X> Feedback` / `Feedback on <X>` / `<X> Backchannel`:** the subject is `<X>`. Examples: `Inlets Feedback` → `Inlets`. `Feedback on Acme` → `Acme`. `Lex Backchannel` → `Lex`.

**Rule 3 — `<Person> (<Company>): <Subject>`** (colon-suffixed): the company in parens is the **source/context provider**, not the subject. Match to `<Subject>`, NOT to `<Company>`. Example: `Sara (Sequoia): Inlets feedback` → match `Inlets`, not `Sequoia`.

**Rule 4 — Paren company as fallback:** only use the paren company as the match candidate when no other rule above produced a subject and the title contains a clear `(<Company>)` capture.

If no rule produces a candidate → log `no-company-from-title` and exit 0. Job B will retry against the body.

**Confidence rule:** only emit a link if confidence is **high**. Don't guess. Better to leave empty and let Job B / the daily sweep retry.

### Step 3: Match against Opportunities DB

Use `notion-search` scoped to Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`):

```
Tool: notion-search
Query: <company name from title>
Filter: data_source_id == fab5ada3-5ea1-44b0-8eb7-3f1120aadda6
```

Match logic:
- **Single confident match** (exact name OR very close stem match, e.g. "Inlets" → "Inlets Inc") → use it.
- **Multiple candidates** with one clearly dominant (exact case-insensitive name match wins over fuzzy) → use the dominant one.
- **Multiple ambiguous candidates** OR **zero matches** → log `ambiguous-or-no-match`, exit 0. Job B retries with body context.

**Hard exclusion:** if the matched Opportunity has a terminal `Pass *` status (`Pass`, `Pass (DNM)`, `Pass (Met)`), do NOT link — log `matched-but-passed` and exit 0. Tom doesn't want call notes auto-resurrecting passed deals.

### Step 4: Set the Opportunity relation

```
Tool: notion-update-page
Command: update_properties
Page ID: <note page ID>
Properties: { "Opportunity": <opp page ID> }
```

### Step 5: Exit silently

No Slack alert at this phase. Job B will produce a single consolidated alert later. Log `linked` to the run log with the matched Opp name + ID.

---

## Mode B-process — Full processor (heavy, runs +45 min)

### Step 1: Read the note + body

Fetch the page via `notion-fetch`, including the page body content. Extract:
- `Name` (title)
- `Opportunity` relation (current value — may have been set by Job A)
- `Category` (current value — may have been set by note-classifier sweep already)
- Page body — full text content of the meeting note (Notion AI's auto-generated Action Items / Summary / etc.)

### Step 2: Thin-content guard

If the page body has fewer than **500 characters** of meaningful content (excluding section headers), log `thin-content-skip` and exit 0. The meeting probably ran long, hasn't generated a summary yet, or Notion AI is still processing. The daily note-classifier sweep will catch it tomorrow as a reconciler.

Heuristic: strip markdown headers and bullet glyphs, count the remaining characters. If < 500, skip.

### Step 3: Opportunity match (if still empty)

If `Opportunity` is empty:

**3a. Retry title-based match** (in case the title was edited/normalized after Job A ran):
- Run the same title regex from Mode B-link, Step 2.
- If a confident single Opps DB match exists (and not a `Pass *` status), use it.

**3b. If title match still fails, try body-based match:**
- Scan the first ~2000 chars of the body for capitalized noun phrases that look like company names. Notion AI summaries typically have a "Company Overview" section that names the company directly.
- Pull explicit candidate names (e.g. "Emily founded a GEO platform called Inlets" → candidate "Inlets").
- Search Opportunities DB for each candidate; require an exact (case-insensitive) name match. Do NOT accept fuzzy body-derived matches — too noisy.
- **Common-word disambiguation gate** — if the matched Opp's `Name` is a single common English word or short token (≤6 chars) like `Current`, `Scout`, `Pulse`, `Echo`, `Core`, `Pillar`, `Arc`, `Atlas`, `Compass`, etc., require corroboration before accepting the body match: founder name from the Opp's `🏁 Founder(s)` relation appearing in the body, OR the Opp's `Contact` email domain appearing in the body, OR explicit framing nearby in the body ("@CompanyName", "called CompanyName", "the company CompanyName"). Without corroboration, skip — bare common-word matches are too prone to false positives (e.g. a note discussing "current ARR" or "scouting talent" should not match a closed `Current` or `Scout` Opp).
- If a confident match found and not `Pass *` status, use it.

If still no match → log `unlinked-after-process` and proceed to Step 4 (classification can still happen on unlinked notes; Round Details can't).

If matched, set the relation:

```
Tool: notion-update-page
Properties: { "Opportunity": <opp page ID> }
```

### Step 4: Category classification

If `Category` is already set → skip.

If `Category` is empty → run the `note-classifier` skill as a subroutine. Read `~/.claude/skills/note-classifier/SKILL.md` and apply Mode C single-note logic to this page.

The note-classifier handles its own Opp lookup + Status/Close Date logic. It will set Update Type to `Diligence` for live deals, `Portfolio` for committed/active companies, or `Other` if no Opp link.

### Step 5: Round Details extraction (only if Opportunity is now linked)

If `Opportunity` is empty → skip this step.

If `Opportunity` is set:

**5a. Fetch the Opp's current Round Details + Stage:**

```
Tool: notion-fetch
ID: <opp page URL>
Fields: Round Details, Stage, Status
```

**5b. If Round Details is non-empty → SKIP. Do not overwrite.** Log `round-details-already-set` with the existing value. Tom curates this manually after first set.

**5c. If the Opp Status is `Pass *` (any pass variant), Active Portfolio, Committed, Portfolio: Follow-On, or Exited → SKIP.** Round Details is only meaningful for in-pipeline deals. Log the skip.

**5d. If Round Details is empty AND status is in-pipeline:**

Read the canonical format spec at `~/.claude/skills/shared-references/round-details-format.md` and follow it strictly. Extract from the call body:
- Look for funding signals: dollar amounts, "raising", "round", "post", "cap", "SAFE", "priced", "lead", "valuation"
- Apply the format rules (lowercase `m`/`k`, `Raising $Xm` for unfinalized, `$Xm on $Ym post` or `$Xm on $Ym cap` for finalized)
- If nothing extractable → leave empty, log `no-round-details-signal`

**5e. Also extract Stage** if confidently inferrable from the same content (per the Stage table in the format spec). Only write Stage if currently empty on the Opp.

**5f. Update the Opp:**

```
Tool: notion-update-page
Page ID: <opp page ID>
Properties: { "Round Details": "<extracted>", "Stage": "<extracted>" }
```

(Omit Stage from the patch if not extractable or already set.)

### Step 6: Upsert Live Company Updates entry

If `Category = Portfolio` AND `Opportunity` is set, this step rolls the call into the Live track of the `📚 Company Updates` DB (`collection://bf491fb9-214f-456e-921b-5194b8187f2a`). Live entries aggregate one row per `Company × Month`. Read the canonical reference at `~/.claude/skills/shared-references/company-updates-db.md` for the full design.

**Gate is Opp-side, not Notes-Category-side.** Note-classifier has gaps — it tagged 13/19 portfolio meetings in Jan 2026 as Diligence/None instead of Portfolio (discovered 2026-05-01 during the broader-filter sweep). The canonical filter is the linked Opportunity's Status. Notes Category is informational, not a hard gate.

**Skip conditions:**
- `Opportunity` empty — can't determine the target row.
- Opportunity Status NOT in {`Committed`, `Active Portfolio`, `Portfolio: Follow-On`, `Exited`} — non-portfolio Opps don't get Live entries (covers `Pass *`, `Lost`, `NR / Missed`, `Qualified`, `Outreach`, `Connected`, `Scheduled`, etc.).
- **`Note.Created` date is BEFORE `Opportunity.Close Date`.** Status is current state, not historical — a deal that's `Active Portfolio` today may have had diligence calls 6 months ago BEFORE the close. Those pre-close calls are diligence, NOT portfolio updates. Reject if `note_created_date < opp.close_date`. If `Close Date` is empty (deal hasn't closed yet despite Status), also reject — Live entries only apply post-investment.
- Title matches a non-meeting pattern (case-insensitive, applied before pre-flight):
  - starts with `Claude` (catches `Claude:`, `Claude Thread:`, `Claude Pre-Mortem:`)
  - contains `Reference` (backchannel reference calls about a portco's founder, not portfolio meetings)
  - contains `Feedback` or `Backchannel` (third-party feedback notes about a portfolio company)
  - contains `Pre-Memo` or `Pre-Mortem`
  - contains `follow up` / `follow-up` (typically post-pass diligence followups, not portfolio meetings)
  - contains `Intro` (Tom intro'ing third parties for/about a portfolio company)
  - contains `Demo` (incl. `Loom Demo`) — product demo videos / sessions
  - contains `Deep Research` / `DDQ` / `Onboarding` / `Deck` — research, diligence, or administrative artifacts

**Step 6a — Compute the target group:**
- `target_opp` = the linked Opportunity (page ID + page URL + Name)
- `target_month` = note's `Created` date formatted as `Mon YYYY` (e.g., `Apr 2026`). Must be one of the existing options in the `Period` multi-select.
- `target_title` = `{opp_name} – {Mon YYYY}` — note: en dash `–`, NOT em dash, per pinned style memory.
- `current_month_key` = today's calendar month, same `Mon YYYY` format.

**Step 6b — Search for an existing Live entry for this group:**
Query Company Updates DB scoped to `Update Type=Live` matches where `Company contains <target_opp>` AND `Period contains <target_month>`. Best path: `notion-search` with the title `{opp_name} – {Mon YYYY}` (exact-match prefix). If a single match returns, use it. If multiple ambiguous candidates, fetch each and disambiguate by exact `Company` relation match.

**Step 6c — UPSERT logic:**

**Case A — entry doesn't exist:**
1. Generate per-call summary via Sonnet — adaptive 2–5 markdown bold-prefixed sections (`**Product:**`, `**GTM:**`, `**Customers:**`, etc.) covering only topics the call actually addressed. Prompt template in `~/.claude/skills/shared-references/company-updates-db.md`.
2. Generate cumulative `Summary` + `Traction` via Sonnet using just this one call's content.
3. Create the Live entry with:
   - `Name = target_title`
   - `Update Type = Live`
   - `Period = ["{target_month}"]`
   - `Company = ["{opp_url}"]`
   - `date:Update Date:start = note.Created` (date only)
   - `Summary`, `Traction` (Sonnet output, en dashes only, inline date parens at end of clauses for Summary; per-line `(Mon DD)` stamps for Traction bullets)
   - `content` = single dated subsection: `### {Mon} {DD} – Call ([Notes]({note_url}))\n\n{per_call_summary}`
4. Add the note's Notion page URL to `Artifacts` via `python3 ~/.claude/local-agents/notion-internal/add_link_to_files_property.py --page-id <new_entry_id> --prop "Artifacts" --url <note_url> --label "<note title>"`.

**Case B — entry exists, `target_month == current_month_key`:**
1. Generate per-call summary via Sonnet (same prompt as Case A).
2. Fetch the existing entry's body content. Prepend a new dated subsection (newest on top): `### {Mon} {DD} – Call ([Notes]({note_url}))\n\n{per_call_summary}\n\n` then existing body.
3. Re-derive cumulative `Summary` + `Traction` via Sonnet using ALL subsections (existing body + new subsection).
4. Update the entry: `replace_content` with the new body; `update_properties` to overwrite `Summary`, `Traction`, and `date:Update Date:start` (set to the new note's date if newer).
5. Add the note's URL to `Artifacts` (idempotent — helper skips dupes).

**Case C — entry exists, `target_month` is a prior calendar month:**
**Frozen rolling-fields rule** (per pinned memory `feedback_live_entry_prior_month_freeze.md`): once we cross into a new calendar month, prior-month Live entries' Summary and Traction are immutable. Body subsections can still be appended for late transcripts.
1. Generate per-call summary via Sonnet.
2. Fetch existing body content. Prepend the new dated subsection.
3. `replace_content` with the new body. **Do NOT** update `Summary`, `Traction`, or `Update Date`.
4. Add the note's URL to `Artifacts`.
5. Log `prior-month-frozen-skip-rolling-fields` so the alert in Step 7 reflects what happened.

**On Sonnet failure (per-call or cumulative):** if either Sonnet call returns malformed output, missing JSON keys, or fails outright, leave the existing entry untouched (Cases B/C skip the body update too) and log a warning. Do NOT write garbage. The daily sweep can re-run this step against the note tomorrow.

**Idempotency:** all three cases are idempotent. Re-running this step against the same note with the same body is a no-op for the body (subsection prepend would produce a duplicate header — guard by checking whether the entry's body already contains the note's URL; if so, skip).

### Step 7: Send Slack alert

Compose ONE Slack alert via `send-alert` summarizing what this run did. Read `~/.claude/skills/send-alert/SKILL.md` for delivery and format.

**Alert body — exactly 2 lines, no header, no extra spacing:**

```
📓 <u>**<Company> | [note](<note URL>) | [opp](<opp URL>)**</u>
**Linked** ✓ · **Cat:** Diligence · **Round:** Raising $500k
```

Line 2 elides parts that didn't change. Use `·` as the separator. Conventions:
- `**Linked** ✓` if Job A or B set the Opp; omit if it was already linked before this run
- `**Cat:** <value>` only if Category was just set this run; omit if already set
- `**Round:** <value>` only if Round Details was just written; omit if already set or skipped
- `**Live:** <Company> – <Mon YYYY>` only if Step 6 created or appended to a Live entry this run. If frozen-prior-month case applied, append `(prior-month, body-only)` so Tom can tell at a glance the rolling fields weren't touched.

**Suppress the alert entirely when:**
- No Opp link was made this run AND no Round Details was written this run AND no Live entry upsert happened.

This means category-only classification on a non-meeting note (a Notes-DB row that the webhook fired on but isn't actually a meeting note) is silent. The note-classifier daily sweep already reports on category counts; no need to double up here.

### Step 8: Intro extraction subroutine

After the Slack alert fires, invoke `intro-note-processor` as a subroutine to scan the note for follow-up intro commitments Tom made on the call, stage them on the Opportunity's `👓 Intros (Qualified)` relation, and save Gmail drafts of the outreach. Read `~/.claude/skills/intro-note-processor/SKILL.md` and run its **Mode B** (Subroutine) flow.

Pass these args (the caller already has them in memory — no re-fetches needed):
- `note_page_id`, `note_title`, `note_url`
- `note_body` (the same body content fetched in Step 1)
- `opp_page_id`, `opp_name` (or empty if Step 3 left the note unlinked)

**Skip conditions** (intro-note-processor's caller-side guard, but enforce here too):
- `opp_page_id` is empty after Step 3 → skip; intros need an Opp anchor.
- Note title starts with `Claude:` → skip (already filtered by pre-flight, but defensive).
- Step 2 thin-content guard already triggered → won't reach this step.

**Failure handling:** intro-note-processor emits its own Slack alert (separate from Step 7's). If it errors, log the error and continue to Step 9 — never fail the parent meeting-note-processor run on intro-extraction failure. The note's primary processing has already succeeded.

### Step 9: Exit

Exit 0 on success. Exit non-zero only on infrastructure errors (Notion API down, etc.) so the processor moves the job to `failed/` and posts a queue-layer failure alert.

---

## Mode C — Manual / retroactive

Tom passes a Notes page URL or ID conversationally (e.g. "run meeting-note-processor on https://notion.so/...").

Run the full Mode B-process flow against that page. No `not_before` delay; assume body content is already complete (skip the thin-content guard, OR keep it but inform Tom if it triggers).

---

## Meeting-note shape detection (used by the webhook gate, NOT this skill)

The `notion-webhook/notion-meeting-note.js` handler decides whether a `page.created` event in the Notes DB warrants enqueueing this skill. Detection signals (broad — see project memory `feedback_subagent_web_perms.md` and the user-confirmed broad-detector decision):

1. Page is in the Notes DB (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`)
2. AND any of:
   - Title matches `^.+\s*\(.+\)\s*/\s*Tom\s*\(Inverted Capital\)\s*@`
   - Page contains a Notion AI meeting block (`ai_block` type) — detected by the body shape from the webhook payload
   - Body has the Action Items + Summary skeleton (best-effort; only if the payload includes content)

If gated through, the handler enqueues two jobs:
- `meeting-note-link:<pageId>` with `not_before = now`, args `{pageId, phase: "link"}`
- `meeting-note-process:<pageId>` with `not_before = now + 45min`, args `{pageId, phase: "process"}`

This skill only runs the work given the args — gating is the webhook's responsibility.

---

## Idempotency

Every step is idempotent against re-runs:

- Step 3 (Opp link) skips if already set
- Step 4 (Category) skips if already set
- Step 5 (Round Details) skips if already non-empty
- Step 6 (Slack alert) suppresses if everything was already in place

So running this skill multiple times against the same page is safe. The note-classifier daily sweep can re-invoke this skill against any meeting note that still has gaps without producing duplicate work or duplicate alerts.

---

## Error handling

- **Notion fetch fails** → log + exit non-zero (transient, processor retries on next tick).
- **note-classifier subroutine fails** → log the error; continue with Round Details step. Don't let a classifier hiccup block the more valuable Round Details write.
- **Round Details extraction LLM ambiguous** → leave empty rather than guessing. False positives are far worse than false negatives.
- **No matching Opportunity found** → fine; the note still gets classified (Mode C path) and the daily sweep can retry tomorrow.
