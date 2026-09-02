---
name: deal-text-scanner
description: >
  Scans NEW inbound 1:1 iMessages on Tom's personal number for deal-flow signals — an
  individual offering an intro to a founder, a founder raising, a deck/LinkedIn shared with
  round details — and texts Tom a card proposing a CRM add: header "🆕 Opportunity:
  <Company>" when the company is named (company only, add-to-crm convention), else
  "🆕 Opportunity: -1 (Founder)" / "🆕 Opportunity: NewCo (Founder)", with
  Source/Stage/Terms/HQ/Description bullets. NEVER adds to CRM itself: Tom's 👍 tapback or "confirm" reply (handled by
  sms-listener) triggers the add-to-crm skill. Scheduled-sweep-only (launchd →
  sweep.sh code gate → this job). TOM-ONLY surface: reads his personal texts; nothing here
  is family-scoped. Most messages are NOT deals — default to silence.
---

# Deal Text Scanner

Input args: `{mode:"scan", messages:[{rowid, ts, sender, from_me, text}]}` — new 1:1 texts
since the last sweep, pre-filtered in code (no groups, no family, no short codes).
`sender` = the thread's peer handle; `from_me:1` rows are TOM'S OWN replies — context, never
candidates themselves, but essential for reading his in-thread response (a pass, an "I'm
in", a question). **Resolve the peer's real name before proposing:** try
`mcp__imessages__tool_find_contact` (or check_contacts) on the handle; the LinkedIn
preview title in-thread often names the founder too. Fall back to the raw handle only if
lookup fails.

**Attachments are prime signal.** `attachments` = display names, `attachment_paths` =
local file paths (`~/Library/Messages/Attachments/…`). A deck filename often names the
company outright ("MaxHeap Fika Version.pdf" → company = MaxHeap). For a PDF on a
candidate thread, read it locally (`pdftotext <path> - | head -80`) to pull company name,
one-line description, stage/terms, HQ — that's where the card's facts usually live.
Never send the attachment anywhere; read-only, quote only the card-relevant basics.

## 1. Classify — high bar, default silent

A message (or a run of messages from one sender — read them as a conversation) is a
**deal-flow candidate** only if it contains a concrete investable signal:
- an intro offer to a founder ("met a founder doing X, raising soon, want an intro?")
- a founder/company + round details (raise amount, valuation, stage, deck, data room)
- a shared LinkedIn/deck accompanying a pitch or referral

NOT candidates (stay silent): scheduling, social chatter, thank-yous, LP/fund-admin
notes, portfolio-company ops, favor-forwards (dinner invites etc.), news links without a
referral, anything from a company/service number. When unsure → silent. False negatives
are fine (Tom sees his own texts); false positives erode trust.

## 2. Dedup

`~/.claude/skills/deal-text-scanner/.proposed` holds one line per prior proposal
(`YYYY-MM-DD <founder/company> via <referrer>`). Grep it first — same founder/company
already proposed → skip. Append a line for each new proposal you send.

## 3. Propose to Tom (1:1, NOT the family group)

For each candidate, ONE text via Sendblue to Tom (`+12012567714`), capturing the handle
for the confirm flow:

```bash
H=$(~/.claude/skills/sms-listener/send_imessage.sh "+12012567714" "<text>") && H=${H#ok }
```

EXACT shape (Tom's -1 sourcing card format — match it, including the blank line after the
header):
```
🆕 Opportunity: -1 (Mason Zhang)

* Source: Erik Ronning
* Stage: Pre-Seed
* HQ: San Francisco
* Description: Automating dental clinics.

👍 to Add to CRM. Respond to make changes.
```
(With disclosed terms, they collapse INTO the Stage line after a semicolon:
`* Stage: Seed; $5-6m on $25-30m pre`. No terms → just `* Stage: Seed`.)
- Header is always `🆕 Opportunity: <subject>`. Subject — same convention as add-to-crm:
  named company → company ONLY (`🆕 Opportunity: MaxHeap`, no founder in parens); no
  company name → founder in parens, split by WHERE THEY ARE (Tom 2026-09-01):
  `NewCo (<Founder>)` = they've settled on an idea and are raising (or about to) — the
  company just isn't named yet (e.g. "automating dental clinics, raising a seed soon");
  `-1 (<Founder>)` = caught BEFORE the leap — still AT their current company (hasn't
  left yet), or exploring/pre-idea with no settled concept. Still-employed beats
  idea-status: someone with an idea who hasn't left is -1; NewCo requires having
  jumped AND settled on the idea + raising. Resolve names (contact lookup, LinkedIn slug/preview) —
  never a raw phone number in the header.
- Fields, one `* ` bullet each, in THIS order: `Source` (the referrer's name), `Stage`
  (MUST be one of the CRM's Notion Stage options — Pre-Seed, Seed, Series A, … — plain
  name on the card; add-to-crm applies the emoji variant (`Pre-Seed 💡`, `Seed 🌾`) at
  write time. Infer from thread: "raising a seed soon" → Pre-Seed; round terms stated →
  that round; a `-1` is ALWAYS Pre-Seed per add-to-crm. Terms disclosed in-thread →
  append to this SAME line after a semicolon, normalized with $ and m units:
  `Stage: Seed; $5-6m on $25-30m pre`. No terms → stage alone, no semicolon), `HQ` (ALWAYS present, formatted to MATCH the
  add-to-crm skill's Notion HQ field convention — the major CITY, e.g. "San Francisco",
  "New York": this value flows straight into the CRM row on confirm. Not in the
  thread/deck? LOOK IT UP: ContactOut on the in-thread LinkedIn
  (`mcp__contactout__contactout_enrich_linkedin_profile`, profile_only=true) — the
  profile `location` gives the founder's metro; normalize to the city (Los Altos/Bay
  Area → "San Francisco"). `HQ: ???` only when there's no LinkedIn and no lookup path —
  never fabricate, never omit the line),
  `Description` (one clause, from the thread).
- Closer line is always `👍 to Add to CRM. Respond to make changes.` (a "confirm" reply
  works too). NO other lines — no `Note:` bullets, no commentary on Tom's in-thread
  response. His 👍/confirm adds to CRM; a reply with corrections updates the proposal
  (sms-listener owns that loop and makes edits land in the CRM entry).
- Include only what the thread actually supports — deep enrichment happens in add-to-crm
  after the confirm, not here.
- Append the audit line with `sent_handle=$H` and `notes=proposed add-to-crm <founder> via <referrer>`
  to `~/.claude/skills/sms-listener/audit-log/$(date +%F).log` — this is what lets
  sms-listener bind Tom's 👍/confirm (or inline-reply) back to THIS proposal.

## 3b. STAGE the CRM payload (this is what makes the 👍 instant)

You already did the expensive work (classification, contact/HQ lookup, deck read). Write it
down so the confirm doesn't redo any of it: save
`~/.claude/skills/deal-text-scanner/staged/<sent_handle>.json` with everything add-to-crm
needs, shaped to its conventions:

```json
{
  "company": "MaxHeap",            // null for -1/NewCo
  "opp_title": "MaxHeap",          // exact Opp title per add-to-crm (e.g. "-1 ([Mason Zhang](li_url))")
  "founder": "Vipul Sharma",
  "founder_linkedin": "https://www.linkedin.com/in/vipulethos",
  "stage": "Seed 🌾",              // exact Notion option incl. emoji
  "round_details": "$5-6m on $25-30m pre",  // = the card's Stage-line terms verbatim ($Xm on $Ym pre/post/cap; "Raising $Xm" only when NO valuation disclosed), or null
  "icon": "🛡️",                    // thematic emoji for the Opp page icon — NEVER blank
  "contact": "N/A",                // founder email if found at stage time, else literal "N/A" — never empty
  "website": "N/A",                // company site if known, else "N/A"
  "hq": "San Francisco",           // CRM HQ convention, or null if ???
  "description": "Risk intelligence layer for the agentic enterprise.",
  "source": "TX Zhuo",             // referrer name (CRM Source)
  "source_context": "<1-3 sentence summary of the text thread — who said what>",
  "links": ["..."],
  "deck_path": "~/Library/Messages/Attachments/.../MaxHeap Fika Version.pdf",
  "deck_drive_link": "https://drive.google.com/file/d/…",  // see below; null if no deck
  "proposed_at": "<ts>"
}
```

**Deck → Drive UPFRONT.** If the thread carried a deck/PDF, upload it NOW (at stage time)
to Drive `Deal Docs/<Company>/` (create the folder if needed — the standard deal-docs
layout), and record the link as `deck_drive_link`. That way the confirm just chips it onto
the Opp's Materials (`notion_files_property.py --no-alert`) — no upload on the hot path.
The whole point: every slow step (lookups, deck read, Drive upload) happens BEFORE Tom's
👍; the confirm is a pure Notion write.

On card EDITS (sms-listener resends a corrected card), the new proposal writes a fresh
staged file under the new handle with the edits applied.

## 4. PIPELINE PROGRESSION — act on Tom's own replies (Tom 2026-09-01)

The batch includes Tom's outbound rows (`from_me:1`). They aren't just context — when a
deal thread shows TOM RESPONDING to the referrer/founder, move the pipeline:

- **Opt-in** ("yes would love that", "make the intro", "send it over") on a deal that has
  (or is getting) an Opp → set the Opp's `Status` to **`Outreach`** (Tom replied opting
  into the intro; per add-to-crm Step 5 — flips to `Connected` only once an actual intro
  thread exists).
