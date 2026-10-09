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

**The processor runs `fast.py` first; you only run when it exited 10** (2026-10-08). It already dropped the
step-0 gate DROPs, the §4b Katya invoices and plain marketing / company-announcement / review-request mail
(logging their step-5 line itself), and it sends every household, personal, Re:/Fwd:, school, thin-body or
DOC-link email straight to you. Everything below stays exactly as written – a job that reaches you is not
pre-judged NOTABLE.

**The judgment – what's worth a ping, what stays silent, act vs offer vs heads-up, alert shape –
is the shared tend-to rulebook `~/.claude/skills/shared-references/tend-to.md` (one rulebook for
every email account and texts, Tom 2026-10-02). READ IT FIRST. This lane binds only the
family-inbox transport and the lane-specific bits below.**

0. **Self-sent replies** (from `kenyonseo@gmail.com` itself) = tend-to.md §2 "our own messages are
   signal": fold what the family committed to into the matching event (the 2026-09-09 case updated
   Jeremy's-birthday with "Elsie + Andy + Benny attending, Tom not"), phrase the heads-up as
   "Updated RSVP: …", and the step 4 cooldown applies with full force.

**As code (2026-10-04) — pre-model gate, run FIRST, before any classification:**
```bash
python3 ~/.claude/skills/family-inbox/family_inbox.py gate --from "<args.from>" --subject "<args.subject>"
```
Exit 1 (`DROP <reason>`) → no calendar work, no text; log the step-5 line with `texted=n reason=<reason>`
and exit. Exit 0 (`PASS`) → continue to step 1 (PASS is "not a hard drop", not a KEEP verdict). The
gate owns the BFX / financial-aid / MS-US-K-4 hard drops in "Relevance filter FIRST" below (regexes in
`gate_decision()`; fixtures = real kenyonseo@ subjects, `tests/test_gate.py`). Line-level filtering
inside a multi-division email (the BFS Weekly) stays model judgment.

