---
name: pass-note-drafter
description: >
  Draft investor pass note emails for opportunities Tom has decided to pass on. Two modes:
  (A) Sweep — daily evening reconciliation pass orchestrated by diligence-agent that scans
  the Opportunities DB for Status = "Pass Note Pending", catches what the pass-note-sent
  webhook handler missed, and drafts for each. (B) Webhook — invoked per single Opp via the
  notion-webhook claude-job-queue when an Opp's Status flips to Pass Note Pending; processes
  just that one page. Both modes share Steps 2–7; Mode B simply skips Step 1's queue query.
  Reads diligence materials and call notes, drafts in Tom's voice as a saved Gmail draft
  (never sends), updates Notion to "Pass (Met)" only for Opps where a pass note was already
  sent (sent-check path), and archives sent pass notes to the Notes DB.

  Trigger phrases: "pass note", "draft pass note", "pass note pending", "write pass notes",
  "draft my passes", "run pass note drafter", "any pass notes to draft", "check pass note queue",
  "I decided to pass on X — draft the note", "mark X as pass note pending and draft".
---

# Pass Note Drafter

You are drafting investor pass notes on behalf of Tom Seo (Founder & GP, Inverted Capital) for
founders whose deals he has reviewed and decided not to invest in. The goal is to produce a draft
that Tom can send with minimal or zero edits — it must sound exactly like him.

Read the Style Guide section carefully before drafting anything. Tom's voice is specific and
consistent, and getting it right is the whole point.

---

## Notion Context

```
Opportunities data_source_id: fab5ada3-5ea1-44b0-8eb7-3f1120aadda6
Database URL: https://www.notion.so/5fa871c765d74251b8f96b63f248ef25
People DB data_source_id: 1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9 (use notion-search only)
Notes DB data_source_id: e8afa155-b41a-4aa2-8e9d-3d4365a11dfb

Key fields on each Opportunity:
- Name (title): Company name
- Status (status): includes "Pass Note Pending" — this is what you're scanning for
- Contact (text): founder email address(es)
- 🏁 Founder(s) (relation): links to People DB — use to get founder's first name
- Description (text): company one-liner / body context
- Diligence Materials (files/links): pitch decks, memos, one-pagers
- Notes (relation): links to call notes and transcripts in the Notes database
- Source(s) (relation): links to the person/firm who made the intro — use to personalize the opener
```

---

## Modes

**Mode A — Sweep (default).** No `page_id` arg. Runs Step 1 (queue query against the Agent View), then Steps 2–7 over every page returned. Used by the scheduled diligence-agent path as reconciliation.

**Mode B — Webhook.** Invoked with `args: { mode: "webhook", page_id: "<uuid>" }` from the notion-webhook claude-job-queue when an Opp's Status flips to Pass Note Pending. Skips Step 1 entirely; the queue is `[{ id: page_id }]` of size 1. Then runs Steps 2–7 for that single page. All step logic, dedup, archive behavior, and the Step 7 alert format are identical — just scoped to one row.

For Mode B specifically:
- Idempotency: if Step 2's draft-dedup check finds an existing Gmail draft for `[Company] - Inverted follow up`, exit silently with no Slack alert (avoid noise on duplicate webhook fires).
- Status guard: re-check the Opp's current Status before drafting. If it's no longer "Pass Note Pending" by the time the job runs (e.g., Tom flipped it to "Pass (Met)" manually), exit without drafting.
- Step 7 alert format unchanged but reports N=1.

---

## Workflow

### Step 1: Get Pass Note Pending Queue from Notion

> **Mode B note:** skip this step — your queue is the single page passed in via `args.page_id`. Fetch that page via `notion-fetch`, verify its current Status is still "Pass Note Pending" (exit if not), and proceed to Step 2 with a queue of one. Resume here only for Mode A.



Notion is the source of truth. Start here — not Gmail.

Use `notion-query-database-view` to query the **Agent View** of the Opportunities database:

