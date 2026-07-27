---
name: meeting-note-processor
description: "Process auto-generated Notion AI meeting notes (Zoom/Granola call summaries) that land in the Notes DB. Two webhook phases — `link` runs immediately on page.created and best-effort sets the Opportunity relation from the title, `process` runs ~45 min later (after the meeting wraps and Notion AI fills in the summary) and finishes the job: opportunity match (if not yet set), category classification via note-classifier, Round Details extraction onto the linked Opportunity, AND for portfolio-tagged notes, upsert of the call into the Company Updates period row (one row per Company × Period shared with Formal letters/board decks; rolling Summary/Traction, prior-month freeze for Live content, Formal-wins precedence). Also supports a manual Mode C for retroactive runs against existing notes. Not user-facing in webhook modes — invoked via the `claude-job-queue` primitive by `notion-webhook/notion-meeting-note.js`."
---

# Meeting Note Processor

Handles new Notion AI meeting-note pages that land in the `✏️ Notes` database (auto-created by Notion's meeting feature when Tom is on a Zoom call). Four jobs:

1. **Tag the note against an existing Opportunity** (when the company is identifiable from the title or the call body)
2. **Set the Category** via `note-classifier` (Diligence for live deals, Portfolio for committed/active companies)
3. **Populate Round Details on the linked Opportunity** when extractable from the call body and the Opp's existing Round Details field is empty
4. **Upsert the call into the Company Updates period row** when `Category=Portfolio` — adds the call as a dated subsection to the `Company × Month` row (shared with Formal letters/board decks), regenerates rolling `Summary` + `Traction` (prior-month freeze for Live content; Formal-wins Traction precedence). See `~/.claude/skills/shared-references/company-updates-db.md` for the full design.

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
- Title does NOT match any exclusion pattern (Claude/Reference/Feedback/Backchannel/Pre-Memo/Pre-Mortem/follow-up/Intro/Demo/Deep Research/DDQ/Onboarding/Deck/Pass Notes)
- Opp Name doesn't end with a follow-on parenthetical suffix (`(Series A)`, `(Seed FO)`, etc.) — if it does, swap to the parent investment Opp before continuing

### Step 3: Idempotency check — has this note already been processed?

For each surviving candidate, query the Company Updates DB for the period row where:
- `Company` relation contains the candidate's linked Opp
- `Period` multi-select contains the `Mon YYYY` derived from `note.Created`

(Do NOT filter by Update Type — the row is shared with Formal content and may not carry `Live` yet.)

If a matching row exists AND its `Artifacts` files property includes this note's URL → already processed, drop the candidate. The Artifacts entry is the canonical "this note has been incorporated" marker because `add_link_to_files_property.py` writes it as the last step of B-process Step 6.

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

If the note is a feedback call, rename it to the `feedback-outreach-scanner` title convention and, if a `[PENDING]` stub already exists for the same (person, Opp) pair, port its context into the meeting note and archive the stub. Skip when: `Opportunity` is empty after Step 3, the title is not a Rule 3 `<Person> (<Company>): <Subject>` feedback/backchannel shape, or the note is already in canonical form (idempotent re-run). Full procedure — title re-parse, stub search, body-append placement below `</meeting-notes>`, stub archive, rename, and `📣 Pending Feedback` removal — in `references/feedback-shape-merge.md`; **read it now before proceeding.**

### Step 5: Round Details extraction (only if Opportunity is now linked)

Extract Round Details (and Stage) from the call body onto the linked Opp — but only when the Opp's Round Details is empty AND Status is in-pipeline (skip if already set, or if Status is any `Pass *` / Committed / Active Portfolio / Portfolio: Follow-On / Exited). GUARD — context co-occurrence cross-check: every extracted figure must co-occur with fundraising context ("raising"/"round"/"post"/"cap"/"SAFE"/"valuation") in the same transcript passage, else leave empty and log `figure-without-fundraising-context`. Full procedure + format-spec pointer in `references/round-details-extraction.md`; **read it now before proceeding.**

### Step 6: Upsert call into the Company Updates period row

If `Category = Portfolio` AND `Opportunity` is set, roll the call into the `📚 Company Updates` DB (`collection://bf491fb9-214f-456e-921b-5194b8187f2a`) — one row per Company × Period, shared with Formal content. Upsert the call as a dated subsection and re-derive rolling `Summary`/`Traction` (Case A new row / Case B current-month / Case C prior-month freeze).

**Gate is Opp-side, not Notes-Category-side.** Skip conditions (all must clear before any write): `Opportunity` empty; Opp Status NOT in {Committed, Active Portfolio, Portfolio: Follow-On, Exited}; `Note.Created` BEFORE `Opp.Close Date` (or Close Date empty); title matches a non-meeting pattern (Claude/Reference/Feedback/Backchannel/Pre-Memo/Pre-Mortem/follow-up/Intro/Demo/Deep Research/DDQ/Onboarding/Deck/Pass Notes); or the per-call Sonnet content gate returns `[not a portfolio update – …]` → log `content-gate-skip` and surface in the Step 7 alert.

Guards that run before any Notion write — DO NOT loosen; all live in the reference:
- **Step 6d.0 grounding check (MANDATORY, runs first):** Layer 1 deterministic check (`summary_grounding_check.py`) on the generated Summary — hallucinated/typo'd proper nouns, hallucinated figures, canonical-name typos. Fail → re-prompt Sonnet once; still failing → HALT the write, post the violation to Slack. Under Mode C / manual → HALT immediately.
- **Step 6d.0b semantic grounding (Layer 2, only after Layer 1 passes):** Sonnet-as-judge per clause (`summary_claim_check.py`) — catches negation / misframing / unaddressed claims. Fail → re-prompt once → HALT.
- **Step 6d.0c speaker-attribution check (after Layer 2):** `verify_speaker_attribution.py` — never publish a founder-attributed claim the transcript shows under Tom's speaker label.
- **Step 6d.1 format validation (MANDATORY before any write, Cases A/B):** Traction + Summary validators (lowercase m/k, single `(Mon DD)` date paren, aggregate-metric keyword, no dollar ranges, en-dash-only, no escaped `\$`/`\~`). Same gate for hand-written Mode C values — no bypass. Plus the Notion-AI dashed-range cross-check against the transcript.
- **Idempotency:** re-running against the same note is a body no-op — skip the subsection prepend if the entry body already contains the note's URL.

Full upsert logic (Cases A/B/C), the embedded validator code, the grounding-failure re-prompt templates, and the Notion-AI cross-check are in `references/company-updates-upsert.md`; **read it now before proceeding.**

### Step 7: Send Slack alert

Compose ONE `send-alert` Slack message summarizing what this run did (linked / category / round / live-entry / feedback-merge), eliding parts that didn't change. Suppress entirely when no Opp link, no Round Details, no Live upsert, and no Step 4b action happened this run. Full 2-line format + conventions in `references/slack-alert.md`; **read it now before proceeding.**

### Step 8: Intro extraction subroutine

After the alert fires, invoke `intro-note-processor` (its **Mode B** subroutine flow) to scan the note for intro commitments, stage them on the Opp's `👓 Intros (Qualified)` relation, and save Gmail drafts. Skip if `opp_page_id` is empty or the title starts with `Claude:`. Never fail the parent run on intro-extraction failure — log and continue to Step 9. Full args list + skip/failure handling in `references/intro-extraction.md`; **read it now before proceeding.**

### Step 9: Exit

Exit 0 on success. Exit non-zero only on infrastructure errors (Notion API down, etc.) so the processor moves the job to `failed/` and posts a queue-layer failure alert.

---

## Mode C — Manual / retroactive

Tom passes a Notes page URL or ID conversationally (e.g. "run meeting-note-processor on https://notion.so/...").

Run the full Mode B-process flow against that page. No `not_before` delay; assume body content is already complete (skip the thin-content guard, OR keep it but inform Tom if it triggers).

---

## Meeting-note shape detection (used by the webhook gate, NOT this skill)

Detection + enqueue logic lives in `notion-webhook/notion-meeting-note.js`, not in this skill — gating is the webhook's responsibility; this skill only runs the work given the args. Full detection signals and the two-job enqueue spec (`meeting-note-link` at `now`, `meeting-note-process` at `now + 45min`) are in `references/shape-detection.md`; read it when reasoning about why a note was or wasn't enqueued.

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
