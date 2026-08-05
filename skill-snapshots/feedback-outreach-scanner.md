---
name: feedback-outreach-scanner
description: >
  Scheduled sweep + per-event webhook handlers for feedback outreach. Scans the Notion 📣 Pending Feedback relation as source of truth for feedback outreach contacts, then searches Gmail sent mail and inbox for outreach activity and replies. The sweep mode runs as a sub-agent of the Diligence Agent and reconciles what the per-event webhooks missed; webhook Mode B has two sub-modes — B-inbound (feedback-reply-detect, one inbound reply at a time) and B-outbound (feedback-outreach-sent-detect, one outbound feedback ask at a time), both via the claude-job-queue primitive. Creates per-person Notion feedback notes on the outbound side, appends replies on the inbound side, and removes people from Pending Feedback once substantive feedback is logged. Uses email-first matching with name-based fallback to avoid missing emails sent to alternate addresses. NOT the feedback-outreach-drafter skill (which drafts outreach emails) — this skill runs after emails are sent, not before.
---

# Feedback Outreach Scanner

Scans Gmail for feedback outreach activity over the past 12 hours, using the Notion `📣 Pending Feedback` relation as the source of truth for who to look for — NOT a hardcoded subject line pattern.

1. **Sent scan** — detects newly sent feedback outreach emails and creates per-person Notion feedback notes
2. **Reply scan** — detects replies from feedback contacts, appends feedback to existing notes, and removes the person from `📣 Pending Feedback` only when substantive feedback has been received
3. **Manual reconciliation** (sweep only, Step 2c) — detects feedback Tom entered into a note by hand (phone-call notes, pasted text threads), then drops `[PENDING]` and clears the relation, since no Gmail or transcript event ever fires for those

## Key IDs

- **Opportunities DB:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
- **Feedback Scanner View URL:** `https://www.notion.so/tomseo/10e00beff4aa80ac8edadd62469d6b63?v=b646ee99c012402b9bb2fa70cc738daf` (custom view dedicated to feedback outreach — verified 2026-04-17 to return populated `📣 Pending Feedback` relations). Do NOT use `?v=31400beff4aa80fdb2e0000c1b6ae673` — that is the general pipeline Agent View, not the feedback view, and will return opportunities with no pending feedback.
- **People DB:** `1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`
- **Notes DB:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`

---

## Mode B: Single-Message (Webhook)

Mode B has two sub-modes selected by the `mode` arg the queue payload carries:

- **B-inbound** (`mode` absent, OR explicitly `'reply'`) — invoked by `gmail-webhook/Code.js` (`feedback-reply-detect` handler) on a single inbound reply.
- **B-outbound** (`mode === 'sent'`) — invoked by `gmail-webhook/Code.js` (`feedback-outreach-sent-detect` handler) on a single outbound feedback ask Tom just sent.

In both sub-modes the queue payload includes `personId` and `oppCandidateIds` (≥ 1) pre-resolved by the webhook gate — no Step 0 roster scan is required.

### Mode B-inbound (single inbound reply)

**Args:**
- `messageId` (required) — Gmail message ID of the inbound reply
- `threadId` (required) — Gmail thread ID for thread context + outreach lookup
- `senderEmail` (required) — pre-parsed `From` address
- `personId` (required) — Notion People DB page ID, pre-resolved by webhook gate
- `oppCandidateIds` (required) — array of Opportunity page IDs where this person sits in `📣 Pending Feedback`. Webhook gate guarantees length ≥ 1.

**Behavior:**
- **Skip Step 0** — the webhook already built the list-of-one (this person, these candidate opps).
- **Skip Step 1** — no need to scan sent mail; this is an inbox reply event.
- Run Step 2's per-message logic on this single message:
  1. If `oppCandidateIds.length > 1`, disambiguate by reading the thread's earliest sent message and haystack-matching its subject + body-head against the candidate opp names. If still ambiguous, log `webhook-mode-ambiguous-opp` and exit 0.
  2. Apply Step 2's "Processing matches" sub-steps (read full thread, sanity-check it's a feedback outreach thread, classify the reply) exactly as written. The classification rubric (substantive vs acknowledgment/deferral) lives in Step 2 — do not restate it here.
- Run Steps 3-5 (note creation OR append + remove from Pending Feedback if substantive) exactly as in scheduled mode.
- **Skip Step 6's scheduled-scan summary.** Emit the single-line run-log entry: `single-message feedback-reply: <person> on <opp> — <classification> → <action>`. Then, **only if the reply classified as substantive**, post the **Reply-logged confirmation** (below) via `send-alert` to `#claude-alerts`. This is the standing behavior Tom asked for 2026-08-01 — substantive inbound feedback written to Notion is confirmed immediately, not just rolled up in the next Diligence Agent daily digest (the digest still lists it too). Fire the confirmation only AFTER the Notion write succeeds, so it is a genuine "it's logged" signal; if the write failed, alert the failure instead.
- **Deferrals get NO Slack alert** (Tom confirmed 2026-08-01 — substantive only). A "will revert EOW / traveling / circle back" reply is logged to the note as usual but stays silent on Slack; it still surfaces in the daily digest. Only the substantive classification pings.