```
Agent View URL: https://www.notion.so/5fa871c765d74251b8f96b63f248ef25?v=31400beff4aa80fdb2e0000c1b6ae673
```

This view returns all pipeline opportunities. After retrieving the results, **filter locally** for pages where `Status = "Pass Note Pending"`. The Agent View includes Pass Note Pending opportunities — you just need to isolate them from the broader result set.

For each matching page, collect the `id`, `Name`, `Status`, `Contact`, `🏁 Founder(s)`, `Source(s)`, `Diligence Materials`, `✍️ Notes`, and `Description` properties. These are your confirmed pending opportunities.

**If no results match `Pass Note Pending`:** Stop. Send the Signal alert (Step 7) saying nothing is pending, and exit.

---

### Step 2: Sent-Check — Identify Already-Sent Pass Notes and Update to Pass (Met)

Now that you have the confirmed pending queue, check each company against Gmail — not the other way around. For each Pass Note Pending opportunity, do a targeted Gmail sent search for that specific company:

```
gmail_search_messages: q="subject:\"[COMPANY NAME] - Inverted follow up\" in:sent -subject:\"Re:\" -subject:\"Fwd:\""
```

If a sent email is found (it must be in Sent mail — not just a draft), it means Tom already sent the pass note manually. Update that Opportunity's status to `"Pass (Met)"` via `notion-update-page` and remove it from the working set. **PROTECTED STATUS GUARD: Before updating Status, verify the opportunity's current status is not Active Portfolio, Portfolio: Follow-On, Exited, or Committed. If it is, do NOT update — skip and note "skipped — protected status" in the Signal summary.**

**Archive the sent pass note to the Notes DB.** After flipping status to `Pass (Met)`, create a Notion Notes entry with the final sent version of the email so the pass note is preserved on the Opportunity:

1. Fetch the full sent email body via `gmail_get_message` using the message ID from the sent search above. Extract the plain-text body (strip the signature block from `–` onward and any quoted reply history — just the outbound pass note Tom wrote).
2. **Dedup check:** Query the Notes DB (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) for an existing page with Name = `[Company Name] - Inverted follow up` and the same Opportunity relation. If one exists, skip creation.
3. Otherwise, create a new Notes page via `notion-create-pages`:
   - **Parent:** `data_source_id: e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`
   - **Name (title):** `[Company Name] - Inverted follow up` (use the exact company name as it appears in the Opportunity title, matching the email subject)
   - **Category:** `Diligence` (hardcoded — do NOT invoke note-classifier for this flow)
   - **Opportunity:** relation to the passed Opportunity's page URL
   - **Body:** the full plain-text pass note as paragraphs, preserving line breaks between bullets. Do NOT include `Tom,` opener addressing wrapper formatting — write the body as it appears in the sent email.

This ensures every sent pass note has a durable record on the Opportunity alongside call notes and diligence materials, not just in Gmail. The archive is created only on the sent-detection path — never at draft time.

This keeps the lookup targeted: you're only querying Gmail for companies you already know are pending — not scanning all of Tom's sent mail and reverse-matching against Notion. Any Pass Note Pending opportunity for which no sent email exists stays in the queue for drafting.

> This step is the only mechanism by which an Opportunity moves out of "Pass Note Pending" — the drafting steps below deliberately do not update Notion status, since Tom reviews and sends the draft himself.

**Deduplication check:** For remaining entries, run `gmail_list_drafts` and check for an existing draft subject matching `[Company Name] - Inverted follow up`. If a draft already exists, skip drafting for that company and note it in the Signal summary as "draft already exists — review and send."

Proceed to draft for any remaining entries.

---

### Step 2.5: Backfill Missing Archives for Recently Flipped Pass (Met)

The Gmail webhook (`gmail-webhook/pass-note-sent.js`) also flips Pass Note Pending → Pass (Met) and creates the Notes archive in real time on send. But if the webhook's archive step fails partway (e.g., a body-decode error or transient Notion error) while the status flip succeeds, the Opportunity ends up flipped without a corresponding Notes entry — and Step 2 above won't backfill because it's gated on the status transition.

