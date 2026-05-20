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

- **Mode A** — Scheduled reconciliation sweep, runs daily at 18:06 ET via launchd. Catches webhook drops (cf-queue retry exhaustion, Notion delivery misses, worker exceptions) by re-processing any portfolio meeting note from the last 48h that doesn't yet have a Company Updates Live entry.
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

Before any step runs in any mode, check if the note's title starts with `Claude:` OR `[Claude]` (case-sensitive). These are LLM-generated artifacts (e.g. `Claude: Caplight First-Pass Diligence — 04.22.2026` from older runs, or `[Claude] Aerion First-Pass Diligence — 05.13.2026` from `first-pass-diligence`), not real meeting notes — even when they end up in the Notes DB and get linked to a portfolio Opportunity. Skip the entire pipeline: no Opportunity link, no Category classification, no Round Details extraction, no Live Company Updates upsert. Log `claude-prefix-skip` and exit 0.

---

## Mode A — Reconciliation sweep (daily, catches webhook drops)

Background: the notion-webhook delivers `page.created` events to a Cloudflare Worker, which retries D1 enqueue 4× with backoff (`cf-queue.ts`). Two failure modes still leak through:

1. **D1 outage longer than the ~1.5s retry budget** — `writeJobToD1` exhausts attempts. The worker fires a `:rotating_light:` Slack alert via `alertSlackOnGiveUp` so Tom hears immediately, but the job itself is lost.
2. **Notion doesn't deliver the webhook at all** — silent miss; no Slack signal from the worker because the worker never ran.

Mode A is the safety net for both. It runs daily at 18:06 ET via launchd (`~/Library/LaunchAgents/com.tomseo.scheduled.meeting-note-processor-sweep.plist` → `~/.claude/scheduled-tasks/meeting-note-processor-sweep/run.sh`). The scheduled wrapper just reads canonical-skill Mode A; the actual logic lives here.

### Step 1: Query candidate notes

