---
name: family-inbox
description: >
  Read-only agent over the family Gmail (kenyonseo@gmail.com) — household scope, usable
  by both Tom and Elsie. Two modes. (C) Manual/on-request — triage & search the inbox
  when someone texts "any new school emails?", "what's in the family inbox?", "did X
  email come in?": handled inline by sms-listener's Family-inbox section (this skill's
  reader). (A) Webhook — the family-gmail-webhook Apps Script (bound to kenyonseo@, ~5-min
  cloud trigger) enqueues one family-inbox job per NEW inbox message; this skill classifies
  it and, if notable, texts a heads-up to Tom via Sendblue and proposes any date-bearing
  item for the family calendar (never silently). Never sends, deletes, or marks mail read.
  NOT Tom's work Gmail — that stays fully off-limits to Elsie.
---

# Family Inbox

Read-only over **kenyonseo@gmail.com** via `family_inbox.py` (IMAP, app password in env
`FAMILY_GMAIL_APP_PASSWORD`). Household scope — Tom AND Elsie. Never write/send/delete.

## Mode C — on request (inline)
Driven by sms-listener's "Family inbox" section — search/triage/summarize on demand. See
`family_inbox.py` commands: `recent N` · `unseen` · `since <date>` · `search '<gmail query>'`
· `get <uid>`.

## Drafting — allowed; SENDING — NEVER (standing rule)
The agent MAY draft family emails from kenyonseo@ on request; it may **NEVER send**. This
is enforced by construction — the only email-write path is a Gmail DRAFT via IMAP APPEND;
there is no SMTP anywhere. The human reviews + sends from Gmail.
```bash
python3 ~/.claude/skills/family-inbox/family_inbox.py draft "<to>" "<subject>" "<body>" ["<in_reply_to_msgid>"]
```
- Triggers: "draft a reply to <the school/pediatrician/…>", "draft an email to X", "reply to
  that email" (when the requester wants a draft, not a send).
- For a reply, fetch the source (`get <uid>`) for context + the Message-ID to thread it.
- After creating: tell the requester it's a DRAFT in the kenyonseo@ Drafts folder for them to
  review and send — and NEVER offer to send it. If anyone says "send it," the answer is that
  the agent can't send; it's ready in Drafts for them to hit send.

## Mode A — webhook (ONE email per job, source=gmail-webhook-family)

Push-driven, not polling. The `family-gmail-webhook` Apps Script (bound to
kenyonseo@gmail.com, ~5-min trigger) enqueues one `family-inbox` job per NEW inbox
message. This job IS one such message — the args carry everything, so NO IMAP re-fetch:
```
args: { messageId, from, subject, date, body }   # body = first ~2000 chars, plain text
```
Unattended. Never ask questions.

1. **Classify** from the args:
   - **NOTABLE** — school (BFS, teachers, PTA, signups), kids' activities (Soccer Stars,
     NikosKids, camps), appointments/confirmations/reschedules, bills/payments due,
     delivery problems, anything time-sensitive or personally addressed.
   - **SKIP** — marketing, newsletters with no dated action, promos, routine receipts.
2. **SKIP → exit silently.** No text. Err toward SKIP.

### 3. Date-extraction harness (the core of NOTABLE handling)
Many notable emails — especially the **BFS weekly** (Andy's school) — carry multiple
dated items. Extract EVERY concrete date/event/deadline (first day, picture day, no-school
days, gatherings, conferences, due dates, appointment times).

**Relevance filter FIRST — drop anything not for this family:**
- **BFS / school mail:** Andy is in the **ECLS / Early Childhood (EC)** program. Keep
  **EC / ECLS / "all school"** items; **DROP Middle School (MS) and Upper School (US)**
  items entirely (see [[reference_family_members]]). A BFS weekly lists all divisions —
  only EC/ECLS lines are ours.
- Other mail: keep only genuinely family-relevant dates.

**For EACH relevant date, dedup-check the family calendar BEFORE anything else:**
`mcp__claude_ai_Google_Calendar__list_events` on the Elsie-Tom calendar
(`cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com`) over that date. Judge
semantically — the BFS feed already auto-adds many milestones (Labor Day, Family Visit
Day, First/Second Day), so most will already exist.

- **DUPE found → ENRICH, don't re-add.** `update_event` to fill in any MISSING detail the
  email provides — a specific time if the existing event is all-day, a location, a
  description. Don't overwrite correct existing info; only add what's missing. (This is an
  authorized auto-update — no approval needed — but you STILL tell them, see step 4.)
- **NET-NEW relevant date → PROPOSE, never auto-add.** Hold it for the text and ask for an
  explicit OK from Tom OR Elsie before it goes on the calendar.

### 4. Text ONE consolidated heads-up (Sendblue) — to the FAMILY GROUP
(Load prefs first — `python3 ~/.claude/skills/sms-listener/prefs.py load core email` — and
honor them, esp. the header/formatting rules.)
Kid/family stuff goes to the **family group** (Assistant + Tom + Elsie) so both parents
see it — NOT Tom's 1:1:
```bash
GID="$(cat ~/.claude/skills/sms-listener/.family_group_id 2>/dev/null)"
if [ -n "$GID" ]; then
  ~/.claude/skills/sms-listener/send_imessage.sh --group "$GID" "<heads-up>"
else
  ~/.claude/skills/sms-listener/send_imessage.sh "+12012567714" "<heads-up>"   # fallback: Tom 1:1
fi
```
Cover both buckets, clearly separated:
```
📬 BFS weekly — 3 EC dates
✓ Already on cal (enriched): Grown-Up Gathering Fri 9/11 — added 8:45am start
✓ Already on cal: First Day of School Wed 9/9
🆕 Picture Day — Tue 10/7 (EC). Add to family cal? reply "add picture day"
```
- `✓` lines = dupes you already handled (enriched noted). Tom asked: even when it's
  already on the calendar, still tell them you saw a relevant date in the email.
- `🆕` lines = net-new relevant dates awaiting explicit approval. **Never added until Tom
  or Elsie replies** (e.g. "add picture day") — that reply runs the calendar fast path
  (dedup-checked) via sms-listener. Multiple new dates → they can approve individually or
  "add all". A reply in the group (from Tom OR Elsie) runs sms-listener → adds to the
  family calendar (group scope applies; adding a family event is in-scope for both).
- **CTA = a reply → offer to draft it.** If the notable email's call-to-action is
  responding (RSVP, "reply if interested", email back to sign up, confirm attendance),
  close the heads-up with an offer to draft the reply — e.g. append a line like
  `Want me to draft a reply? reply "draft it"`. On that reply, use the Drafting path above
  (`family_inbox.py draft …`, threaded via the source Message-ID) → creates a kenyonseo@
  DRAFT for review. NEVER sends. (Non-date notes with a reply CTA — like a class volunteer
  ask — are notable on their own; text the heads-up + draft offer even with no dates.)
- If, after the relevance filter, there are NO relevant dates, no reply CTA, and nothing
  else notable → exit silently.

### 5. Log
Append to `~/.claude/skills/family-inbox/sweep-log/YYYY-MM.log`:
`[ts] msg=<messageId> from=<from> dates_found=<n> enriched=<k> proposed=<j> texted=<y/n>`.

**Calendar access:** the processor's cold-path `claude --print` inherits the Google
Calendar MCP, so `list_events` / `update_event` work here. `SENDBLUE_API_SECRET` is
injected; the send helper reads `.sendblue_config` from the sms-listener dir. The email
body is in the args — no IMAP fetch needed.
