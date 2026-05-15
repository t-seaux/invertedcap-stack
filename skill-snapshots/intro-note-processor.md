---
name: intro-note-processor
description: >
  Process a Notion meeting note for follow-up intro commitments. Reviews the note body
  for explicit intro mentions ("intro X to [Founder]") and implicit intro signals (Tom
  promising to bring in a coinvestor, customer, downstream investor, or advisor). For
  each candidate, resolves the person in the People DB, dedups against all four intro
  lifecycle fields on the linked Opportunity, appends new candidates to 👓 Intros
  (Qualified), and saves a Gmail draft of the outreach email Tom should send to the
  target — using historical sent emails of the same type as the voice/structure
  template. Missing People DB entries surface as a Slack alert (no auto-stub). Two
  modes: (B) Subroutine — called by meeting-note-processor at the end of its Mode
  B-process flow, inheriting the already-fetched note + Opportunity. (C) Manual —
  Tom triggers explicitly. Trigger phrases for Mode C: "process my call with [person]
  for intros", "process the [company] note for intros", "extract intros from [note]",
  "draft intro outreach from this call", or any variant pointing at a specific note
  and asking for follow-up intro extraction.
---

# Intro Note Processor

After Tom takes a call (Notion AI auto-creates a meeting note in the Notes DB, or Tom records a transcript), this skill reviews the note for follow-up intros Tom committed to making, stages them on the linked Opportunity, and saves Gmail drafts of the outreach emails so Tom can review and send.

This is the bridge between "Tom said on a call: I'll intro you to Lauren" and "the outreach email is sitting in Tom's drafts folder." It does NOT send. It does NOT formally intro (that's `intro-draft-agent`'s double-opt-in). It only **stages Qualified + drafts the initial outreach to the target**.

---

## The Notion Data Model

- **Notes DB:** `collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`
- **Opportunities DB:** `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
- **People DB:** `collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`

**Key fields on Opportunity (the four intro lifecycle relations):**
- `👓 Intros (Qualified)` — this skill writes here
- `☎️ Intros (Outreach)` — read for dedup; populated by `intro-outreach-agent` when Tom sends
- `✉️ Intros (Made)` — read for dedup
- `🚫 Intros (Declined / NR)` — read for dedup

**Key fields on Person (People DB):**
- `Name`, `Email`, `Company`, `Role`, `LI`

---

## Operating Modes

- **Mode B — Subroutine.** Called by `meeting-note-processor` at the end of its Mode B-process flow. The caller passes `note_page_id`, `note_body`, `opp_page_id`, `opp_name` so this skill does NOT re-fetch the note or the Opportunity. Skips Step 1's note resolution entirely.
- **Mode C — Manual.** Tom passes a Notes page URL/ID conversationally. Skill resolves the note + Opportunity itself in Step 1, then runs Steps 2–8.

The convention is `B=webhook-triggered, C=manual` per pinned `feedback_skill_mode_abc_convention.md`. Mode A (sweep) is intentionally absent — the meeting-note-processor pipeline already has a daily reconciliation sweep that re-invokes this skill against any note still missing intro processing, so we don't need a separate sweep here.

---

## Args (Mode B)

The caller passes:
- `note_page_id` — Notion page ID of the meeting note
- `note_body` — full text content of the note (already fetched by meeting-note-processor)
- `opp_page_id` — Notion page ID of the linked Opportunity (may be empty if upstream couldn't link)
- `opp_name` — the Opportunity's `Name` value (company name)
- `note_title`, `note_url` — for alert composition

**Skip conditions for Mode B (caller's responsibility, but this skill double-checks):**
- `opp_page_id` empty → skip silently. Without an Opp anchor, we don't know which Opportunity's Qualified relation to write to. Log `no-opportunity-skip` and exit 0. Mode C with explicit user intent can override.
- Note title starts with `Claude:` OR `[Claude]` (LLM-generated artifact) → skip. The meeting-note-processor already filters these but the caller may not always.
- Note body has < 500 chars meaningful content → skip. Same thin-content guard as upstream.

---

## Args (Mode C)

Tom says e.g. "process the Inlets call note for intros" or pastes a Notion URL. Resolve:
- The Notes page (via `notion-fetch` on the URL/ID, or via `notion-search` if Tom only gave a company + date).
- The linked Opportunity from the note's `Opportunity` relation field.
- If the note has no Opportunity link, attempt to infer it from the title using the same priority rules as `meeting-note-processor` Mode B-link Step 2 (Rules 1–4). If still no confident link → ask Tom which Opportunity, do not guess.

---

## Execution Workflow

**Transcript Rule (applies to every `notion-fetch` call on a Notes-DB page in this skill):**
ALWAYS pass `include_transcript: true`. Verbal intro commitments ("yeah I'll connect you with
Sarah") almost always live in the raw transcript and get compressed away by Notion AI's
summary. Without the transcript, this skill's whole reason for existing — catching follow-up
intros — silently misses cases. Param is a no-op on non-meeting-note pages. No exceptions,
no "only if needed" — default ON.

### Step 1: Resolve note + Opportunity (Mode C only)

Mode B inherits these from the caller — skip to Step 2.

Mode C:
1. Fetch the Notes page via `notion-fetch` with **`include_transcript: true`** (MANDATORY — see Transcript Rule below). Extract `Name`, `Opportunity` relation, full body INCLUDING the `<transcript>` block.
2. If `Opportunity` is empty, run the title-parsing rules from `meeting-note-processor` Mode B-link Step 2 (priority order: meeting-note format → `<X> Feedback` / `<X> Backchannel` → colon-suffixed → paren fallback) to infer the company. Search Opportunities DB; require a confident single match. If ambiguous → ask Tom and stop.
3. Apply the same skip conditions (Claude prefix, thin content, terminal Pass status).

Carry forward: `note_page_id`, `note_body`, `note_title`, `note_url`, `opp_page_id`, `opp_name`.

### Step 2: Identify intro candidates from the note

Two parallel passes over `note_body`:

**Pass 2a — Explicit mentions.** Regex/string-scan for phrases like:
- `intro <Name>`, `introduce <Name>`, `connect you with <Name>`, `connect <Name>`
- `bring in <Name>`, `loop in <Name>`
- `I can put you in touch with <Name>`, `I'll reach out to <Name>`
- LinkedIn URLs adjacent to a name
- Bulleted/numbered lists of names following an "intros:" / "people to connect with:" header

