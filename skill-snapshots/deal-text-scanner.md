---
name: deal-text-scanner
description: |-
  Text-channel router for Tom's personal iMessages. A code-gated launchd sweep hands it new 1:1 (and small-group) messages; it triages each into one or more lanes and loads only that lane's reference file. PORTCO ASK — a portfolio founder asking for people in a category → 🙋 card (shared-references/portco-asks.md). DEAL — deal-flow signals become a 🆕 Opportunity card texted to Tom; his 👍 (via sms-listener) does the add, never a direct CRM write. INTRO — Tom's in-thread replies move pipeline status, unknown numbers get identified as intro'd founders, the Blockit handoff is pre-staged, and the reverse arrow (opt-ins/declines to Tom's own portco-intro offers) moves relations and queues the connect draft. FEEDBACK — backchannel asks and debriefs (incl. voice notes) are auto-written to the Notion feedback notes with no 👍 gate; the email side stays with feedback-outreach-scanner. ADDRESS — a home address texted in a thread → 🏠 card; 👍 saves it to Apple contacts. Scheduled-sweep-only. TOM-ONLY surface — never family-scoped. Most messages are NOT anything — default to silence.

---

# Text Scanner — lane router

> Named `deal-text-scanner` for historical reasons (launchd plist, `sms-listener` paths, and
> the live state files all key off that name). It is no longer deal-only — it routes five
> lanes. Rename only as a deliberate, separately-verified change.

## Input

Args: `{mode:"scan", messages:[{rowid, ts, sender, sender_name, chat, participants, from_me,
text, attachments, attachment_paths}]}` — new messages since the last sweep, pre-filtered in
code by `sweep.sh` (1:1 threads plus ≤4-member groups; family, short codes, and Tom's own
agent number already excluded).

A **`reconcile:true`** flag marks a daily backfill (`reconcile.sh`) — a re-scan of the last 36h
that catches any batch whose classify job timed out and was dropped past the watermark. Treat it
EXACTLY like a normal scan: the per-lane dedup ledgers (`.proposed` etc.) make re-seeing an
already-processed message harmless (no duplicate card/alert). Only caveat — these rows may be up
to 36h old, so weight anything already-resolved or stale toward silence.

`sender` = the thread's peer handle. **`from_me:1` rows are TOM'S OWN messages** — never
candidates in the deal lane, but load-bearing everywhere else: they carry his opt-in/pass
(intro lane) and his outbound asks (feedback lane).

**Use the peer's real name — it is already resolved for you.** `sender_name` is the contact
name `sweep.sh` resolved from AddressBook at enqueue time (via `resolve_contacts.sh`, which
reads the AddressBook DB from the FDA-granted launchd context — the headless MCP cannot,
which is why referrers used to show as a raw number). **Prefer `sender_name` whenever it is
non-null** — that is the referrer/peer name for every card. Only when it is null: try
`mcp__imessages__tool_find_contact` on the handle, or an in-thread LinkedIn preview title.
Fall back to the raw handle only if all of those fail — and never put a raw phone number in
anything Tom reads.

`attachments` = display names, `attachment_paths` = local paths under
`~/Library/Messages/Attachments/`. A PDF is usually a deck (deal lane); an audio file
(`.caf`/`.m4a`/`.amr`) is usually a voice note (feedback lane). Read attachments locally,
read-only — never send one anywhere.

## Triage — which lane(s)

Read each thread as a conversation, not message-by-message. **A thread can hit more than one
lane; classify each independently and never let one lane's silence suppress another.**
(Canonical: Byron Edwards' thread carried a Redwagon feedback arrangement *and* his own raise
— "90 yard line starting ai lab / chem biz… We'll need like $8-10m" — which is a live
`-1`/NewCo signal.)

**Step 0 — roster checks: ALREADY RUN IN CODE before the batch reached you (2026-10-05).** `sweep.sh` /
`reconcile.sh` pipe every batch through `roster_precheck.py`, so each message carries `roster_checked: true` and, on a
hit, a `roster` object: `pending_feedback` or `prior_feedback_note` → you MUST load `references/feedback-lane.md`;
`intro_pending` → you MUST load `references/intro-lane.md` (§5b) — whatever the message text says; `portco_founder` → you MUST load `references/intro-lane.md` (§5c) and run its step 1 check, AND load `references/portco-ask-lane.md` (2026-10-06). Do not re-query.
**Only if a message lacks `roster_checked`** (the precheck failed open — Notion was unreachable at sweep time) do you
run the three checks below by hand. The prose below is the WHY and that fallback. The original rule: for EVERY
thread with an inbound peer, run THREE cheap deterministic checks: (1) query the Opportunities
DB for rows with a non-empty `📣 Pending Feedback` and resolve whether this sender is on that
roster (People row → phone vs. the thread handle); (2) query the Notes DB
(`collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) for a feedback note titled with this
sender's name (`Name LIKE '<Sender Full Name>%'`, giver-first title convention) on a live Opp
— the roster empties after the FIRST piece of feedback lands (Tom, 2026-09-10: pending = first
feedback only, never re-added), so a follow-up debrief (the second call, a later text) is
invisible to the roster and is caught ONLY by the prior-note check; it appends a new dated
Response block to that same note. And (3) resolve the sender to a People
row (via `sender_name` / handle) and check whether it sits in ANY Active-Portfolio Opp's
`👓 Intros (Qualified)` or `☎️ Intros (Outreach)` relations — a hit FORCES loading
`references/intro-lane.md` (§5b): their text may be the yes/no to Tom's own intro offer, and
content alone can't tell you (the canonical miss was a bare "Of course!" — Zack Parker/Tuor,
2026-09-17). **A hit on check (1) or (2) FORCES loading
`references/feedback-lane.md`, no matter what the messages say.** A person with an open ask texting Tom substantively IS presumptively
the debrief — debriefs routinely name no company (the connective "I spoke to your red wagon
guy" may have landed hours earlier in a different sweep batch). This check is NOT optional and
NOT content-gated: the 2026-09-10 miss happened exactly here — the 22:30 sweep held Byron
Edwards' full Redwagon read ("I can see the problem, we don't have it specifically…"), even
summarized it as him evaluating a founder's business, then concluded "no feedback signals"
because no one asked the roster. The 16:35 sweep the same day roster-matched him and classified
correctly. Content can't tell you someone owes Tom a read — only the roster can.

| Signal in the thread | Lane | Load |
|---|---|---|
| Intro offer to a founder; a company + round details; a deck/LinkedIn with a pitch | **Deal** | `references/deal-lane.md` |
| Tom replying opt-in/pass on a deal; an unknown number that may be an intro'd founder; a REFERRER announcing an intro is live ("meet / re-meet X", "you two connect", "I'll let you find time"); an email or scheduling exchange on a known deal | **Intro** | `references/intro-lane.md` |
| Tom OFFERING one of his portfolio founders to the peer ("can I intro you to…" + portco/founder link); or ANY inbound that could be a yes/no to such an offer — `.portco_intro_asks` hit, OR the sender resolves to a People row on any Active-Portfolio Opp's `👓 Qualified`/`☎️ Outreach` roster (the reply may be a bare "Of course!") | **Intro** | `references/intro-lane.md` |
| **Step 0 roster hit (mandatory check above)**; or Tom asks someone for a read | **Feedback** | `references/feedback-lane.md` |
| A portfolio founder (`roster.portco_founder`) asks Tom for people in a category – "do you know any lenders…", "anyone in your network who…" | **Portco Ask** | `references/portco-ask-lane.md` |
| The peer texts a street address that is their (or someone's) HOME, e.g. answering Tom's "what's your address?" | **Address** | `references/address-lane.md` |
| Anything else | none | — exit silently |

**The bar is high and silence is the default.** Not lanes (except a home address, Address lane): scheduling chatter, social talk,
thank-yous, LP/fund-admin, a portfolio company's own news / metrics, favor-forwards, news links without a referral,
anything from a service number. When unsure → silent. False negatives are fine (Tom sees his
own texts); false positives erode trust.

**Founders' texts go through every lane like anyone else's (Tom 2026-10-06: "they should be a part of all the
texts that come in that you're listening to").** The one thing their threads never produce is a NEW-deal signal
about their OWN company – it already has an Opp (opp_dedup.py catches it; Erik / Rengo 2026-09-24 was fundraising
color logged as a 📈 card). Asks for people → Portco Ask lane; intro yes/no → Intro lane; reads on other companies →
Feedback lane; an address → Address lane; a third company they pitch or refer → Deal lane.

**Feedback-lane detection is the one exception to keyword matching.** Do not look for the
word "feedback" — the corpus shows it is usually absent and the ask rides inside an intro
offer. Judge against `~/.claude/skills/shared-references/feedback-ask-signals.md`.

## Gates that apply to every lane

- **Dedup before writing — in code (as code 2026-10-04).** Each lane owns its own ledger, read and written ONLY
  through `python3 ~/.claude/skills/deal-text-scanner/lane_ledger.py has|add|prune --lane deal|feedback|intro|handles [--rowid R] --key K` (exit 0 hit / 1 miss on `has`;
  `add` exit 3 = already there, nothing written; never grep or hand-append): `.proposed` (deal, key + rowid),
  `.feedback_logged` (feedback, rowid membership — and sweep.sh / reconcile.sh DROP already-logged rowids before you
  see them), `sms-listener/.home_address_proposed` (address, checked in code). The intro lane's real dedup is the
  live Opp status (moves are idempotent, never a downgrade); `.expected_intros` and `.known_handles` are OPTIONAL
  caches (`--lane intro` / `--lane handles`; prune intro lines once Connected). `.known_handles` was cited as a dedup
  ledger here and in reconcile.sh but never existed — nothing wrote it; `lane_ledger.py add --lane handles` now
  creates it on first use, and nothing depends on it. Tests: `tests/test_lane_ledger.py`.
- **Never write to the CRM from the deal lane** — it proposes; the confirm loop in
  `sms-listener` writes. The feedback lane DOES write directly (see its file for why).
- **Never message the family group** from this skill.
- **Never put a raw handle** where a name belongs.
- **Every alert follows the canonical alert convention**
  (`~/.claude/skills/send-alert/references/alert-convention.md`): one domain emoji + Title Case
  `Headline: Subject` (colon, never a dash), state glyph (→/✓/⚠) inline on its own line, no
  date suffix on single events. These are iMessage texts, so render the grammar's SHAPE in
  plain text — NOT the Slack `<u>`/`**`/`[label](url)` markup — with the Opp URL as a raw
  footer line. Applies to every net-new alert, no exceptions.

## Exit

No lane fires → exit silently. No text, no alert, no log noise.