Query the Notes DB (`collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) for pages where:
- `Created` is within the last **48 hours**
- `Opportunity` relation is non-empty

Use `notion-search` with a `created_date_range` filter scoped to the Notes data source. 48h gives a generous buffer beyond the 45-min `not_before` delay so even late-evening calls from yesterday are covered.

### Step 2: Apply portfolio gate

For each candidate, fetch the linked Opportunity. Drop the candidate unless ALL of these hold (mirrors B-process Step 6's gate — see `~/.claude/skills/shared-references/company-updates-db.md` "Skip / exclusion rules" section for the canonical list):

- Opportunity Status ∈ {`Committed`, `Active Portfolio`, `Portfolio: Follow-On`, `Exited`}
- `note.Created >= Opp.Close Date` (post-investment only — pre-close calls are diligence, not portfolio updates)
- Title does NOT match any exclusion pattern (Claude/Reference/Feedback/Backchannel/Pre-Memo/Pre-Mortem/follow-up/Intro/Demo/Deep Research/DDQ/Onboarding/Deck)
- Opp Name doesn't end with a follow-on parenthetical suffix (`(Series A)`, `(Seed FO)`, etc.) — if it does, swap to the parent investment Opp before continuing

### Step 3: Idempotency check — has this note already been processed?

For each surviving candidate, query the Company Updates DB for Live entries where:
- `Update Type = Live`
- `Company` relation contains the candidate's linked Opp
- `Period` multi-select contains the `Mon YYYY` derived from `note.Created`

If a matching entry exists AND its `Artifacts` files property includes this note's URL → already processed, drop the candidate. The Artifacts entry is the canonical "this note has been incorporated" marker because `add_link_to_files_property.py` writes it as the last step of B-process Step 6.

If no matching entry OR matching entry exists but doesn't reference this note → this is a real gap; keep the candidate.

### Step 4: Heal each gap via Mode B-process

For each remaining candidate, run the full B-process flow against its `pageId`. B-process is idempotent across its Steps 3/4/5/6, so any partially-completed work (e.g., Opp linked but Category not set) gets finished cleanly. Step 6's body-content guard prevents duplicate subsection prepend if a parallel B-process race appended the call already.

Run candidates sequentially within this Mode A invocation — no parallel sub-skill spawning. The volume is bounded (≤ a few notes/day in steady state) and sequential keeps Notion rate-limit pressure bounded.

### Step 5: Slack alert — silent on green

If ≥1 gap was healed, post one consolidated alert via `~/.claude/skills/send-alert/send.sh`. GFM links, never Slack mrkdwn (per pinned memory `feedback_send_alert_gfm_not_mrkdwn.md`):

```
🛡 **Meeting-note sweep healed N drop(s)**
- [Company A](opp-url) — [note](note-url) → [Live entry](entry-url)
- [Company B](opp-url) — [note](note-url) → [Live entry](entry-url)
```

If 0 heals (the happy path — webhook worked all day) → silent. No alert. Tom doesn't want a daily "0 drops" pulse.

### Step 6: Reconciliation manifest

Emit one JSONL file to `~/.claude/scheduled-tasks/reconciliation/inbox/meeting-note-processor-sweep-YYYY-MM-DD-HHMMSS.jsonl` for the admin-agent orchestrator. One record per candidate touched (healed, observed-already-done, or skipped). Always include `entity_url`:

```json
{"ts": "<iso8601>", "source_task": "meeting-note-processor-sweep", "entity_id": "<note_page_id>", "entity_name": "<note_title>", "entity_url": "<note_notion_url>", "db": "NOTES", "transition": "Note: → Company Updates Live entry upserted", "outcome": "wrote|observed_already_in_state|skipped", "detail": "<entry url or skip reason>"}
```

Empty-manifest policy: zero candidates → empty file (still write it).

### Step 7: Exit

Exit 0 on success. Mode A NEVER ask questions — runs unattended per the standard Unattended Execution Guard (`~/.claude/scheduled-tasks/SHARED_SAFETY.md`). On infrastructure errors that prevent reaching Step 5, log to the run log AND attempt the Slack alert in degraded form (`🛡 **Meeting-note sweep crashed** — see logs`) so silent failures are impossible.

---

## Mode B-link — Title-based Opportunity match (fast, runs immediately)

### Step 1: Read the note

Fetch the page via `notion-fetch` (do NOT pass `include_transcript: true` here — this phase
runs at +0 min on `page.created`, before Notion AI has processed the call, so the transcript
doesn't exist yet, and the link decision uses only the title anyway). Extract:
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

**Transcript Rule (scoped):** Pass `include_transcript: true` on the meeting-note fetch in
this mode. The transcript is consumed by **Step 5d (Round Details extraction)** for spoken-
metric cross-check ("$1.2M ARR" vs. summary's dashed "$1-2M" artifact) and by the **Live
Company Updates upsert (Step 6)** which builds Summary/Traction strings — both benefit from
transcript-grounded content over summary-only. Category classification (Step 3) and
Opportunity relink (Step 4) use title + body and do NOT depend on the transcript, but a
single fetch with the param covers both needs at the same cost. Do NOT re-fetch later for the
transcript — one fetch, transcript included.

### Step 1: Read the note + body

Fetch the page via `notion-fetch` with **`include_transcript: true`**, including the page body
content. Extract:
- `Name` (title)
- `Opportunity` relation (current value — may have been set by Job A)
- `Category` (current value — may have been set by note-classifier sweep already)
- Page body — full text content of the meeting note (Notion AI's auto-generated Action Items / Summary / etc.)
- **`<transcript>` block content** — raw call transcript, used downstream by Step 5d's metric cross-check and Step 6's Live Company Updates upsert

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
- **Transactional-line gate** — if EVERY occurrence of the candidate token in the first 2000 chars is inside an email-metadata line (`Subject:`, `From:`, `To:`, `Cc:`, `Bcc:`, `Re:`, `Fwd:`, or markdown variants like `**Subject:**`, `**From:**`), reject the match. Email subjects persist across thread drift — a reply on an old `OpenAI <> Outmarket` thread that's actually about OpenAI's portfolio offer is NOT about Outmarket. The candidate must appear in at least one substantive (non-metadata) line of the body. Log `body-match-only-in-email-headers:<candidate>` and treat as no match.
- **Primary-subject LLM gate** — before accepting any body-match (after the above gates pass), classify whether the candidate Opp is the PRIMARY subject of the note. Inputs: note title, first 2000 chars of body, candidate Opp `Name`. Output one of: `SUBJECT` (note is about this Opp), `MENTIONED_NOT_SUBJECT` (Opp is referenced but isn't the main topic — e.g. "we've considered partnering with X but this call is about Y"), `NOT_PRESENT` (false match). Accept only `SUBJECT`. On any other result, log `body-match-not-primary-subject:<candidate>` and treat as no match. This is the general catch-all for body matches where the candidate appears in substantive prose but isn't the focus — backstops the cheap filters above.
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

### Step 4b: Feedback-shape merge (rename + port stub context + archive)

This step handles the case where the meeting note IS a feedback call (Tom hopped on a Zoom with someone who's providing feedback on an Opp). Two goals:
1. Rename the meeting note to match the `feedback-outreach-scanner` title convention so format stays consistent across the Notes DB.
2. If a `[PENDING]` stub note was already created by `feedback-outreach-scanner` for the same (person, Opp) pair (outreach went out by email, then Tom hopped on a call), port the stub's context (person mention + LinkedIn + outreach body) into the meeting note and archive the stub.

**Skip conditions:**
- `Opportunity` is empty after Step 3 → skip; we have no anchor.
- Title does NOT match the `<Person> (<Company>): <Subject>` shape with `Feedback`/`Backchannel` in the subject → skip. This step targets Rule 3 feedback titles specifically; standalone `<X> Feedback` (Rule 2) has no person to attribute and isn't renamed.
- The note is already in the canonical `[Opp]: [Person] ([Company]) Feedback` form → skip (idempotent re-run).

**Step 4b.1 — Re-parse the title** using Rule 3 to extract:
- `person_name` — text before the `(`
- `person_company` — text inside the parens
- `subject` — text after the `): ` up to the trailing `Feedback`/`Backchannel`

Example: `Gilad Rom (OnOrder): Factir Feedback` → `person_name=Gilad Rom`, `person_company=OnOrder`, `subject=Factir`.

If parsing fails (multiple people, malformed parens, no trailing Feedback/Backchannel), log `feedback-merge-unparseable` and skip.

**Step 4b.2 — Search for an existing stub** on the linked Opp's `✍️ Notes` array. Fetch each note's title and check (case-insensitive) for:
- `[PENDING] {subject}: {person_name}` (prefix match — current scanner convention) OR
- `[PENDING] {subject} – {person_name}` (older en-dash variant) OR
- `[PENDING] {subject} — {person_name}` (defensive: em-dash legacy)

If multiple `[PENDING]` matches exist for the same (subject, person_name), pick the EARLIEST by `Created` timestamp; archive the others as part of Step 4b.5 cleanup.

**Step 4b.3 — Compose the canonical new title:**
```
{subject}: {person_name} ({person_company}) Feedback
```
Example: `Factir: Gilad Rom (OnOrder) Feedback`. No `[PENDING]` prefix — the call transcript IS substantive feedback.

**Step 4b.4 — Append stub body to the meeting note (only if a stub was found):**

Placement matters: the outreach context goes **BELOW** the meeting-notes transcript block, NOT above. Content inserted inside the `<meeting-notes>...</meeting-notes>` widget renders as inline text within the widget (markdown headers/dividers collapse into `<br>` line breaks instead of separate blocks). Appending below the widget keeps the outreach text as proper sibling blocks with real header/divider rendering.

Use `update_content` with `</meeting-notes>` as the anchor — this is the closing tag of the meeting-notes block in the page's markdown representation. Match it exactly and replace with `</meeting-notes>\n\n---\n\n<appended content>`:

1. Fetch the stub's full body content.
2. Extract the person header line (the `<mention-page url="..."/> ([LinkedIn](...)) — Role, Company. [background]` line at the top).
3. Extract the `## Outreach Note — [Date sent]` section and its verbatim body (everything from that header to end of stub, or to the next `---` separator).
4. Discard the stub's `## Response — [No reply yet]` placeholder — the meeting transcript IS the response.
5. Compose the appended content:
   ```
   ---

   {person header line}

   ---

   ## Outreach Note — [Date sent]

   {verbatim outreach body from stub}
   ```
6. Issue `notion-update-page` with `command: update_content`, `content_updates: [{old_str: "</meeting-notes>", new_str: "</meeting-notes>\n\n{appended content}"}]`. This places the appended blocks as siblings AFTER the meeting-notes widget — they render with proper block formatting (real `##` headers, real `---` dividers).

If no stub was found, skip the body append — just rename the title in Step 4b.6.

**Step 4b.5 — Archive the stub (only if a stub was found):**
1. Use the Notion REST API pattern from `feedback-outreach-scanner` Step 3b: `PATCH /v1/pages/{stub_page_id}` with `{"archived": true}`. Token decryption pattern: `cd ~/code/notion-backup && SOPS_AGE_KEY_FILE=... python3 -c "from export import get_token; ..."`.
2. Re-fetch the Opp's `✍️ Notes` array and remove the archived stub's URL.
3. Write the updated array back.
4. If Step 4b.2 found multiple `[PENDING]` matches (rare race-condition leftover), archive each.

**Step 4b.6 — Rename the meeting note:**
```
notion-update-page
  command: update_properties
  page_id: <meeting note ID>
  properties: { "Name": "{subject}: {person_name} ({person_company}) Feedback" }
```

**Step 4b.7 — Remove the person from `📣 Pending Feedback` (only if a stub was found and merged):**
The stub's existence proves the person was on the Opp's `📣 Pending Feedback` list. A call transcript is substantive feedback by definition — same logic as `feedback-outreach-scanner` Step 5.

1. Parse the stub's person mention to extract the People DB page URL (the `<mention-page url="..."/>` at the top of the stub's body).
2. Fetch the Opp's current `📣 Pending Feedback` array.
3. Remove the person's People DB page URL from the array.
4. Write the updated array back.