- **Decline** ("out of scope", "gonna pass", "not for me") → `Status` = **`Pass (DNM)`**.
- Find the Opp by company/founder title search on the Opportunities DB
  (`collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). No Opp yet → just reflect the
  response in the proposal card flow (the staged status can be Outreach/Pass accordingly
  when Tom later confirms the add).
- **Every status change gets an alert text to Tom**, nested under that deal's 🆕 card when
  one exists (sent_handle from audit log), exactly two lines:
  `🎯 Moved to Outreach — you opted into <referrer>'s intro` / `🚫 Moved to Pass (DNM) —
  you passed in-thread`, then the Opp URL.
- Never DOWNGRADE a status (e.g. Connected → Outreach) and never touch protected statuses
  (Active/Track/portfolio states) — only Qualified→Outreach and →Pass moves from this
  flow. Ambiguous reply → do nothing.
- **Cross-channel handoff:** the EMAIL side owns the next transitions — `outreach-detector`
  (Qualified→Outreach on Tom's sent emails; matches text-originated Opps by name/contact,
  no Gmail thread needed) and `intro-connected-detect` (Outreach→**Connected** when the
  three-way intro email lands). This skill owns the TEXT-side equivalent, below.
- **On every opt-in, write the watchlist:** append a line to
  `~/.claude/skills/deal-text-scanner/.expected_intros`:
  `<date> | <founder name> | <opp url> | referrer=<name> <referrer handle(s)>`. This is
  what makes an unknown number identifiable later. Remove the line once Connected.

## 4b. TEXT INTROS — identifying an unknown number

An intro'd founder texts from a number that ISN'T in Contacts. Never expect a pre-tagged
handle — INFER from converging signals against `.expected_intros`:

1. **Group composition (strongest).** A small group thread whose `participants` include a
   watchlisted REFERRER's handle + one unknown handle → the unknown is that referrer's
   founder. The intro text usually confirms ("Tom, meet Mason").
2. **Self-identification.** A 1:1 from an unknown handle whose text names a watchlisted
   founder or referrer ("Hi Tom, Mason here — Erik connected us").
3. **Expectation timing.** Unknown handle appearing within ~2 weeks of that founder's
   opt-in strengthens either signal. (Also try `tool_find_contact` on the handle first —
   sometimes they ARE in Contacts.)

**Confidence gate:**
- **≥2 signals agree (e.g. referrer-in-group + name in text)** → act: flip the Opp to
  `Connected`, alert Tom (`🔗 Connected — <referrer> intro'd <founder> over text` +
  Opp URL ↗, nested under the deal's card), record the mapping in
  `~/.claude/skills/deal-text-scanner/.known_handles` (`<handle> = <founder> | <opp url>`),
  and drop the `.expected_intros` line.
- **One weak signal / ambiguous** → don't guess. One-line ask to Tom:
  `❓ Is <handle> <founder name>? 👍 to confirm` — his 👍 completes the flip + mapping
  (sms-listener confirm flow). Never bind a number to a person on a hunch; a wrong
  mapping poisons every later inference.
- `.known_handles` is checked FIRST on any unknown sender before re-inferring.

## 4c. AFTER the text intro — capture email → draft the Blockit scheduling thread

Tom's next move after a text intro is ALWAYS: get the founder's email, start an email
thread, schedule via Blockit. The scanner watches for the email and pre-stages that:

1. **Capture the email.** After a text intro (Connected), watch the thread for an email
   address (Tom asks "what's your email?", founder replies; or it's in the intro text).
   When one appears for a watch-listed/known founder → update the Opp's `Contact` field
   (replaces N/A).
2. **Draft the scheduling email** (Gmail DRAFT on tom@invertedcap.com — NEVER send; Tom
   reviews + sends). Tom's exact pattern, taken from his real sent threads (2026-08 —
   this template IS his voice; swap names only, don't rewrite):
   - To: founder's email. **Cc: `bot@blockit.com`.**
   - Subject: `<Founder> (<Company>) / Tom (Inverted Capital)` — or for a -1/NewCo just
     `<Founder> / Tom (Inverted Capital)`.
   - Body:
     ```
     <First name> – great to be connected (thanks <referrer first name>!). Looking
     forward to chatting.

     + Blockit to coordinate – talk soon!

     Tom
     ```
   - Append Tom's standard signature; mute the draft Slack alert
     (`draft_alert_mute.sh`) since this flow sends its own text alert.
3. **Alert Tom**, nested under the deal's card:
   `📧 Scheduling draft ready — Blockit cc'd, in your Gmail drafts` + Opp URL ↗.

**The full trail (each hop owned by an existing system — don't duplicate):**
| Hop | Owner |
|---|---|
| Referral in text → 🆕 card → 👍 → Opp (Qualified) | this skill + sms-listener |
| Tom opts in / declines in-thread → Outreach / Pass (DNM) | this skill (§4) |
| Text intro from unknown number → **Connected** | this skill (§4b) |
| Email captured → Contact field + Blockit draft | this skill (§4c) |
| Tom sends; Blockit books; invite lands on GCal → **Scheduled** (+ Close Date) | `calendar-scheduled-detect` webhook + pipeline-agent sweep (already live — they specifically watch `bot@blockit.com` and verify the GCal event) |
This skill NEVER flips to Scheduled itself — the email/calendar side owns that hop —
**with ONE rare exception: scheduling agreed directly in text.** (RARE — Tom's normal
move is the §4c path: he starts the email thread himself with Blockit cc'd, from the
pre-staged draft. Only when the time gets agreed IN the text thread does this apply.
Don't stretch to find these.) Read the EXCHANGE, not
just explicit times — text scheduling is informal ("you free now?" / "free tomorrow?" /
"how about Friday morning?" / "Friday 10am?"). The trigger is a TWO-WAY CONFIRM: both
sides agree to a resolvable time, usually sealed by Tom replying something like
**"awesome – will send an invite"**. When you see that:

**→ MESSAGE BLOCKIT ON SLACK to send the invite.** Blockit lives as a conversational
bot in a Slack DM with Tom — channel **`D0BN2F3K90Q`** (bot user `U0BM1TEL5EF`). Send a
plain-English request via `slack_send_message` with everything it needs:
```
Please send an invite: me + Mason Zhang (mason@…) — Friday Sep 5, 10am ET, Zoom.
Context: pre-seed founder intro'd by Erik Ronning.
```
- **Blockit needs three things: EMAIL, DATE, TIME.** Email comes from the Opp `Contact`
  or the thread. Not available → Tom asks the founder in-thread himself; nudge him if
  he hasn't: `❓ Need <founder>'s email for the Friday invite — ask them?` and WAIT.
  The moment the email appears in the thread (§4c captures it), send Blockit the
  email + date + time. Never message Blockit with the email missing.
- Day+window but no exact time ("Friday morning") → still hand it to Blockit with the
  window; it can propose exact times.
- Then alert Tom (nested under the deal's card): `📅 Asked Blockit to send the invite —
  <founder>, Fri 10am` + Opp URL ↗.
- **Do NOT flip Scheduled yourself** — once Blockit's invite lands on the calendar,
  `calendar-scheduled-detect` flips it (same as the email path). One exception:
  an IMMEDIATE call ("free now?" + yes / "calling you") is already happening — no
  invite needed; flip `Scheduled` with Close Date = today + 2-line alert.
An unanswered "you free?" or vague "let's find time" is NOT scheduling — the two-way
confirm is the trigger. When in doubt, ask Tom, never act blind.

## 5. Exit

No candidates and no progression → exit silently (no text, no alert). Never message the
family group from this skill. Never run add-to-crm from this skill — the confirm loop
owns that.
