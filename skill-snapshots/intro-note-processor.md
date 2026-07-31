---
name: intro-note-processor
description: >-
  Process a Notion meeting note for follow-up intro commitments. Scans the note body for explicit ("intro X to
  [Founder]") and implicit intro signals (Tom offering a coinvestor, customer, downstream investor, or advisor).
  For each, resolves the person in the People DB, dedups against all four intro lifecycle fields on the linked
  Opp, appends new candidates to Intros (Qualified), and saves a Gmail draft of the outreach using historical
  same-type sends as the voice template. Missing People DB entries surface as a Slack alert (no auto-stub).
  Modes: (B) Subroutine — called by meeting-note-processor at the end of Mode B-process; (C) Manual — "process
  my call with [person] for intros", "process the [company] note for intros", "extract intros from [note]",
  "draft intro outreach from this call".
---

# Intro Note Processor

After Tom takes a call (Notion AI auto-creates a meeting note in the Notes DB, or Tom records a transcript), this skill reviews the note for follow-up intros Tom committed to making, stages them on the linked Opportunity, and saves Gmail drafts of the outreach emails so Tom can review and send.

This is the bridge between "Tom said on a call: I'll intro you to Lauren" and "the outreach email is sitting in Tom's drafts folder." It does NOT send. It does NOT formally intro (that's `intro-draft-agent`'s double-opt-in). It only **stages Qualified + drafts the initial outreach to the target**.

---

**Canonical lifecycle rules:** `shared-references/intro-lifecycle-contract.md` — on any conflict, the contract wins. The inline gates/rules in this file remain in force as defense-in-depth.

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

Intent: run two parallel passes over `note_body` — explicit string-scan (2a) + implicit LLM classification (2b) — merge them, drop unnamed/generic entries, and exit silently if none remain. Full procedure in `references/step-2-candidate-identification.md` — read it now before proceeding.

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
5. **Subject-agnostic sent-check (MANDATORY).** Query B only catches canonical outreach subjects — Tom often intros manually with free-form subjects ("Re-intro'ing! …", "X <> Y", inline replies). For each candidate with a resolved email, also run `in:sent (to:<email> OR cc:<email>) newer_than:90d` and inspect hits for this Opp (founder/`Contact` email as co-recipient, or Opp corroboration in subject/body). Dual-recipient hit → intro already Made: drop with `intro-already-sent`, skip both the Qualified write and the draft. Single-recipient hit about this Opp → treat as `outreach-sent-but-untracked` per point 4. Full rule: `shared-references/intro-lifecycle-contract.md`, Pre-Draft Sent-Check section.

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

Intent: for each newly-Qualified person, save ONE Gmail draft to the target only (recipient resolution 7a, universal `<opp_name> – would love to intro` subject 7b, voice templates 7c, body composition 7d, bucket-specific framing 7e, save 7f, and the 7g draft/sent idempotency check that skips duplicates). Full procedure in `references/step-7-outreach-draft.md` — read it now before proceeding.

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
  - `⏭️ intro already sent — no action` — subject-agnostic sent-check found Tom's own send (dual-recipient = Made); Made relation should be updated by the scanner or manually
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

Intent: per-branch handling for portfolio Opps, repeat mentions, company-less targets, same-person-across-Opps, the founder-of-the-Opp filter, and empty `opp_page_id`. Plus the rationale for the universal `– would love to intro` subject line. Full detail in `references/edge-cases-and-rationale.md` — read it now before proceeding when any of these branches applies.

---

## Integration with meeting-note-processor

This skill is invoked as a subroutine at the end of `meeting-note-processor` Mode B-process. The caller passes `note_page_id`, `note_body`, `opp_page_id`, `opp_name`, `note_title`, `note_url` so this skill skips its own fetches.

The integration point is documented in `meeting-note-processor/SKILL.md` (Step 8 — Intro extraction). Failures here are non-fatal to the caller.