If the stub's person mention can't be parsed (malformed), log `feedback-merge-pending-removal-unparseable` and continue — the rename + body merge + archive still happened, and the daily scanner sweep can reconcile `📣 Pending Feedback` later.

**Step 4b.8 — Log the action:**
- `feedback-merge: stub <stub_url> archived, body ported, renamed to "{new_title}", removed <person> from Pending Feedback on <opp>` (full merge case)
- `feedback-merge-rename-only: no stub found, renamed to "{new_title}"` (consistency rename case)

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
4. Add the note's Notion page URL to `Artifacts` via `python3 ~/.claude/scripts/notion_files_property.py --page-id <new_entry_id> --prop "Artifacts" --url <note_url> --label "<note title>"`.

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

**Step 6d — Pre-write format validation (MANDATORY before any Notion write in Cases A/B):**

Before calling `notion-create-pages` (Case A) or `notion-update-page` (Case B) with `Summary` + `Traction`, run the validator below against BOTH the Sonnet-generated values AND any hand-written values (Mode C / manual / debugging — same gate, no bypass). Case C skips this step because rolling fields stay frozen.

This guard exists because Live `Traction` has a strict shape (canonical reference: `~/.claude/skills/shared-references/company-updates-db.md` "Track 2" section + FORBIDDEN counter-examples block). Past failure modes include: multi-line bullets, per-customer ACVs, uppercase `M`/`K`, missing date paren, and accepting Notion-AI mis-transcriptions of spoken numbers (e.g., founder says "$1.2m" → AI summary writes "$1-2M" → un-cross-checked Traction landed wrong).