For each, capture `(name, surrounding context sentence, optional company/role hints from adjacent text)`.

**Pass 2b — Implicit signals (LLM classification).** Pass the full `note_body` to a Sonnet call with this prompt:

```
Read this call note. Identify any intro commitments Tom made — places where Tom
said he would connect the founder to a specific person, or where the conversation
implies Tom would help bring in a coinvestor, customer, partner, downstream
investor, or advisor as a follow-up.

For each intro commitment, return JSON with:
- name: the person Tom said he'd intro (or the descriptor if no name was given,
  e.g. "head of growth at Stripe" — flag these as "unnamed")
- type: one of "coinvestor", "customer", "partner", "downstream_investor",
  "advisor", "other"
- context: the verbatim sentence(s) from the note grounding this intro
- explicit: true if a specific name was named, false if generic ("someone who
  knows the SMB GTM playbook")

Skip generic statements like "happy to help with intros down the road" with no
specific person or type. Return empty array if no commitments found.
```

Merge 2a and 2b: for any name that appears in BOTH, prefer the LLM-typed entry (it has the type tag). Drop unnamed/generic entries — we can't act on "head of growth at Stripe" without a specific person.

If the merged list is empty → log `no-intro-candidates` and exit 0 (silent, no Slack alert).

### Step 3: Pull historical context for the Opportunity

Two queries, both scoped to the Opportunity:

**Query A — All Notes tagged to this Opportunity.** Use `notion-search` against the Notes DB filtered to `Opportunity contains <opp_page_id>`, sorted by Created desc, limit 10. Read titles + first 200 chars of each. Used in Step 4 to detect intros Tom may have already offered/discussed in prior calls (corner case: same person came up two calls ago, was committed verbally, but Tom never made the actual outreach).

**Query B — Past outreach emails Tom sent for this Opportunity.** Gmail search:
```
in:sent (subject:"would love to intro" OR subject:"want to chat" OR subject:"want to connect" OR subject:"intro request" OR subject:"up for an intro") <opp_name>
```
Limit 10. Read subject + first 300 chars. Used in Step 6 as voice/structure templates for new drafts AND in Step 5 to dedup against intros already in flight.

### Step 4: Dedup against current Opp state + recent history

Fetch the Opportunity page once (Mode C: re-fetch; Mode B: caller already has the link but not the four lifecycle relations — fetch them now):

```
Tool: notion-fetch
Page ID: <opp_page_id>
Fields: 👓 Intros (Qualified), ☎️ Intros (Outreach), ✉️ Intros (Made), 🚫 Intros (Declined / NR), Status, Name, Contact
```

