---
name: family-inbox
description: >
  Read-only agent over the family Gmail (kenyonseo@gmail.com) — household scope, usable
  by both Tom and Elsie. Two modes. (C) Manual/on-request — triage & search the inbox
  when someone texts "any new school emails?", "what's in the family inbox?", "did X
  email come in?": handled inline by sms-listener's Family-inbox section (this skill's
  reader). (A) Webhook — the family-gmail-webhook Apps Script (bound to kenyonseo@, ~5-min
  cloud trigger) enqueues one family-inbox job per NEW inbox message; this skill classifies
  it, dedup-checks and adds any net-new date-bearing item to the family calendar as first
  pass, and texts a heads-up to Tom + Elsie via Sendblue with what it added for them to
  audit/edit (never silently). Never sends, deletes, or marks mail read.
  NOT Tom's work Gmail — that stays fully off-limits to Elsie.
---

# Family Inbox

Read-only over **kenyonseo@gmail.com** via `family_inbox.py` (IMAP, app password in env
`FAMILY_GMAIL_APP_PASSWORD`). Household scope — Tom AND Elsie. Never write/send/delete.

## Mode C — on request (inline)
Driven by sms-listener's "Family inbox" section — search/triage/summarize on demand. See
`family_inbox.py` commands: `recent N` · `unseen` · `since <date>` · `search '<gmail query>'`
· `get <uid>`.