**Traction validator (Python — run via inline `python3 -c` before write):**

```python
import re, sys

t = (traction or "").strip()

# 1. Single-line — no newlines, no carriage returns
if "\n" in t or "\r" in t:
    raise ValueError(f"Traction must be single-line; got multi-line value: {t!r}")

# 2. Allowed terminal values
if t in ("N/A", "$0"):
    pass  # short-circuit OK
else:
    # 3. Must contain at least one aggregate metric keyword (case-sensitive)
    METRIC_RE = re.compile(r"\b(ARR|MRR|CARR|Live ARR|GMV|Revenue|ACV|NDR|NRR|HC)\b")
    if not METRIC_RE.search(t):
        raise ValueError(f"Traction must contain an aggregate metric keyword (ARR/MRR/CARR/Live ARR/Revenue/etc.): {t!r}")

    # 4. No uppercase M or K directly following a digit/period
    if re.search(r"[\d.]\s*[MK]\b", t):
        raise ValueError(f"Traction must use lowercase m/k, not uppercase M/K: {t!r}")

    # 5. Must contain a date paren (Mon DD format) — single date paren only
    DATE_PAREN_RE = re.compile(r"\((?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec) \d{1,2}\)")
    parens = DATE_PAREN_RE.findall(t)
    if len(parens) != 1:
        raise ValueError(f"Traction must contain exactly one (Mon DD) date paren; got {len(parens)}: {t!r}")

    # 6. Suspicious-range guard — if the metric value looks like a range like "$1-2m"
    # or "$3-5m", that's almost always a Notion-AI mis-transcription of a single number
    # (e.g., founder says "$1.2m" → AI writes "$1-2M"). Range values are forbidden in
    # Traction — either pick the canonical single figure or use `N/A`. (Real ranges
    # belong in Summary as prose, never in Traction.)
    if re.search(r"\$\d+-\d+[mk]\b", t):
        raise ValueError(f"Traction must not contain a dollar range; resolve to a single figure or N/A. Got: {t!r}")
```