**Reply-logged confirmation.** Header MUST start with `💬` (not a `claude-alerts-listener` reserved emoji — no listener action is wanted here, it is FYI-only). Send via `send-alert`:

> 💬 Feedback logged — **[Opp Company]**: [Person Name] ([Person Company]) on [subject — founder name or company gut-take]
> [1–2 sentence excerpt of the substantive reply verbatim]
> Note: [Notion feedback-note URL] · Opp: [Notion Opp URL]

Substantive replies only — the note URL points at the feedback note the reply was written into (created-from-scratch or appended), so Tom is one click from the logged record.

Idempotency at queue layer (key = `feedback-reply-{messageId}`); skill does not re-check.

### Mode B-outbound (single outbound feedback ask)

**Args:**
- `mode` (required) — literal string `'sent'`
- `messageId` (required) — Gmail message ID of Tom's outbound feedback ask
- `threadId` (required) — Gmail thread ID (almost always a fresh thread, but pass through anyway)
- `recipientEmail` (required) — the resolved recipient's email address
- `recipientName` (optional) — display name parsed from the `To`/`Cc` header; present mainly on the founder-backchannel path, where it seeds contact creation
- `personId` (required **unless** `needsPersonCreate` is true) — Notion People DB page ID, pre-resolved by webhook gate
- `needsPersonCreate` (optional, default false) — see "Founder-backchannel path" below
- `oppCandidateIds` (required) — array of Opportunity page IDs where this person sits in `📣 Pending Feedback`. Webhook gate guarantees length ≥ 1. The webhook ALSO ran an LLM classifier before enqueueing, so the body is already confirmed — do not re-classify intent.

#### Founder-backchannel path (`needsPersonCreate: true`)