This step closes that gap. After Step 2 completes:

1. Query the Opportunities DB for entries where `Status = "Pass (Met)"` AND `Last Edited` is within the last 7 days. Filter via `notion-query-database-view` against the Agent View; locally narrow to recent Pass (Met) rows.
2. For each, check whether a corresponding Notes DB page exists with Name = `[Company Name] - Inverted follow up` and an Opportunity relation pointing to that Opp. Use `notion-search` against the Notes DB.
3. If no Notes page exists:
   - Find the sent pass note in Gmail with `gmail_search_messages: q="subject:\"[Company Name] - Inverted follow up\" in:sent -subject:\"Re:\" -subject:\"Fwd:\""`
   - If found, fetch the message body and create the Notes page using the same template as Step 2 (Diligence category, Opportunity relation, body = stripped plain-text pass note, with a `📧 View sent email` link at the top).
   - If no sent email is found, skip — the Pass (Met) flip happened via some other path and there's nothing to archive.
4. Note any backfills in the Signal alert as `🔄 Backfilled archive: [Company]`.

Skip this entire step if no Pass (Met) opportunities were edited in the last 7 days — common case is zero work.

---

### Step 3: Gather All Context for Each Opportunity

For each confirmed "Pass Note Pending" opportunity, collect:

**3a. Founder info + recipient list**
- Pull the founder's first name from the `🏁 Founder(s)` relation (use `notion-search` on People DB with the linked page ID, then fetch to get the Name field).
- If People DB lookup fails, infer the first name from the first email in `Contact` (e.g., `peter@sync.bio` → "Peter") or from any call notes/transcripts.
- **Recipient list = full Contact field.** The Contact field is the canonical recipient store. It contains a `;`-separated list of every email Tom met with on this deal (founder + any co-founders / partners / chiefs of staff who joined the calendar invite — populated by the calendar-scheduled-detect webhook). Split on `;`, trim whitespace, lowercase, dedup. Use ALL of them in the draft's To: field — pass notes go to everyone Tom spoke with, not just the primary founder.
- The first email in the list is the primary founder (calendar handler keeps it at index 0). Address the email body to that person's first name only.
- **Empty Contact fallback:** If the `Contact` field is empty, search Gmail for the company name to find the original inbound email thread (e.g., `gmail_search_messages: q="[Company Name]"`). The founder's email can typically be found as the sender in the earliest thread related to this deal. Extract it from there. If still not found, skip the draft and flag in Signal notification.

**3b. Intro source**
- Fetch the `Source(s)` relation to get the referrer's name and firm. This feeds the opener line ("I'm glad [NAME] @ [FIRM] made the intro!"). If the source is "Direct" (i.e., `https://www.notion.so/0fb9a64034fd46f9934768d590e69dc9`), omit the referrer mention entirely.

**3c. Diligence materials**
- Fetch the `Diligence Materials` field. These may be:
  - Notion-hosted files: download/read content
  - External URLs (Google Drive, Docsend, etc.): use `google_drive_fetch` or `WebFetch` to retrieve content where possible; if inaccessible, note the title and proceed with other materials
- Read each document to extract: the business description, market thesis, product approach, traction/metrics, and the team background.

**3d. Call notes and transcripts**
- The `Notes` relation links to pages in the Notes database. Fetch each linked page to read the full content — these typically contain meeting notes, call summaries, or transcripts.
- Extract key themes discussed: what Tom seemed excited about, any concerns raised, open questions, stage/context of the business.

**3e. Opportunity body**
- Read the `Description` and any body content on the Opportunity page itself for additional context (e.g., how the deal came in, quick notes Tom may have left).

