---
name: feedback-outreach-scanner
description: >
  Scans the Notion 📣 Pending Feedback relation as source of truth for feedback outreach contacts, then searches Gmail sent mail and inbox for outreach activity and replies. Creates per-person Notion feedback notes, appends replies, and removes people from Pending Feedback once their feedback is logged. Uses email-first matching with name-based fallback to avoid missing emails sent to alternate addresses. This is a sub-agent of the Diligence Agent — it does NOT run standalone. It is NOT the feedback-outreach-drafter skill (which drafts outreach emails). This skill runs after emails are sent, not before.
---

# Feedback Outreach Scanner

Scans Gmail for feedback outreach activity over the past 12 hours, using the Notion `📣 Pending Feedback` relation as the source of truth for who to look for — NOT a hardcoded subject line pattern.

1. **Sent scan** — detects newly sent feedback outreach emails and creates per-person Notion feedback notes
2. **Reply scan** — detects replies from feedback contacts, appends feedback to existing notes, and removes the person from `📣 Pending Feedback` only when substantive feedback has been received

## Key IDs

- **Opportunities DB:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
- **Feedback Scanner View URL:** `https://www.notion.so/tomseo/10e00beff4aa80ac8edadd62469d6b63?v=b646ee99c012402b9bb2fa70cc738daf` (custom view dedicated to feedback outreach — verified 2026-04-17 to return populated `📣 Pending Feedback` relations). Do NOT use `?v=31400beff4aa80fdb2e0000c1b6ae673` — that is the general pipeline Agent View, not the feedback view, and will return opportunities with no pending feedback.
- **People DB:** `1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`
- **Notes DB:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`

---

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
3. **Deduplication check**: Fetch the opportunity page and inspect its `✍️ Notes` relation array. For each note URL in that array, fetch the note page and check its title against both prefixed and unprefixed forms:
   - `[PENDING] [Company] — [First Name] [Last Name]`
   - `[Company] — [First Name] [Last Name]`
   If a title match exists, skip — do not create a duplicate. This approach reads from the Opportunity page itself (updated atomically at note-creation time) rather than relying on Notion's search index, which may not reflect pages created in the same or immediately prior scanner run.
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
4. Search for an existing note for this person + opportunity by fetching the opportunity page and inspecting its `✍️ Notes` relation array. Check each linked note's title against both `[PENDING] [Company] — [First Name]...` and `[Company] — [First Name]...` forms. This is the same relation-based check used in Step 1 — do not use Notion search here.
5. If a note exists — append the reply under `## Response — [Date]` (Step 4).
6. If no note exists yet — create the full note now using the thread's sent message as the outreach note body and the reply as the response (Step 3 + Step 4 together).
7. **After appending the reply**, act based on classification:
   - **Substantive feedback**: remove `[PENDING]` from the note title (Step 4b) and remove this person from `📣 Pending Feedback` (Step 5).
   - **Acknowledgment / deferral**: keep `[PENDING]` in the title and keep the person in `📣 Pending Feedback`. Log the acknowledgment in the note so there's a record, but treat them as still outstanding.

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

### Page title

All new notes start as `[PENDING]` — feedback has not been received yet:

```
[PENDING] [Company] — [First Name] [Last Name] ([Current Company]) Feedback
```

Example: `[PENDING] Clusia — Jeff Green (Hatch Bank) Feedback`

The `[PENDING]` prefix is removed only when substantive feedback arrives (Step 4b).

### Page content structure

```
<mention-page url="[Notion People DB URL]"/> ([LinkedIn](LI URL)) — [Role], [Current Company]. [1-2 sentence background, grounded in People DB only — omit if data is insufficient].

---

## Response — [No reply yet]

---

## Outreach Note — [Date sent]

[Complete verbatim body of the outreach email copied from Gmail — opener, questions, company blurb, every line. Do not summarize or use a placeholder.]
```

If a reply already exists at note-creation time (Step 2 case), populate the Response section immediately with the **complete verbatim reply text** (same raw-text rule as the outreach note — do not summarize or rewrite) and apply the classification logic to determine whether to keep `[PENDING]` in the title.

### Page icon

Do NOT set a Claude icon (or any icon) on feedback response notes. The content of these pages is raw third-party text — Tom pastes verbatim replies and the outreach body is copied directly from Gmail. Setting the Claude icon would falsely imply Claude authored the content. Leave the page icon blank.

### Link to opportunity

After creating the note:
1. Search the Opportunities DB for the company name to get the opportunity page.
2. Fetch the current `✍️ Notes` array from the opportunity page.
3. Append the new note's URL and write the full array back as a JSON array string in a single call.

---

## Step 4: Append Reply to Existing Note

When a reply is detected (Step 2) and a note already exists:

1. Fetch the existing note page to confirm current content.
2. Use `notion-update-page` with `command: update_content` to insert the response.
3. If the `## Response — [No reply yet]` section is blank, replace it with the **complete verbatim reply text** under `## Response — [Date]`. Do not summarize, paraphrase, or rewrite the reply into third person — paste the raw email text exactly as received (stripping only quoted prior messages).
4. If a response already exists (prior reply), append a new `## Response — [Date]` block after the existing one — do not overwrite. Same verbatim rule applies.
5. Also update the respondent's email in the People DB if the reply came from a different address than what is on file.

### Step 4b: Remove [PENDING] prefix when substantive feedback arrives

If the reply is classified as **substantive feedback**, update the note title to remove the `[PENDING]` prefix using `notion-update-page` with `command: update_properties`:

- Before: `[PENDING] Clusia — Jeff Green (Hatch Bank) Feedback`
- After: `Clusia — Jeff Green (Hatch Bank) Feedback`

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
- **Removed from Pending Feedback**: [count] — [list: Person → Opportunity]
- **Still pending (deferral/ack only)**: [count] — [list: Person → Opportunity]
- **Already existed / skipped**: [count]
- **Name-fallback matches**: [count] — [list: Person → email used (if different from People DB)]
- **Errors**: [any issues]
```

Do NOT send any Signal/Beeper notifications — the Diligence Agent handles all alerts centrally.