Added 2026-07-31. Set by `feedback-founder-backchannel.js` when Tom writes an ad-hoc backchannel
asking about a FOUNDER (subject is the founder's name — `Re: Avery Alchek`, `Nipun Jasuja
reference`) rather than a company gut-take. Two things differ from the normal outbound path:

1. **`personId` is null** — the recipient has no People DB row, so the webhook could not write the
   `📣 Pending Feedback` relation (a relation needs a page id). The webhook deliberately does NOT
   mint the row itself; creation happens here, where the enrichment tooling lives.

   **✅ AUTO-CREATION IS AUTHORIZED ON THIS PATH — do not stop and ask Tom.** This is a deliberate
   carve-out from the usual "surface a new contact to Tom" norm (cf. `neg1-promote`, which never
   creates People rows). The justification is that Tom *explicitly and deliberately* emailed this
   person asking for feedback, so the relationship is already established — there is no judgement
   call left for him to make. Confirmed by Tom 2026-07-31. **Post a Slack heads-up after creating**
   (see below); the notification replaces the approval gate, it does not precede it.

   **Resolution chain.** The email is the one thing always known, and since the 2026-07-31 MCP fix
   it is a *strong* key:
   1. `contactout_email_to_linkedin(recipientEmail)` → LinkedIn URL.
   2. `contactout_enrich_linkedin_profile(url)` → full profile (headline, location, seniority,
      complete experience history).
   3. If step 1 finds nothing, `contactout_enrich_person(email=…)` returns Name / Company /
      LinkedIn / Location and is enough for a usable row.
   4. For *current* employer trust the profile's `is_current` experience entry over cached values.
   5. Populate Name, Email, LI, Company, Role, Category, City, State per the field rules in
      `add-to-contacts/SKILL.md`. **Never skip the lookup for speed** — that hard rule still binds;
      what changed is only whether Tom is asked first, not whether enrichment runs.
   5. If every lookup comes back empty, still create the row with the name + email you have, and say
      so explicitly in the Slack message so Tom knows it needs manual enrichment.

   Then add the new page to the Opp's `📣 Pending Feedback` relation — read the existing array and
   write the FULL merged array back, since relation writes are a full replace.

   **Slack heads-up (required whenever a row was created).** Send via `send-alert` to
   `#claude-alerts`:

   > 👤 Created People DB entry: **[Name]** ([email])
   > Reason: feedback outreach on **[Company]** — not previously in People DB
   > Enriched: [LinkedIn URL | "⚠️ no ContactOut match — needs manual enrichment"]
   > [Company] · [Role] · [City]
   > → [Notion People page URL] · [Opportunity URL]
   > ↩️ Reply with their LinkedIn URL and I'll enrich the row.

   The header MUST start with `👤 Created People DB entry` — `claude-alerts-listener` special
   branch 8 keys on that exact string to route a LinkedIn-URL reply back into enrichment, and the
   People page URL is how it locates the row. Include the `↩️` line whenever enrichment was empty.

2. **`oppCandidateIds` has exactly one entry**, resolved deterministically off the founder's name
   (People row → Contact email → Active Opp, or a `-1 (Founder Name)` title). No disambiguation
   needed — but do confirm the resolved Opp is still `Active` before writing.

When `needsPersonCreate` is false the recipient was already promoted onto `📣 Pending Feedback` by
the webhook; behave exactly as the normal outbound path.

**Behavior:**
- **Skip Step 0** — the webhook already resolved (person, candidate opps).
- **Skip Step 2 entirely** — this is an outbound event, no inbound reply scan needed.
- Run Step 1's "Processing matches" sub-steps on this single sent message:
  1. If `oppCandidateIds.length > 1`, disambiguate by reading the sent message's subject + body-head and haystack-matching against the candidate opp names. If still ambiguous, log `webhook-mode-ambiguous-opp` and exit 0.
  2. Read the full sent message to extract subject, recipient name/email, send date, and the **complete verbatim body**.
  3. Sanity check: skim the body to confirm it reads as a feedback ask (mentions the opp company name, asks for input/take/perspective, or contains diligence questions). The LLM gate is the primary trust signal, but a second pass here is cheap insurance against misclassification.
  4. Deduplication check: fetch the Opp page and inspect its `✍️ Notes` relation array for a note titled with this person — giver-first `[First Name] [Last Name] ([Their Company]): [Subject]` or legacy `[Company]: [First Name] [Last Name]`, with or without the `[PENDING]` prefix. If a match exists, exit 0 — the scheduled sweep or a prior webhook run already logged it.
- Run Step 3 (create the per-person `[PENDING]` note) exactly as in scheduled mode, using the sent-message body as the outreach note body and `## Response — [No reply yet]` placeholder.
- Set the new note's `Opportunity` relation to the resolved Opp page URL on creation — Notion bidirectional sync then auto-links it into the Opp's `✍️ Notes` array.
- **Do NOT remove the person from `📣 Pending Feedback`.** Outbound = note created; PF removal happens only when substantive feedback arrives (Step 5, triggered on the inbound reply later).
- **Skip Step 6's scheduled-scan summary.** Emit a single-line run-log entry: `single-message feedback-outreach-sent: <person> on <opp> — note created`. No standalone Slack alert.

Idempotency at queue layer (key = `feedback-outreach-sent-{threadId}-{personId|recipientEmail}`);
skill does not re-check. Keyed on **threadId**, not messageId — Gmail fires multiple webhooks for the
same outreach thread, and messageId-keying once produced duplicate `[PENDING]` notes (Gilad Rom /
Factir, 2026-05-12). Falls back to the lowercased recipient email when `personId` is null.

## Step 0: Build the Pending Feedback Contact List from Notion

This is the foundation of the scanner. Instead of searching Gmail for a specific subject pattern, we resolve the universe of people we're looking for from Notion.

Use `notion-query-database-view` with the **Feedback Scanner View** at `https://www.notion.so/tomseo/10e00beff4aa80ac8edadd62469d6b63?v=b646ee99c012402b9bb2fa70cc738daf`. This custom view surfaces opportunities associated with feedback outreach. For each result, inspect the `📣 Pending Feedback` relation field — if the key is absent from the JSON or the array is empty, skip (feedback is already resolved or out of scope). If populated, proceed.

Do NOT use the general pipeline Agent View (`?v=31400beff4aa80fdb2e0000c1b6ae673`) for this step — past runs did and returned 9 opportunities with 0 populated Pending Feedback relations.

For each opportunity returned:
1. Fetch the `📣 Pending Feedback` relation array — this contains People DB page URLs.
2. For each person in the array, fetch their People DB page to get:
   - **Full name** (first + last)
   - **Email** (may be empty or may not match the address Tom actually emailed)
   - **People DB page URL** (for mention links in notes)
   - **LI** field (LinkedIn URL)
   - **Company** and **Role** fields
3. Build a lookup list: `[ { name, email, people_page_url, li_url, company, role, opportunity_name, opportunity_page_url } ]`

> **Critical — no hallucination**: Use only what is actually returned by the People DB fetch. Do NOT infer, guess, or fill in company names, titles, or background from prior context or external knowledge. If a field is empty in the People DB, leave it blank in the note. A blank field is far easier to fix than incorrect data.

This list drives both the sent scan and the reply scan. Because the field tracks only people whose feedback is still outstanding, the list is naturally small — people are removed once their feedback is logged (Step 5).

---

## Step 1: Scan Sent Mail for Newly Sent Feedback Outreach Emails

For each person in the contact list from Step 0, search Gmail sent mail for outreach sent in the past 12 hours.

### Email-first search

If the person has an email on file in the People DB, try:
```
q: "in:sent to:<email> newer_than:12h"
```

### Name-based fallback

If the email search returns no results — OR if the person has no email on file — fall back to a name-based search:
```
q: "in:sent to:\"<First Name> <Last Name>\" newer_than:12h"
```

This catches cases where Tom emailed a personal address that differs from the one in the People DB, or where the People DB email field is blank.

### Processing matches

For each sent email found:
1. Read the full message to extract: subject, recipient name and email, send date, and the **complete verbatim email body** — every line, including opener, questions, and company blurb. You will paste this directly into the note in Step 3.
2. **Sanity check**: Skim the email body to confirm it's a feedback outreach (contains diligence questions, a company blurb, or references the opportunity). Skip unrelated emails to the same person (e.g., a scheduling email).
3. **Deduplication check**: Fetch the opportunity page and inspect its `✍️ Notes` relation array. For each note URL in that array, fetch the note page and check its title (with or without the `[PENDING]` prefix) against both the current giver-first form and the legacy company-first form:
   - `[First Name] [Last Name] ([Their Company]): [Subject]` (current)
   - `[Company]: [First Name] [Last Name]` (legacy, pre-2026-08-03)
   If a title match exists, skip — do not create a duplicate. This approach reads from the Opportunity page itself (Step 3 now writes the `Opportunity` relation atomically inside the create call, so the array is authoritative the instant a note exists) rather than relying on Notion's search index, which may not reflect pages created in the same or immediately prior scanner run.

   **Also check for orphans.** A note left unlinked by a pre-atomic-era run (or a dropped relation write) is absent from the array and will not be caught above, producing a duplicate. Additionally run the Step 3b **orphan sweep** (`notion-search` over the Notes DB, `workspace_search`, matching title + empty `Opportunity` relation) before deciding to create. If an orphan match exists, do not create a second note — repair the orphan instead by writing its missing `Opportunity` relation, then proceed as if it were found normally.
4. If no matching note exists in the relation, proceed to create one (Step 3).

---

## Step 2: Scan Inbox for Replies from Feedback Contacts

For each person in the contact list from Step 0, search Gmail inbox for replies received in the past 12 hours.

### Email-first search

If the person has an email on file in the People DB, try:
```
q: "is:inbox from:<email> newer_than:12h"
```

### Name-based fallback

If the email search returns no results — OR if the person has no email on file — fall back to:
```
q: "is:inbox from:\"<First Name> <Last Name>\" newer_than:12h"
```

### Processing matches

For each inbox message found:
1. Read the full thread to extract: sender name and email, reply date, reply body (strip quoted prior messages — extract only the new reply text). **The reply body must be pasted verbatim into the note — do not summarize, paraphrase, or rewrite into third person.**
2. **Sanity check**: Confirm the reply is part of a feedback outreach thread (the thread should contain a prior outreach message from Tom about the relevant opportunity). Skip unrelated emails from the same person.
3. **Classify the reply** — this determines downstream actions:
   - **Substantive feedback**: The person shares actual opinions, market reactions, answers to diligence questions, or relevant observations about the opportunity. This counts as feedback received.
   - **Acknowledgment / deferral**: The person says they'll respond later ("I'll send notes soon", "give me a few days", "will get back to you after vacation"). This does NOT count as feedback received — the person remains pending, even though they replied.
4. Search for an existing note for this person + opportunity by fetching the opportunity page and inspecting its `✍️ Notes` relation array. Check each linked note's title (prefixed or unprefixed) against both the giver-first `[First Name] [Last Name] ([Their Company]): ...` and legacy `[Company]: [First Name]...` forms. This is the same relation-based check used in Step 1 — do not use Notion search here.
5. If a note exists — append the reply under `## Response — [Date]` (Step 4).
6. If no note exists yet — create the full note now using the thread's sent message as the outreach note body and the reply as the response (Step 3 + Step 4 together).
7. **After appending the reply**, act based on classification:
   - **Substantive feedback**: remove `[PENDING]` from the note title (Step 4b) and remove this person from `📣 Pending Feedback` (Step 5).
   - **Acknowledgment / deferral**: keep `[PENDING]` in the title and keep the person in `📣 Pending Feedback`. Log the acknowledgment in the note so there's a record, but treat them as still outstanding.

---

## Step 2c: Reconcile Manually-Entered Feedback (scheduled sweep only)

Gmail replies (Step 2) and Zoom transcripts (meeting-note-processor Step 4b) have automated paths, but feedback sometimes arrives OUTSIDE both — a phone call whose notes Tom types directly into the Notion note, or a text thread he pastes in by hand. Nothing fires on a manual Notion edit, so the note stays `[PENDING]` and the person stays in `📣 Pending Feedback` forever. This step closes that gap. (Origin: Paul Drinkwater / Fair, 2026-08-03 — phone-call notes manually added, stale `[PENDING]` + stale relation both needed hand-cleanup.)

Run this in **scheduled sweep mode only** (skip in webhook Modes B-inbound/B-outbound — they're single-event handlers).

For each person still in the Step 0 contact list AFTER Steps 1–2 ran (i.e., no inbound reply resolved them this sweep):
1. Find their existing note via the Opp's `✍️ Notes` relation (same title-matching as Step 2.4). No note → nothing to reconcile; skip.
2. If the note's title still carries `[PENDING]`, fetch the note body and check for substantive content the scanner did NOT write:
   - A populated `## Call Notes — [Date]` section (non-empty body under the heading)
   - A populated `## Response — [Date]` section whose content the sweep/webhook didn't append (manual paste)
   - Any other dated section with actual feedback content (raw text thread, voice-note transcription, etc.)
3. Classify that content with the same substantive-vs-deferral standard as Step 2.3. Scheduling chatter, empty placeholder headings, or "will call you Monday" content is NOT substantive — leave pending.
4. If substantive: run Step 4b (drop `[PENDING]`) and Step 5 (remove from `📣 Pending Feedback`). Do NOT rewrite, reformat, or summarize the manual content — Tom's notes stay exactly as he typed them.
5. Count these in the Step 6 summary under **Manually-resolved (reconciled)**.

Inverse-drift guard: if a note's title has NO `[PENDING]` prefix but the person is still in `📣 Pending Feedback` (e.g., Tom dropped the prefix by hand), treat the unprefixed title as the substantive signal — verify the body isn't empty placeholders, then run Step 5 to clear the relation.

---

## Step 3: Create Per-Person Feedback Note

For each newly sent feedback outreach email (from Step 1) or unannotated thread (from Step 2), create a Notes DB page.

Use `notion-create-pages` with `data_source_id: e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`.

### Resolve person context

Use only the People DB data fetched in Step 0 — do not look up or infer information from external sources, prior conversation context, or general knowledge:
- People DB page URL (for `<mention-page>`)
- `LI` field (LinkedIn URL) — include only if present in the DB; omit entirely if blank
- `Company` and `Role` fields — use exactly as they appear in the DB; if blank, leave blank
- Background sentence: write only what can be grounded in the DB fields; if there's insufficient data, omit the background rather than fabricating it

### Page title + content structure

**Canonical contract: `~/.claude/skills/shared-references/feedback-note-format.md`. Read it and follow it exactly** — title (giver-first, `[PENDING]` prefix, Feedback-vs-Reference subject variants), body layout (`## Response — [No reply yet]` before `## Outreach Note — [Date sent]`), People-DB grounding rules, and the no-page-icon rule all live there. Do not restate them here; that duplication is what caused the 2026-08-04 drift with `feedback-outreach-drafter`.

This skill owns the runtime **lifecycle** around that format: creation timing (Step 3), the race guard (Step 3b), reply append + `[PENDING]` removal (Step 4/4b), and `📣 Pending Feedback` clearing (Step 5).

If a reply already exists at note-creation time (Step 2 case), populate the Response section immediately with the **complete verbatim reply text** (same raw-text rule as the outreach note — do not summarize or rewrite) and apply the classification logic to determine whether to keep `[PENDING]` in the title.

### Link to opportunity — set the relation ATOMICALLY in the create call

**Set the `Opportunity` relation inside the `notion-create-pages` properties object, in the SAME call that creates the page.** It is a synced relation pair: writing `Opportunity` on the note automatically populates the Opp's `✍️ Notes`. One call, no window in which an unlinked note can exist.

```
properties: {
  "Name": "[PENDING] ...",
  "Category": "Diligence",
  "Opportunity": ["https://app.notion.com/p/<opp-page-id>"]
}
```

Resolve the Opp page id BEFORE creating: in webhook modes use the `oppCandidateIds` arg (the gate guarantees ≥ 1 — never re-search); in sweep/manual mode resolve it in Step 0 alongside the contact list. **If the Opp id cannot be resolved, do NOT create the note** — abort and alert. A note with no Opp is worse than no note (see the orphan trap below).

> **Never use the old two-step pattern** (create the page, then separately fetch + rewrite the Opp's `✍️ Notes` array). That second write is a separate failure point, and when it fails the note is **orphaned** — and an orphan is invisible to BOTH the dedup check (Step 1.3 / Mode B Step 4) and the Step 3b race guard, because both read the Opp's `✍️ Notes`. The guard designed to clean up duplicates cannot see the exact object it needs to clean up. **Canonical incident 2026-08-04:** the B-outbound handler created `[PENDING] John Hor (Box): Avery Alchek Reference` at 16:44:20 with no `Opportunity` relation, one minute after a manually-created note for the same person. Dedup didn't see the manual note's peer; Step 3b didn't see its own orphan; the duplicate survived until Tom spotted it in the UI.

### Verify the link before exiting (fail loud, never silent)

Immediately after the create call:
1. Re-fetch the new note page and read its `Opportunity` relation.
2. If it is non-empty → proceed to Step 3b.
3. If it is **empty**, the atomic write silently dropped the relation. Retry once with `notion-update-page` (`command: update_properties`, `Opportunity: ["<opp-url>"]`), then re-fetch again.
4. If it is STILL empty after the retry, **do not exit 0**. Emit run-log `feedback-note-orphaned: <person> on <opp> — note <url> has no Opportunity relation` and post a `⚠️` alert via `send-alert` naming the note URL, so the orphan is surfaced to Tom the same run rather than discovered weeks later. An unlinked note must never be left silently in the DB.

---

## Step 3b: Post-create reconciliation (race-condition guard)

**Always run this immediately after Step 3's link-to-opportunity write, regardless of mode (sweep, webhook B-outbound, manual).**

The pre-write dedup check at Step 1.3 / Mode B Step 4 reads the Opp's `✍️ Notes` array — but two concurrent jobs (e.g. webhook + sweep, or two webhook events from the same thread before the queue-layer dedup catches up) both pass that check because neither has written yet, then both write. Observed in production 2026-05-12 (Gilad Rom / Factir, two `[PENDING]` notes created ~1s apart).

Procedure:
1. Re-fetch the Opp page's `✍️ Notes` array.
2. For each note URL in the array, fetch and inspect the title. Build the list of notes whose title matches this person + subject in any recognized form — giver-first `[First Name] [Last Name] ([Their Company]): [Subject]` or legacy `[Company]: [First Name] [Last Name]`, prefixed or unprefixed — for the SAME `(Company, First Name, Last Name)` triple you just wrote.
2b. **Orphan sweep — the Opp array is not sufficient on its own.** A peer note whose `Opportunity` relation failed to write does NOT appear in the array, so steps 1–2 are blind to exactly the object this guard exists to remove. Additionally run `notion-search` against the Notes DB (`data_source_url: collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`, `content_search_mode: "workspace_search"`) for the note title, and add to the match list any result that (a) matches the same person + subject in any recognized form, prefixed or unprefixed, and (b) has an **empty `Opportunity` relation**. Orphans count as peers for the tiebreaker below, and archiving them needs no `✍️ Notes` removal — they were never in the array.
3. If exactly one match exists (your own write), done — exit normally.
4. If two or more matches exist, you raced a peer. Apply the deterministic tiebreaker: **keep the note with the EARLIEST `Created` timestamp; archive every other matching note and remove its URL from the Opp's `✍️ Notes` array.** Earliest-wins means both racers converge on the same survivor without coordination.
5. Archive via the Notion REST API: `PATCH /v1/pages/{pageId}` with `{"archived": true}`. The token decryption pattern is the same one `~/.claude/scripts/network_sync_notion.py` uses (`cd ~/code/notion-backup && SOPS_AGE_KEY_FILE=... python3 -c "from export import get_token; ..."`). Do NOT skip this step — leaving the duplicate as a live `[PENDING]` note pollutes the Opp's Notes feed.
6. Re-fetch the Opp's `✍️ Notes` array, remove the archived note URLs, and write it back.
7. Emit a run-log entry: `post-create reconciliation: archived N peer note(s) on <opp> for <person> (kept <survivor-url>)`.

This guard makes Step 3 self-healing under concurrent writes. The gmail-webhook layer's `(threadId, personId)` idempotency key prevents the common case (two webhook events for the same outreach thread); Step 3b is the safety net for webhook+sweep overlap and any other path that races.

---

## Cross-skill: meeting-note-processor handles the email-then-call case

If Tom hops on a Zoom with a feedback giver AFTER the `[PENDING]` stub has been created (call transcript becomes the substantive feedback instead of an email reply), `meeting-note-processor` Step 4b detects the stub on the linked Opp, ports the outreach context into the meeting note, archives the stub, renames the meeting note to this skill's giver-first `[Person] ([Their Company]): [Company] Feedback` convention, and removes the person from `📣 Pending Feedback`. No action needed here — the scanner's prefix-based dedup at Step 1.3 / Step 2.4 will recognize the renamed meeting note on any subsequent inbound reply and append rather than re-create.

---

## Step 4: Append Reply to Existing Note

When a reply is detected (Step 2) and a note already exists:

1. Fetch the existing note page to confirm current content.
2. Use `notion-update-page` with `command: update_content` to insert the response.
3. If the `## Response — [No reply yet]` section is blank, replace it with the **complete verbatim reply text** under `## Response — [Date]`. Do not summarize, paraphrase, or rewrite the reply into third person — paste the raw email text exactly as received (stripping only quoted prior messages).
4. If a response already exists (prior reply), insert a new `## Response — [Date]` block **ABOVE** the existing one — do not overwrite, and do not append below. The page is **reverse-chronological: newest input on top**, so the oldest event (`## Outreach Note`) always sits last (Tom, 2026-08-04). Same verbatim rule applies. If the new block shares a date with an existing one, disambiguate both with a parenthetical channel tag — `## Response — August 4, 2026 (reference call)` above `## Response — August 4, 2026 (email)`.
5. Also update the respondent's email in the People DB if the reply came from a different address than what is on file.

### Step 4b: Remove [PENDING] prefix when substantive feedback arrives

If the reply is classified as **substantive feedback**, update the note title to remove the `[PENDING]` prefix using `notion-update-page` with `command: update_properties`:

- Before: `[PENDING] Jeff Green (Hatch Bank): Clusia Feedback`
- After: `Jeff Green (Hatch Bank): Clusia Feedback`

Do NOT remove the prefix for acknowledgments or deferrals — those people are still pending.

---

## Step 5: Remove Person from Pending Feedback

After **substantive feedback** is successfully logged, remove the person from the opportunity's `📣 Pending Feedback` relation.

Do NOT remove the person if the reply was an acknowledgment or deferral — they stay in `📣 Pending Feedback` so the scanner picks them up again on the next run.

**Procedure:**
1. Fetch the current `📣 Pending Feedback` array from the opportunity page.
2. Remove the person's People DB page URL from the array.
3. Write the updated array back using `notion-update-page` as a JSON array string in a single call.

---

## Step 6: Return Summary

Return a structured summary for the Diligence Agent orchestrator:

```
### Feedback Outreach Scanner
- **New notes created**: [count] — [list: Person (Company) → Opportunity]
- **Replies logged**: [count] — [list: Person → Opportunity, note whether substantive or deferral]
- **Manually-resolved (reconciled)**: [count] — [list: Person → Opportunity, source (call notes / manual paste)]
- **Removed from Pending Feedback**: [count] — [list: Person → Opportunity]
- **Still pending (deferral/ack only)**: [count] — [list: Person → Opportunity]
- **Already existed / skipped**: [count]
- **Name-fallback matches**: [count] — [list: Person → email used (if different from People DB)]
- **Errors**: [any issues]
```

Do NOT send any Signal/Beeper notifications — the Diligence Agent handles all alerts centrally.