**3f. First-pass diligence doc (if present)**
- If a first-pass diligence analysis exists for this company (produced by the `first-pass-diligence` skill), read it in full. These docs contain Tom's structured evaluation of the market opportunity, risks, open questions, and conviction gaps — they are high-signal source material for both the "what excited Tom" bullets (1 and 2) and the "where his hesitation lies" bullet (3).
- Look for first-pass docs via: (a) the `Notes` relation on the Opportunity (first-pass docs are logged to the Notes DB), (b) a Notion search for the company name scoped to diligence pages, or (c) the Diligence Materials field.
- Draw specifically on: the market sizing section, the thesis/strengths section, and the risks/open-questions section. These map directly onto the three-bullet structure. Do not copy language verbatim — filter through Tom's voice per the Style Guide.

---

### Step 4: Review Historical Pass Notes for Voice Calibration

Before drafting, pull 3–5 recent historical pass notes to keep Tom's voice fresh in context. Two sources, either is acceptable:

**Primary — Notion archive view (authoritative):**
```
https://www.notion.so/tomseo/74d2a6a715cd42c5aecb6f1dabd702f8?v=da24ae08f2494c19ae7f83008cdaf1d3
```
This view aggregates all historical pass notes Tom has written. When Tom asks you to refer to the historical structure and tone of pass notes, this is the canonical reference. Use `notion-fetch` or `notion-query-database-view` to pull recent entries.

**Fallback — Gmail sent mail:**
```
gmail_search_messages: q="subject:\"Inverted follow up\" in:sent -subject:\"Re:\" -subject:\"Fwd:\""  maxResults=5
```

Read 2–3 of them in full. You've already internalized the style guide below, but skimming the real emails confirms the current register and any recent stylistic shifts. Do not spend excessive time here — a quick scan is sufficient.

---

### Step 5: Draft the Pass Note

Before drafting, read the pass-note stylebook:

- `~/.claude/skills/writing-style/pass-note/STYLE.md` — canonical voice + scaffold + anti-patterns. Read in full.
- `~/.claude/skills/writing-style/pass-note/EDIT_PATTERNS.md` — two sections: **Canonical Principles** (durable, foundational rules — apply as hard rules) and **Recent Edits** (append-only log of how Tom edits Claude-drafted pass notes — apply as priors). Read both sections in full. (Re-introduce a recency/frequency cap on Recent Edits if the file ever grows large enough that drafts start coming out derivative or over-fit.)
- `~/.claude/skills/writing-style/pass-note/VOICE_EXAMPLES.md` — full pass notes Tom wrote from scratch (no Claude draft involved). Scan the 2–3 most recent for canonical voice — these are ground truth.

Both files are auto-maintained by the `draft-feedback` pipeline (FRAMEWORK_PRD.md §13). Patterns are observations, not commands.

Then produce the email body, following the Style Guide precisely.

---

### Step 6: Create the Gmail Draft + Drive Snapshot

**THIS STEP IS MANDATORY. Do NOT skip it, summarize it, or present the draft inline as a substitute. The skill is not complete until `gmail_create_draft` has been called and confirmed. Presenting the email body in the conversation is not a replacement for creating the actual Gmail draft.**

Use `gmail_create_draft` with:

- **To:** the FULL `;`-split list from Contact (Step 3a), passed as an array of email strings. Founder's email first, all other meeting attendees after. The MCP `to` parameter is `string[]`.
- **Subject:** `[COMPANY NAME] - Inverted follow up`
  - Use the exact company name as it appears in the Notion Opportunity title
  - Subject format is always `[Company] - Inverted follow up` — never deviate from this
- **No BCC.** The old Zapier BCC (`passnotes.mhcrey@zapiermail.com`) has been retired. The `pass-note-sent` gmail-webhook handler now does the same archive work natively (creates the Notes DB entry with Diligence category + Opportunity relation + view-sent-email link, and flips Status → Pass (Met) on send).
- **Body:** the drafted pass note

Do NOT send the email — only create it as a draft for Tom to review. Do NOT update the Notion status after creating the draft — the status update to "Pass (Met)" only happens in Step 1 once Tom has actually sent the email (see Step 1 above).

**After draft creation, write the Drive snapshot for `draft-feedback`.**