For each candidate from Step 2:
1. Resolve to a People DB page (Step 5 below).
2. Check all four lifecycle fields — if the person's page URL appears in any, drop them with status:
   - `already-qualified` → already staged, no action
   - `already-outreach` → Tom already reached out, drafts not needed
   - `already-made` → intro already completed
   - `previously-declined` → drop
3. Cross-check against Query A (prior notes) — if the candidate was named in a prior note for the same Opp BUT is NOT in any of the four lifecycle fields, that's a corner case worth surfacing in the alert (`prior-mention-but-untracked`). Still process them this run.
4. Cross-check against Query B (past sent emails) — if a sent email exists addressed to this person about this Opp BUT they're not in any lifecycle field, surface as `outreach-sent-but-untracked` and skip this skill's draft step (intro-outreach-agent's sweep should pick it up). Don't add to Qualified — they're effectively already at Outreach.

### Step 5: Resolve each candidate in the People DB

Two-pass per the convention from `log-intro` Step 3:

**Pass 1 — Scoped DB search:**
```
notion-search query="<person name>" data_source_url="collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9"
```

**Pass 2 — Workspace search fallback:**
```
notion-search query="<person name>" query_type="internal" content_search_mode="workspace_search"
```
Filter results to People DB pages.

**Disambiguation:** if multiple People DB candidates match by name, use context from the call note (company, role hints) to pick the right one. If still ambiguous, mark as `ambiguous-match` and skip this candidate (Slack alert lists for Tom to resolve manually).

**If person NOT found in People DB:**
- Do NOT create a stub. Per pinned memory `feedback_no_people_entry_without_permission.md`, missing people require explicit permission + ContactOut enrichment + photo before a People DB entry is created.
- Surface in the Slack alert (Step 8) with the candidate's name, the inferred type, and the verbatim context sentence from the note. The alert is the "note to Tom" — Tom decides whether to add them to People DB and re-run Mode C against the same note.
- Skip the draft step for this candidate (no email on file anyway).
- Do NOT write a placeholder to the Opportunity's page body. Page-body writes risk clobbering existing content; the Slack alert is the canonical surface for unresolved candidates.

### Step 5.5: Pre-Write Guards (MANDATORY)

Before any Qualified write, every candidate must pass these gates:

**Gate 1 — Directionality.** Confirm the meeting note describes intros Tom (or the Opp's founder) is committing to make to a third party. If the note instead describes someone else offering to intro Tom INBOUND to an Opp (e.g., a coinvestor on a call said "I can intro you to X"), that's a deal-sourcing signal for `add-to-crm`, NOT a Qualified candidate for this Opp. Skip with `direction-inbound-skip` and surface in the alert.

**Gate 2 — Terminal-status skip.** If the Opp's `Status` ∈ `{Pass (DNM), Pass (Met), Pass Note Pending, Lost, NR / Missed, Exited}`, skip ALL candidates for this Opp. Log: `[Opp] terminal status [status] — skipping N candidates`. Closed Opps should not accumulate new intro lifecycle entries. (Active Portfolio status is fine — portfolio companies still get intros.)

**Gate 3 — Common-word Opp name corroboration.** If the Opp's `Name` is a single common English word or short token (≤6 chars) — `Current`, `Scout`, `Pulse`, `Echo`, `Core`, `Pillar`, `Arc`, `Atlas`, `Compass`, etc. — and the Opp link was set via title-based matching upstream (meeting-note-processor Mode B-link), require that the note body corroborates the Opp identity: founder name appearing in the note, `Contact` email domain appearing, or explicit framing ("@CompanyName", "[CompanyName Inc.]"). If no corroboration, skip all candidates with `ambiguous-common-word-opp` and surface in the alert for Tom to re-link manually.

If any gate fails, write nothing for that Opp and surface in the Slack alert (Step 8).

### Step 6: Append to 👓 Intros (Qualified) — single batched write

Collect all resolved-and-deduped candidates that need to be added (i.e., found in People DB AND not in any lifecycle field).

Read the current `👓 Intros (Qualified)` JSON array from the fetch in Step 4. Compose the new array as `existing + new_candidates`, deduplicated by page URL. Write once:

```
Tool: notion-update-page
Command: update_properties
Page ID: <opp_page_id>
Properties: { "👓 Intros (Qualified)": "<JSON array string of all page URLs>" }
```

The relation field expects a JSON array string (e.g. `"[\"https://www.notion.so/p1\",\"https://www.notion.so/p2\"]"`). Always include existing entries — passing only the new entries replaces the relation and wipes existing Qualified intros.

### Step 7: Draft an outreach email per new Qualified person

For each newly-Qualified person, save ONE Gmail draft. The draft is to the target person ONLY (not the founder — that's the double-opt-in stage handled later by `intro-draft-agent`). This is Tom asking the target if they're up for chatting with the company.

**7a. Recipient resolution.** Use the person's `Email` from the People DB. If empty → skip the draft for this person, log `no-target-email`, surface in Slack alert.

**7b. Subject line.** Universal format:
```
<opp_name> — would love to intro
```
The em-dash is actually an en-dash (`–`) per pinned `feedback_use_en_dash.md` — the file source uses `–`. Render as: `Inlets – would love to intro`.

This phrasing is intentional: `would love to intro` is one of the canonical `OUTREACH_SUBJECT_PHRASES` recognized by `intro-outreach-agent`'s Gmail scanner (`intro-outreach-agent/SKILL.md`). When Tom sends this draft, the outreach scanner will detect it and flip the person from Qualified → Outreach automatically. **Do not customize the subject** — the universal phrasing is what makes the downstream automation work.

**7c. Voice + structure templates.** Pull 3–5 of Tom's actual past outreach emails matching this candidate's `type`:

```
in:sent subject:"would love to intro" newer_than:180d
```

Filter the results by inferred type — for `coinvestor`, look for VC firm names in the recipient domain; for `customer`/`partner`, look for operator-style domains; for `downstream_investor`, similar to coinvestor but later-stage signals; for `advisor`, look for emails to senior individual contacts. Read the body of each match. Use them collectively as voice reference — not as a literal template, since Tom's emails vary.

If fewer than 3 matches exist for the type, fall back to the broader `would love to intro` corpus as the voice reference.

**7d. Body composition.** Write the body as a single short message — typically 3–5 sentences — that:
1. Greets the target by first name (`Hi <First>,` — no comma vs. dash conventions; use a comma).
2. Names the company and gives a one-sentence description sourced from the note + Opp context (the company's product/positioning, not a sales pitch).
3. Names the connection between the company and the target — WHY this intro is interesting for the target specifically (the bucket-specific framing — see 7e).
4. Asks if they're open to a quick chat with the founder(s), light-touch (`If you're open to it, happy to make the intro` / `lmk and I'll connect you both`).
5. Signs `Tom`. No signature block (Gmail appends).

Use en dashes (`–`), never em dashes (`—`). Don't use the "Sent from my iPhone" phrasing. Match Tom's casual lowercase-feel from the historical examples — not formal VC firm speak.

**7e. Bucket-specific framing.** Type-conditional micro-template for sentence 3 of the body:

- `coinvestor` — "We're [raising / closing] a [round size] [stage], [lead status], and I thought of you given [your focus on X / your check at Y / your portfolio overlap with Z]."
- `customer` — "I think [Company] could be a fit for [target's company]'s [problem / workflow], and the founder [name] is sharp."
- `partner` — "There's a clean integration / distribution angle with [target's company] that I think the founder would value getting your read on."
- `downstream_investor` — "They're tracking towards a [Series A / B] in [rough timeline] and I'd love to put you on their radar early."
- `advisor` — "Would love to get your read on [the specific area where target has expertise], and I think the founder would benefit from your perspective."
- `other` — generic, lean on the note context.

The framing should be GROUNDED in specific note content — don't invent product details. If the note doesn't contain enough material to write a confident sentence 3, leave the framing soft (`thought you two should know each other`) and surface in the Slack alert as `thin-context-draft` so Tom knows to thicken before sending.

**7f. Save the draft.**
```
Tool: gmail_create_draft
to: <target_email>
subject: "<opp_name> – would love to intro"
body: <composed body>
```

**7g. Idempotency check.** Before creating the draft, check Gmail drafts:
```
Tool: gmail_list_drafts
```
Scan subjects for an exact match on `<opp_name> – would love to intro` to `<target_email>`. If a draft already exists, skip — log `draft-already-exists`.

Also check sent mail (caught in Step 4's Query B already, but a second cheap check):
```
in:sent subject:"<opp_name> – would love to intro" to:<target_email>
```
If a sent email exists, skip the draft AND skip adding to Qualified (the person is effectively at Outreach already; intro-outreach-agent's sweep handles it).

### Step 8: Slack alert

Compose ONE Slack alert via `send-alert`. Read `~/.claude/skills/send-alert/SKILL.md` for delivery and format. Use the per-entity row convention (no bullets, two-line, `🧍` emoji for people).

**Alert body:**

```
🤝 <u>**INTRO NOTE — <opp_name>**</u>
[note](<note_url>) · [opp](<opp_url>)

🧍 <u>**<Person Name> | [Notion](<person_url>) | [draft](<gmail_draft_url>)**</u>
**Type:** coinvestor · **Status:** Qualified ✓ + draft saved

🧍 <u>**<Person Name> | [Notion](<person_url>)**</u>
**Type:** customer · **Status:** ⚠️ not in People DB — please add
**Context:** <one-line context sentence from the note>

🧍 <u>**<Person Name> | [Notion](<person_url>)**</u>
**Type:** advisor · **Status:** ⏭️ already in Outreach — no action
```

Conventions:
- Use `🧍` emoji for each candidate row (per pinned `reference_slack_notification_channel.md` row convention).
- Status values:
  - `Qualified ✓ + draft saved` — happy path
  - `⚠️ not in People DB — please add` — surfaced for Tom; one-line context line follows
  - `⚠️ thin-context-draft — review before sending` — draft was saved but framing is soft
  - `⏭️ already in <stage> — no action` — dedup hit
  - `⚠️ ambiguous People DB match — please clarify` — multiple candidates with same name
  - `⚠️ no email on file — add to People DB or send manually` — found in DB but no `Email` field
- For `[draft]` link: Gmail draft URLs follow `https://mail.google.com/mail/u/0/#drafts/<draft_id>`. Use the draft ID returned by `gmail_create_draft`.

**Suppress the alert entirely when:**
- Step 2 found zero candidates AND Mode C wasn't manual (silent for Mode B subroutine runs that find nothing).
- Mode C with zero candidates → still emit a one-line `_no intro candidates found_` body so Tom gets confirmation his manual trigger ran.

### Step 9: Exit

Exit 0 on success. Exit non-zero only on infrastructure errors (Notion API down, Gmail API errors). Never block the meeting-note-processor caller with non-fatal errors — log them and continue.

---

## Idempotency

- Step 4 (lifecycle dedup) prevents double-staging.
- Step 6 (Qualified write) is JSON-array-merge — re-running with the same input is a no-op.
- Step 7g (draft dedup by subject + recipient) prevents duplicate drafts.
- Step 8 (`send-alert`) has a 1h body-hash dedupe at the helper layer.

So re-running this skill against the same note (e.g. meeting-note-processor's reconciliation sweep tomorrow) is safe.

---

## Edge cases

- **Note tagged to a portfolio Opportunity (post-investment) — process anyway.** Portfolio companies still get intros (customers, advisors, downstream investors). Active Portfolio / Committed / Connected / Scheduled / Track / Qualified / Outreach are all valid — process candidates normally. Only terminal statuses (Pass DNM / Pass Met / Pass Note Pending / Lost / NR / Missed / Exited) gate writes per Step 5.5 Gate 2.

- **Multiple intros to the same person across different notes for the same Opp.** Idempotency in Step 6 handles this — second mention is a no-op since the person is already in Qualified.

- **Tom committed to an intro but the target was named without a company in the note.** If People DB has a unique match by name → resolve and use. If multiple → mark ambiguous and surface in the alert.

- **The same person across different Opportunities.** Each Opportunity gets its own evaluation — the same person can legitimately be Qualified for two Opps simultaneously. No cross-Opp dedup.

- **Founder of the Opportunity itself shows up in the candidate list.** Skip — the founder is already on the Opp's `🏁 Founder(s)` relation, they shouldn't be a Qualified intro for their own company. Filter out anyone already in `🏁 Founder(s)` before Step 5.

- **Mode B caller passes empty `opp_page_id`.** Skip silently per the skip condition above. The note's intros are unanchored without an Opp link — Tom can re-run Mode C manually once the Opp link is set.

---

## Why universal subject line

The subject `<Company> – would love to intro` is universal across all four intro types because:

1. `intro-outreach-agent` scans Gmail sent mail for `OUTREACH_SUBJECT_PHRASES` (`would love to intro` is one). When Tom sends the draft, that scanner detects it and moves the person Qualified → Outreach automatically — no manual flip needed.
2. Tom doesn't want to think about subject phrasing per-intro. The body carries the per-intro context.
3. The downstream `intro-draft-agent` (double-opt-in step) uses a different subject format (`<Founder> (Company) / <Target> (Target Co)`) — there's no collision.

Don't customize the subject. The voice variation lives in the body.

---

## Integration with meeting-note-processor

This skill is invoked as a subroutine at the end of `meeting-note-processor` Mode B-process. The caller passes `note_page_id`, `note_body`, `opp_page_id`, `opp_name`, `note_title`, `note_url` so this skill skips its own fetches.

The integration point is documented in `meeting-note-processor/SKILL.md` (Step 8 — Intro extraction). Failures here are non-fatal to the caller.