1. **Classify** the message per tend-to.md §1-§2 → **NOTABLE** (ping-worthy) or **SKIP**. Typical
   family-inbox NOTABLE senders (illustrative): Brooklyn Friends School (teachers, PTA, signups, the
   weekly digest), kid activities (Little Gym, Super Soccer Stars, Music Together, camps),
   reservations/travel, health (doctor's offices, Magnus Health, pharmacy), bills.
2. **SKIP → exit silently.** No text.

### 3. Date-extraction harness (the core of NOTABLE handling)
Many notable emails — especially the **BFS weekly** (Andy's school) — carry multiple
dated items. Extract EVERY concrete date/event/deadline (first day, picture day, no-school
days, gatherings, conferences, due dates, appointment times).

**OPEN THE LINKS *AND THE ATTACHMENTS* FIRST — the email body is not the email**
(Tom 2026-09-04 / 2026-09-06, HARD RULE). The webhook `body` arg is only the first ~2000
chars of **text/plain**, so it misses HTML-only anchors, anything past the cutoff, **and
every attachment** (the webhook never carries them). Three proven cases where the dates live
*entirely* outside the body:
- **Pink Room "Essential Info" sheet** — body had no dates; the linked Google Doc held the
  whole 2026-2027 date list **and** the 5-day phase-in schedule. Reported "no dated events
  to add" and missed all of it.
- **BFS: The Weekly** — body is literally *"To view the contents of this message, click on
  the following link"*. Every weekly digest has been classified on zero content.
- **Regal movie ticket** (Tom 2026-09-06) — body was just prices + a login-walled order link;
  the showtime/theater/auditorium/seats were printed on the **attached JPEG ticket**. Told Tom
  "no date to auto-catch" because the image went unopened. Tickets, boarding passes, and event
  confirmations put the date/time/seat in an IMAGE, not the text — just like BFS puts it behind
  a link.

So, before extracting anything, enumerate what the message actually points to:
```bash
python3 ~/.claude/skills/family-inbox/family_inbox.py links <messageId|uid>
```
This reads the FULL message off IMAP (both text/plain and text/html) and prints deduped
URLs plus attachment names, with `DOC`-tagged lines first. **If ANY `ATTACHMENT` line
appears, pull the bytes and read them** — the same reflex as clicking a BFS link:
```bash
python3 ~/.claude/skills/family-inbox/family_inbox.py attachments <messageId|uid>
```
This saves each attachment to `/tmp/family_attach/` and prints a `SAVED <path>` line per file.
Then **open every DOC line AND every saved attachment** before deciding what dates exist:
- Google Doc/Sheet/Slides → `mcp__claude_ai_Google_Drive__read_file_content` with the file
  ID from the URL (`/d/<id>/` or `?id=<id>`).
- Other web pages (myschoolapp push pages, Smore, SignUpGenius) → `WebFetch`.
- Image attachments (JPEG/PNG — tickets, passes, flyers) → `Read` the saved file (vision).
- PDF/DOCX attachments → `Read` the saved file.
- If a link or attachment won't open (permissions, login wall, fetch error), say so explicitly
  in the heads-up — **never** let an unopened link/attachment become a silent "no dates found."

**NEVER report "no dated events to add" when a DOC link OR an attachment went unopened.** That
claim is only valid once every link and every attachment has actually been read.
**As code (2026-10-04):** before logging `dates_found=0` or saying "no dated events", run
```bash
python3 ~/.claude/skills/family-inbox/family_inbox.py unopened-check <messageId> --opened <url> --opened <SAVED path> ...
```
listing every DOC URL and SAVED attachment path you actually read. Exit 1 prints `UNOPENED <item>`
lines → the heads-up must name each unopened item ("couldn't open X") instead of "no dates";
exit 0 `ALL-OPENED` → the zero-dates claim is valid.

**Relevance filter FIRST — drop anything not for this family:**
- **BFS / school mail:** Andy is in the **ECLS / Early Childhood (EC)** program, **Pink
  Room** (2s), teachers **Linda and Camille** (`pink@brooklynfriends.org`). Keep
  **EC / ECLS / ECLC / "all school"** items; **DROP Middle School (MS), Upper School (US),
  and Grades K-4** items entirely (see [[reference_family_members]]). A BFS weekly lists
  all divisions — only EC/ECLS lines are ours.
  - **DROP BFX Extended Day items — Andy is not enrolled** (Tom 2026-09-04). Trimester
    start/end dates, registration windows, dismissal-option changes: all N/A.
  - **DROP financial-aid items — not applying** (Tom 2026-09-04). The Clarity application
    deadline and related reminders are N/A.
  - *These three subject-level drops are enforced in code by `family_inbox.py gate` (above, Mode A
    step 0 gate). Extending them = edit `_GATE_DROPS` / `_OTHER_DIVISION` + add a fixture row.*
  - Adult-facing all-school events (Giving Day, The Benefit, volunteer fairs) ARE wanted,
    but add them **all-day and FREE** — they don't block the day.
- Other mail: keep only genuinely family-relevant dates.

**Calendar rules = the shared contract**
`~/.claude/skills/shared-references/calendar-event-handling.md` (this lane, the personal
lane, and the work lane share it — behavior changes land THERE first; this skill keeps only
family-lane transport). Sections used here: invariant 1 (confirmed vs invitation, CTA),
invariant 2 + the attendance exception under Known event sources (cross-calendar dedup,
never cross-copy), invariant 3 (reconcile: fill missing, correct conflicts, note old value),
invariant 4 (headers + item lines), invariants 5-6 and 9 (defaults, dash glyphs, location
format), Known event sources (kids' classes), Lane-specific (family bullet).

**For EACH relevant date, dedup-check BOTH calendars BEFORE anything else** (contract invariant 2): `python3 ~/.claude/scripts/calendar_write/calendar_write.py find-dupes both --day <YYYY-MM-DD> [--start HH:MM] [--keyword <title/venue/person word>]...` (exit 0 = no match → create · 10 = exactly one → reconcile THAT event in place, on the calendar it lives on (calendar-event-handling.md invariant 3: fill missing, correct conflicts, never a copy on the other calendar) · 11 = two+ → ask, don't guess · 2 = calendar unreachable → don't create). The BFS feed already auto-adds many milestones (Labor Day, Family Visit Day, First/Second Day), so most will already exist — judge the returned candidates semantically.

- **DUPE found → RECONCILE per contract invariant 3**, via `update_event` in place on the
  calendar where it lives. Authorized auto-update — no approval needed — but you STILL
  report it in step 4 (a correction states old → new, never hidden in a plain line).
- **NET-NEW relevant date → ADD it directly, then report it** (Tom 2026-09-02 — "take
  first pass and text alert us with what you added, let us audit/ask for edits"). No
  more propose-and-wait: once the dedup check above comes back clean, create the event
  on the Elsie-Tom calendar right away (BUSY unless it's clearly a FREE-type marker per
  the calendar fast-path rules) and put it in the heads-up as an **already-done** `Added` line
  (contract invariant 4), not a 🆕 ask. Dedup-before-add is still absolute — never skip that check just because
  adding is now automatic. If a date is genuinely ambiguous (ex: ambiguous group/time
  assignment, conflicting info in the email) hold it as a 🆕 ask instead of guessing.
- **INVITATIONS awaiting a yes/no are ALWAYS a 🆕 ask, never an auto-add** (contract
  invariant 1): a 🆕 line with when/where/RSVP deadline closing with the contract's CTA —
  the ONLY CTA on this lane. A 👍 or worded yes from either parent adds it (dedup-reconcile
  first — never double-add). Published school-calendar dates (BFS milestones, class
  events) stay auto-add — calendar facts, not invites. Kids' activity providers and
  location format: contract Known event sources + invariant 9.

### 4. Text ONE consolidated heads-up (Sendblue) — to the FAMILY GROUP

**Same-thread cooldown FIRST** (tend-to.md §2 "ping once"). Transport: grep this month's
sweep-log for a `texted=y` entry from the last 30 minutes on the same thread (subject with
`Re:`/`Fwd:` stripped):
```bash
grep -i "subject=.*<normalized subject>" ~/.claude/skills/family-inbox/sweep-log/$(date +%Y-%m).log | tail -5
```
Hit → still do the calendar work (step 3), text only a genuine delta, else log
`texted=n reason=thread-cooldown`.
(Load prefs first — `python3 ~/.claude/skills/sms-listener/prefs.py load core email` — and
honor them, esp. the header/formatting rules.)
Kid/family stuff goes to the **family group** (Assistant + Tom + Elsie) so both parents
see it — NOT Tom's 1:1:
```bash
FAMILY_GROUP_ID="$(cat ~/.claude/skills/sms-listener/.family_group_id 2>/dev/null)"   # not GID: read-only in zsh
if [ -n "$FAMILY_GROUP_ID" ]; then
  ~/.claude/skills/sms-listener/send_imessage.sh --group "$FAMILY_GROUP_ID" "<heads-up>"
else
  ~/.claude/skills/sms-listener/send_imessage.sh "+12012567714" "<heads-up>"   # fallback: Tom 1:1
fi
```
Calendar heads-ups use the contract's header vocabulary and item lines (invariant 4 —
`📅 Added/Edited/Enriched/Removed`, most-significant-kind heads a mixed run, correction
lines state old → new) plus this lane's `✓ Already on cal` acknowledgment (contract
Lane-specific). Non-calendar notable mail keeps a `📬 <Topic>:` headline. Example:
```
📅 Added: Picture Day Tue 10/7 (EC)

✓ Added – Picture Day Tue 10/7 (EC)
✓ Edited – Family Visit Day Tue 9/8 – was 10:15, sign-up sheet says 10:00-10:15
✓ Enriched – Grown-Up Gathering Fri 9/11 – added 8:45am start
✓ Already on cal – First Day of School Wed 9/9
📸 Photos: <link>
```
- `📸` line = **class-recap photos link (Tom 2026-09-18, HARD RULE).** A *classroom* recap
  newsletter — the Pink Room "Newsletter" from `pink@brooklynfriends.org` (Linda/Camille),
  and any future class/activity recap — will **almost always** carry a link to *that week's
  photos* (a Google Photos / SmugMug / class-app album; confirmed 2026-09-18 "first Pink
  Room Newsletter" → `https://photos.app.goo.gl/...`). When enumerating the message's links
  in step 3, spot that photo-album anchor and put it in the heads-up as a `📸 Photos:
  <clickable link>` line. It's the parents' favorite part — never drop it. If a
  recap genuinely has no photo link this time, just omit the line (don't announce its
  absence). This is a link-alert only — never treat the album as a dated event.
  **NOT "BFS: The Weekly"** — that all-school click-through shell is a different email and
  is *not* the photo carrier; the album lives in the teacher's classroom recap.
- `Added` lines are first-pass, not a proposal — Tom/Elsie audit after the fact; a reply
  like "move picture day to 9am" / "remove that" / "wrong calendar" is an edit request,
  handled via the normal sms-listener calendar fast path against the event you just
  created (you know its id from this same turn). Only fall back to a 🆕 **ask** (ends with
  a question) for a date that's genuinely ambiguous and you don't want to guess wrong.
- **Draft-offer CTAs are RETIRED on this lane (Tom 2026-09-21: "i dont think we need a
  draft as a CTA for family bot related stuff").** Never append a `reply "draft it"` /
  "Want me to draft a reply?" line to family-group heads-ups — even when the email's
  call-to-action is responding (RSVP, "reply if interested", sign-up, confirm attendance).
  The Drafting path itself stays available: a WORDED request from either parent ("draft
  it", "draft a reply saying …") still uses `family_inbox.py draft …` (threaded via the
  source Message-ID) → creates a kenyonseo@ DRAFT for review, NEVER sends. (Non-date
  notes that are genuinely notable — like a class volunteer ask — still get the heads-up,
  just with no draft-offer line.)
- **CTA = a task/call outside email → offer to add a reminder.** If the notable email's
  call-to-action requires Tom/Elsie to actually DO something outside the email thread
  (call a number, show up somewhere, collect/hand over an item) rather than just reply,
  close the heads-up with an offer to add a reminder — e.g. append a line like
  `Want me to add a reminder to call them? reply "remind me"`. On that reply, create it
  via the `add-reminder` skill (native Apple Reminder, not eventkit-freeform). This is
  separate from the reply-CTA bullet above — a message can carry both action shapes
  (draft-reply AND a call-back), in which case offer both CTAs. (Tom, 2026-09-10 — a
  Little Gym "call to collect payment info" alert offered only "draft it" with no
  reminder offer.)
- If, after the relevance filter, there are NO relevant dates, no reply CTA, no task CTA,
  and nothing else notable → exit silently.

### 4b. Special case — "Nourishment By Katya" weekly chef invoices — SKIP, owned elsewhere
QuickBooks payment-request emails from `Nourishment By Katya LLC <quickbooks@notification.intuit.com>`
(the weekly household chef) are **fully owned by the dedicated code watcher**
`~/.claude/scheduled-tasks/family-inbox/katya_invoice_watch.sh` → `check_katya_invoice.py`
(every 5 min, built 2026-09-03). It aggregates paired invoices into ONE
`[TS] Pay Katya Invoice: $<total>` reminder on Kenyon-Seo and posts the one-line
family-group alert itself. The watcher reads EVERY Katya QuickBooks email, including
the "REMINDER from Nourishment By Katya LLC about your … invoice" format, which has no
invoice number. It flags off-pattern batches with ⚠️ in both the title and the text: more
than 2 invoices, a total over $599, or a no-number email. Tom 2026-09-24: a third
$172.53 REMINDER email was silently dropped, so the reminder said $531.33 instead of $703.86.
- **Thread reconcile** (Tom 2026-09-24, "follow the thread"): the 5-min paid-watch also
  compares Katya's "Total is $X" text with the reminder total. A mismatch sends ONE ⚠️
  family-group flag and records it in the notes, naming the likely odd invoice when exactly
  one subset of invoices adds up to her total. Only her explicit "ignore the one for $X" /
  "it was a mistake" drops an invoice and re-titles the reminder, with a ⏰ Reminder
  Updated text.
- **This triage agent must NOT act on these emails at all** — no reminder, no text, no
  calendar entry. Just skip them silently. (2026-09-11 incident: this section's old
  per-email "always create a reminder" instruction ran once per invoice email on top of
  the watcher's own create — three reminders and three texts for one invoice pair. The
  alert body also picked up Zelle/card-fee commentary Tom doesn't want; the watcher's
  one-liner is the only sanctioned message.)
- **Auto-completes itself — no manual check-off needed** (Tom 2026-09-02): the
  meal-prep-reschedule 5-min sweep (`com.tomseo.scheduled.meal-prep-reschedule`)
  piggybacks `check_katya_paid.py` on every tick (Tom 2026-09-11 — the standalone
  `com.invertedcap.katya-paid-watch` job is retired to `_disabled-plists`). It reads
  the Katya 3-way thread (+19178224622) for a payment-confirmation message ("Sent",
  "paid", "Zelle'd", etc.) AND the KSeo Bot family thread (hard payment words only —
  "paid"/"zelled"/"venmoed" — and only when it's about Katya: names Katya, an invoice #,
  or the amount, or is a bare "paid" whose latest bot message was the Katya alert;
  "paid Lupe" never counts), from **either Tom or Elsie** after the reminder's
  creation time, and if found calls `eventkit complete` + texts the family group a
  one-line ✅. Elsie's replies sync to this Mac's chat.db too, so either of them
  confirming is enough. Script lives alongside this SKILL.md; the wrapper lives at
  `~/.claude/scheduled-tasks/family-inbox/katya_paid_watch.sh` (machine-local, not
  in this repo).

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
  passenger (`✈️ Family Flight: JFK → Seoul (ICN) – KE 082`, not `✈️ Tom: JFK → ...`).
  Description lists the flight facts once, then a running list of `<Person> – Booking
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
`[ts] msg=<messageId> from=<from> subject="<subject>" dates_found=<n> enriched=<k> proposed=<j> texted=<y/n>`.
**`subject=` is mandatory on every line** — the step-4 same-thread cooldown greps the
log by subject; a line without it makes the next job on the same thread double-text.

**Calendar access:** the processor's cold-path `claude --print` inherits the Google
Calendar MCP, so `list_events` / `update_event` work here. `SENDBLUE_API_SECRET` is
injected; the send helper reads `.sendblue_config` from the sms-listener dir. The email
body is in the args — no IMAP fetch needed.