`gmail_create_draft` returns an `r-XXXX` transaction ID — call `gmail_list_drafts` with `query: "to:{email}"` and grab the most recent entry's persistent hex `id` (e.g., `19da8bae7d10166e`) plus its `threadId`. Then write a JSON snapshot to:

```
~/Library/CloudStorage/GoogleDrive-tom@invertedcap.com/My Drive/_system/draft-snapshots/<hex_id>.json
```

File contents:

```json
{
  "skill": "pass-note-drafter",
  "messageId": "<hex_id>",
  "threadId": "<gmail thread id>",
  "recipient": "<founder email>",
  "subject": "<Company> - Inverted follow up",
  "draftText": "<full plain-text body of the pass note — exclude the signature block from `–` onward since Gmail auto-appends>",
  "createdAt": "<ISO 8601 timestamp>"
}
```

For multi-recipient sends, write `"recipients": ["<email1>", "<email2>"]` instead of `recipient`. The `draft-feedback` processor accepts either shape.

Use the `Write` tool. Drive Desktop syncs the file within seconds. The webhook handler picks it up on send and queues a diff job for the local processor (FRAMEWORK_PRD.md §13). Unsent snapshots auto-purge after 30 days.

**Formatting:** Always create the draft as **plain text** — use `contentType: text/plain`. Do NOT use `text/html`. The reason: HTML drafts bake in font-family and font-size via inline styles, which renders differently depending on which client opens the email (Gmail web vs. Apple Mail on Mac vs. mobile). Plain text avoids this entirely — Gmail applies its own default styling on send, and the recipient sees a clean, consistent message regardless of client.

**Greeting matches recipient count.** If the To list has one address: `Hey [Founder First Name],`. Two addresses: `Hey [First Name 1], [First Name 2],` (comma-separated). Three+: `Hey [F1], [F2], and [F3],` (Oxford comma). Pull first names from People DB Founder relation when possible; else infer from the email local-part. Body references that previously named one founder ("Between you and Yehuda") should switch to plural framings ("Between you two", "Between the three of you") when multiple are addressed.

Use this exact template structure — note the single blank line between each paragraph block:

```
Hey [First Name],

[opening paragraph]

[transition + "100% unsolicited..." line]

* [bullet 1]

* [bullet 2]

* [bullet 3]

[humility close]

Best,
Tom

–

Tom Seo
Founder & GP, Inverted Capital
m:  +1 (201) 256-7714
e:   tom@invertedcap.com
```

Key points: each paragraph and each bullet is separated by a single blank line. Bullets use `* ` prefix (asterisk + space). The signature block starts with an em dash (—) on its own line, followed by a blank line, then the name/title/contact lines with no extra spacing between them. Throughout the body, use en dashes (–) not em dashes — the signature separator is the sole exception. Write dollar signs as plain `$` — do not escape as `\$`., followed by a blank line, then the name/title/contact lines with no extra spacing between them. Write dollar signs as plain `$` — do not escape as `\$`.

---

### Step 7: Send Alert

Read the `send-alert` skill (discover via Glob pattern `**/send-alert/SKILL.md`) for the delivery channel, tool, chatID, and guardrails. Use the consolidator-style format below — header with colon + date, no intro line, one bold/underlined entity header per company with a Notion link, drafted-for line directly underneath, no trailing call-to-action. Compose with `md_to_blocks.py` using GFM (per memory `feedback_md_to_blocks_format_traps.md`).

```
✍️ Pass Note Drafter: YYYY-MM-DD

🏢 **<u>[Company] | [Notion](url)</u>**
Drafted for: [Founder First Name] ([founder email])
```

Multi-entity batches stack the entity blocks with a single blank line between each. Single-entity (Mode B / webhook) drops straight into the entity block — no count line.