**Re-query fresh before asserting a negative — never trust a stale local cache** (Tom
2026-09-02, missed a real return-flight ticket this way). If a conversation spans many
turns and touches the same live inbox repeatedly (a fast-moving multi-email situation
like a family booking a trip), a PDF/email you downloaded 10 messages ago may be stale —
someone could have forwarded a new confirmation since. Before saying "X doesn't exist" /
"there's no Y ticket" / any negative claim about inbox contents, re-run the search against
the LIVE inbox first, don't just re-read files already sitting in `/tmp`.

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
   - **NOTABLE** — anything time-sensitive, dated, or personally addressed. Categories
     (Tom 2026-09-02 — **illustrative, not exhaustive**; new activities/vendors get
     added over time, classify by KIND not by matching a name on this list):
     - **School** — Brooklyn Friends School (teachers, PTA, signups, the weekly digest).
     - **Kid activities** — Little Gym, Super Soccer Stars, Music Together, camps, and
       any future class/activity signup.
     - **Reservations** — meals, tickets, flights, hotels, rentals — any booking
       confirmation with a date/time attached.
     - **Health** — doctor's office visits, appointment confirmations/reschedules,
       Magnus Health alerts, pharmacy.
     - **Bills/payments** due, delivery problems, and anything else time-sensitive or
       addressed directly to Tom/Elsie.
   - **Outside all of the above** (Tom 2026-09-02: "if I'm missing anything above and
     you deem something important enough to alert, do it") — use judgment. The category
     list is a floor, not a ceiling: something genuinely important/time-sensitive that
     doesn't fit a named bucket still gets flagged, don't silently drop it just because
     it lacks a matching label.
   - **SKIP** — marketing, newsletters with no dated action, promos, routine receipts.
2. **SKIP → exit silently.** No text. "Err toward SKIP" means low-stakes/ambiguous
   marketing-ish mail, not "only alert on an exact category match" — genuine judgment
   calls on importance still go out.

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
- **NET-NEW relevant date → ADD it directly, then report it** (Tom 2026-09-02 — "take
  first pass and text alert us with what you added, let us audit/ask for edits"). No
  more propose-and-wait: once the dedup check above comes back clean, create the event
  on the Elsie-Tom calendar right away (BUSY unless it's clearly a FREE-type marker per
  the calendar fast-path rules) and put it in the heads-up as an **already-done** ✅ line,
  not a 🆕 ask. Dedup-before-add is still absolute — never skip that check just because
  adding is now automatic. If a date is genuinely ambiguous (ex: ambiguous group/time
  assignment, conflicting info in the email) hold it as a 🆕 ask instead of guessing.

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
🆕 Added: Picture Day — Tue 10/7 (EC)
```
- `✓` lines = dupes you already handled (enriched noted). Tom asked: even when it's
  already on the calendar, still tell them you saw a relevant date in the email.
- `🆕 Added:` lines = net-new relevant dates you ALREADY put on the calendar (dedup-
  checked first, per above) — first-pass, not a proposal. Tom/Elsie audit after the fact;
  a reply like "move picture day to 9am" / "remove that" / "wrong calendar" is an edit
  request, handled via the normal sms-listener calendar fast path against the event you
  just created (you know its id from this same turn). Only fall back to a 🆕 **ask**
  (no "Added:", ends with a question) for a date that's genuinely ambiguous and you don't
  want to guess wrong — see below.
- **CTA = a reply → offer to draft it.** If the notable email's call-to-action is
  responding (RSVP, "reply if interested", email back to sign up, confirm attendance),
  close the heads-up with an offer to draft the reply — e.g. append a line like
  `Want me to draft a reply? reply "draft it"`. On that reply, use the Drafting path above
  (`family_inbox.py draft …`, threaded via the source Message-ID) → creates a kenyonseo@
  DRAFT for review. NEVER sends. (Non-date notes with a reply CTA — like a class volunteer
  ask — are notable on their own; text the heads-up + draft offer even with no dates.)
- If, after the relevance filter, there are NO relevant dates, no reply CTA, and nothing
  else notable → exit silently.

### 4b. Special case — "Nourishment By Katya" weekly chef invoices (Tom 2026-09-02)
QuickBooks payment-request emails from `Nourishment By Katya LLC <quickbooks@notification.intuit.com>`
(the weekly household chef) arrive as **two invoices back-to-back** (minutes apart). Handle
as ONE unit, not two separate heads-ups:
- These emails have **no plain-text part** — pull the dollar amount from the HTML body:
  `grep -oE '\$[\d,]+\.[0-9]{2}'` (first match is the invoice total) and the due date from
  subject/body context.
- Before texting, check `family_inbox.py recent 10` for a companion Katya invoice received
  within the last ~30 min that hasn't been texted yet (or was texted separately by a race).
  If both are in hand, send **ONE** combined text:
  ```
  📬 Nourishment By Katya — Weekly Invoices

  Two invoices in, both due <date>:

  • Invoice <#> — $<amt>
  • Invoice <#> — $<amt>

  Total: $<sum>
  ```
  If only one has arrived so far, it's fine to wait briefly for the companion rather than
  firing immediately — these two are reliably paired.
- **Always** (whether sent solo or combined) create an all-day reminder via eventkit, due
  the NEXT calendar day from processing (not the invoice's own due date):
  `eventkit add --title "[TS] Pay Katya Invoice" --list Kenyon-Seo --due tomorrow --notes "<breakdown + due date>"`.
  **List = Kenyon-Seo, not Tasks** (Tom 2026-09-02) — this is personal/household, not work.
- **Auto-completes itself — no manual check-off needed** (Tom 2026-09-02): a launchd
  watcher (`com.invertedcap.katya-paid-watch`, every 30 min) runs
  `check_katya_paid.py`, which reads the Katya iMessage thread (+19178224622) for a
  payment-confirmation message ("Sent", "paid", "Zelle'd", etc.) sent by **either Tom
  or Elsie** after the reminder's creation time, and if found calls `eventkit complete`
  on it + texts the family group a one-line ✅. It's a single 3-way iMessage GROUP
  (Tom, Elsie, Katya) — Elsie's replies sync to this Mac's chat.db too, so either of
  them confirming is enough. Script lives alongside this SKILL.md; the watcher plist +
  wrapper live at `~/.claude/scheduled-tasks/family-inbox/katya_paid_watch.sh`
  (machine-local, not in this repo).

### 4c. Special case — family travel bookings, one event PER LEG not per email (Tom 2026-09-02)
Root cause of a real incident: 6 separate flight-confirmation emails came in for one trip
(2 resends for Tom + 1 each for Elsie/Andy/Benny, all Korean Air), and per-email auto-add
(4b policy) created 6 fragmented, inconsistent duplicate calendar events for what were
really just 2 flights. Tom: **"assume we're traveling as a family... consolidate
everything into a single event"** — a family member booking a flight/hotel/rental
almost always means the WHOLE family is on it, even when each passenger gets their own
confirmation email.

- **Dedup key for travel = the trip segment, not the exact event/title.** Before adding
  any flight/hotel/rental confirmation, check the target calendar for an existing event
  matching the same **flight number + date + route** (flights) or **property + check-in
  date** (hotels) — NOT an exact-title match, since per-passenger emails all have
  different subject lines/passenger names for the identical segment.
- **First email for a segment → create ONE event** titled by the segment itself, not the
  passenger (`✈️ Family Flight: JFK → Seoul (ICN) — KE 082`, not `✈️ Tom: JFK → ...`).
  Description lists the flight facts once, then a running list of `<Person> — Booking
  ref: <ref> [· Seat <seat>]` lines.
- **Every subsequent email for the SAME segment → ENRICH, don't create.** Add that
  passenger's line to the description (or update their seat if it's a resend with a
  newly-assigned seat) via `update_event`. Never a second event for a flight/leg already
  on the calendar.
- **Multi-leg trips (outbound + return, or multi-city) → one event per leg**, same
  consolidation logic applied independently per leg.
- This is a specific case of the general "family activity in the household inbox implies
  the whole family" pattern noted for the NOTABLE categories — apply the same "assume
  family" default reasoning to any household-addressed booking, not just Korean Air.

### 5. Log
Append to `~/.claude/skills/family-inbox/sweep-log/YYYY-MM.log`:
`[ts] msg=<messageId> from=<from> dates_found=<n> enriched=<k> proposed=<j> texted=<y/n>`.

**Calendar access:** the processor's cold-path `claude --print` inherits the Google
Calendar MCP, so `list_events` / `update_event` work here. `SENDBLUE_API_SECRET` is
injected; the send helper reads `.sendblue_config` from the sms-listener dir. The email
body is in the args — no IMAP fetch needed.