**Summary validator (lighter — Summary is prose):**

```python
s = (summary or "").strip()

# 1. No uppercase M / K following digits in dollar amounts (lowercase-only convention)
if re.search(r"[\d.]\s*[MK]\b(?!\.)", s):  # the negative lookahead skips abbreviations like "M.D."
    raise ValueError(f"Summary must use lowercase m/k for dollar amounts; got uppercase: {s!r}")

# 2. No escaped tildes or dollar signs (per pinned `feedback_no_escaped_tildes.md`)
if "\\$" in s or "\\~" in s:
    raise ValueError(f"Summary must not contain escaped \\$ or \\~ — write plain: {s!r}")

# 3. No em dashes (per pinned `feedback_use_en_dash.md`)
if "\u2014" in s:
    raise ValueError(f"Summary must use en dash –, not em dash —: {s!r}")
```

**On validation failure:**
1. If running under Sonnet output (the normal flow): log the violation, re-prompt Sonnet ONCE with the validator error message appended to the prompt as a corrective hint. If the re-prompt still fails, halt the write and surface via Slack alert (`**Validation:** ✗ {field} failed: {reason}`) — leave the entry untouched.
2. If running under Mode C / manual / debugging: HALT immediately. Do not "fix it manually and proceed" — the whole point of the guard is to catch instinct-overrides. Surface to Tom in the response with the bad value and the rule it violated.

**Notion-AI cross-check before write:** if the Notes body contains the founder's stated metric in narrative form (e.g., "ARR run rate at 1-2M" in the body), treat dashed dollar ranges as suspect transcription artifacts. Cross-check against the `<transcript>` block (which Step 1 fetched via `include_transcript: true`) — find where the founder actually says the number and use that figure verbatim. The transcript IS the ground truth here; only halt and ask Tom if the transcript itself is ambiguous (founder genuinely said "one to two million") or absent (rare — only if the call had no audio).

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
- `**Feedback merged:** ✓` if Step 4b ported a stub's context into this meeting note and archived the stub. If Step 4b only renamed the title (no stub found), use `**Renamed:** ✓` instead.

**Suppress the alert entirely when:**
- No Opp link was made this run AND no Round Details was written this run AND no Live entry upsert happened AND no Step 4b action happened.

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