If any opportunities failed (e.g., couldn't find founder email, draft creation failed), append:
```
⚠️ Failed: [Company] — [brief reason]
```

---

## Style Guide

This section is the core of the skill. Study it carefully — the whole point is for drafts to be indistinguishable from Tom writing them himself.

### Voice and Tone

Tom's pass notes are warm, thoughtful, and genuinely personal. They are not form letters. He clearly spent time thinking about each company and wants the founder to feel that. The tone is: collegial, intellectually engaged, honest without being blunt, and self-deprecating at the close. He is rooting for the founder even though he's passing.

Key characteristics:
- **Casual but substantive.** Uses contractions, conversational rhythm, en dashes (–) throughout the body, ellipses. Not stiff or overly formal. Note: the signature separator is an em dash (—), which is the one exception to this rule.
- **Specific.** Generic praise ("great team!") never appears. He always names the actual thing he found impressive.
- **Self-aware about his own limitations.** When he passes, he often frames his hesitation as a gap in his own conviction or fit, not a verdict on the company's quality.
- **Genuine humility at the close.** Always acknowledges he might be wrong.

### Email Structure

Follow this structure exactly — it is consistent across every pass note:

**1. Greeting**
First name only (no last name), followed by a comma:
- `Hey [First Name],` — most common, casual
- `[First Name],` — also used; slightly more direct

**2. Opening paragraph**
Two to three sentences. Warmly re-anchors to the conversation. Almost always:
- Thanks them for their time
- Expresses genuine enjoyment of the conversation ("Really enjoyed the convo", "Hopefully you could tell that I really enjoyed the convo")
- Names the referrer if the deal came via intro: "I'm glad [First Name] @ [Firm] made the intro!" — note the format is first name + "@" + firm name (not full name), e.g. "I'm glad David @ Shirlawn made the intro." Use this format when the firm name adds useful context. If no firm is known, use "I'm glad [First Name] intro'd us."
- Optionally references something specific from the timing ("Thanks again for taking the time to chat earlier this week", "Hope you had a great weekend and thanks again for…")

**3. Transition sentence**
This is a consistent phrase — use a close variant every time:
> "As promised, I had a chance to review, and unfortunately have decided to sit this round out."

Variations seen:
- "As promised, I spent more time sitting with the opportunity, and unfortunately have decided to sit this round out."
- "I had a chance to review, and have unfortunately decided to sit this round out."

Then immediately follow with the feedback framing line. The canonical version Tom likes:
> "100% unsolicited, but to the extent you're interested here's where my head's at:"

Variations:
- "Entirely unsolicited, but in case helpful, here's what my head's at:"
- "To the extent you're interested, here's my thinking:"

**4. Bullet points — this is the core content**

Always exactly **3 bullet points** (using `*` not `-`):

- **Bullet 1 — Market/opportunity strength:** What genuinely impressed Tom about the market or thesis. Specific to THIS company. He often comments on: the size/pain of the market, the non-obviousness of the approach, a specific insight or strategy that resonated. Phrases like "You're clearly going after...", "You're very obviously going after...", "I appreciate the rigor with which you've validated..."

- **Bullet 2 — Founder strength:** What impressed Tom about the founder specifically. He almost always has a founder-specific observation: their domain expertise, their empathy with customers, their execution speed, their strategic thinking. Phrases like "I'm also struck by...", "I also appreciate that you're...", "It's also evident that you're the right person to build this..."

- **Bullet 3 — The constructive feedback (the pass reason):** What Tom still needs to see de-risked, or where his hesitation lies. **Important: the pass reason is not always a thesis critique.** When the deal is genuinely strong but out of scope for Tom's fund (capital intensity, geography, sector fit), Bullet 3 should frame the pass as a fund-fit limitation rather than a company-specific concern. The canonical construction for this: "More of my own doing than anything else: as a small pre-seed fund, it's hard for me to get behind a business that will eventually be capital-intensive." This localizes the pass entirely to Tom's constraints and is a deliberate act of respect for the founder. Do NOT manufacture a company-level critique when the real reason is fund-fit. Use "I'm hoping to see more de-risking on...", "That said, I'm unfortunately not quite 'there' on...", "Where things feel earlier is..." only when the pass reason is genuinely thesis-related. When it's fund-fit, lean into the self-attribution: "More of my own doing," "just the reality of my fund structure," "nowhere near as deep as I want to be in [space]."

Bullets should be substantive — 2–5 sentences each. They must be grounded in the actual diligence materials and call notes, not generic observations.

**In-bullet forward references:** Tom sometimes uses a parenthetical within a bullet to foreshadow what's coming in the next one — e.g., "(more on this below)". This creates forward momentum within the email and signals to the founder that the points are connected. Use this sparingly (at most once per email) when two consecutive bullets are causally linked.

**5. Closing paragraph**
This is nearly identical every time — match it closely:
> "I will say… there's a good chance that I look silly in hindsight for making this decision. If you're up for it I'd love to stay in touch. Wishing you the very best!"

Variations:
- "there's always a good chance that I look silly in hindsight"
- "there's a good chance that I look like an idiot in hindsight"
- The stay-in-touch line is sometimes omitted if the relationship context doesn't warrant it (rare)

Note on the ellipsis: "I will say…" uses an ellipsis (no period before it) as a deliberate rhetorical beat — a breath before the humility close. Do not replace with a period or comma. This is a stylistic signature.

**6. Sign-off and signature**
Always:
```
Best,
Tom

—

Tom Seo
Founder & GP, Inverted Capital
m:  +1 (201) 256-7714
e:   tom@invertedcap.com
```
Note the two-space indent before the phone/email values — match exactly. The signature separator is an em dash (—). Throughout the body of the email, use en dashes (–) not em dashes — the signature is the one exception.

### What to Avoid

- Do NOT write bullets that are generic ("great team", "large market"). Every observation must be tied to a specific detail from the materials or call notes.
- Do NOT be harsh or deliver a verdict on the company. The tone is always "not the right fit for me right now" not "this won't work."
- Do NOT over-explain the pass reason. One well-constructed constructive bullet is better than a laundry list of concerns.
- Do NOT use formal investor jargon ("market sizing", "go-to-market motion", "unit economics thesis"). Tom writes naturally.
- Do NOT skip the humility close. It's a signature element.
- Do NOT deviate from the subject line format. It must be `[Company] - Inverted follow up` — no variations.
- Do NOT regurgitate raw statistics as facts in the bullets. A line like "given that CodePath does $30M annually" reads like an AI reciting a data point — Tom writes from observation and inference, not from a fact sheet. Instead of citing a metric flatly, express what that metric *reveals*: e.g., "what you've built at CodePath — taking it to meaningful scale as a non-profit — speaks to your ability to navigate complex stakeholder environments" is Tom's register; "CodePath does $30M annually" is not. Use specific context from the diligence materials as texture, but always filter it through Tom's perspective and voice.
- Do NOT escape dollar signs with backslashes. Write `$1-2M` not `\$1-2M`. The body is plain text, not markdown.

### Handling Sparse Context

If diligence materials and call notes are thin:
- Pull whatever is available from the Opportunity body/description
- Fall back to reading the company's public website via WebFetch if needed
- Write bullets that are genuine but appropriately concise — better a shorter, accurate note than a longer, padded one
- If you genuinely cannot identify the founder's email, do NOT create the draft — flag this in the Signal alert and leave the Notion status unchanged

---

## Edge Cases

- **Multiple founders:** Address the email to the primary founder (the one Tom spoke with or whose email is first in the Contact field). If unclear, address the one whose name appears first in the Founder(s) relation.
- **Multiple opportunities in one run:** Process each independently. Create a separate Gmail draft per company. Report all in a single Signal notification.
- **Direct deal (no intro source):** Omit the referrer mention from the opening paragraph. Adjust the opener accordingly ("Thanks again for taking the time to chat!")
- **Gmail draft creation fails:** Leave Notion status as "Pass Note Pending" so it gets picked up next run. Flag in Signal notification with a brief reason.
- **Draft already exists (deduplication):** If a Gmail draft for `[Company] - Inverted follow up` already exists from a prior run, skip re-drafting and note it in the Signal summary as "draft already exists — review and send."
