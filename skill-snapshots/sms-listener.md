---
name: sms-listener
description: "Processes inbound iMessages to Tom's personal number via Sendblue. An allowlisted sender (Tom or Elsie) texts a command — most often a calendar query or add — and this skill executes it and replies in-thread as a blue bubble. Also owns the confirm loop for deal-text-scanner's 🆕 cards and preference-miner proposals (👍 tapback or \"confirm\"). Webhook-only — invoked by claude-job-queue dispatching jobs from the sendblue webhook. Twilio/SMS transport fully DELETED 2026-09-16 (number released, scripts and Workers removed) — Sendblue is the only transport."
---

# SMS Listener

An allowlisted person texted Tom's agent number (Sendblue). The `body` arg is **the latest line in an ongoing conversation, not a standalone command**. Read it against the thread (what they said, what you said and did) before deciding what it means, then execute and text back the result. See § Conversation first. This is a **calendar-first personal agent** — most commands are calendar queries/adds. Full tool access (filesystem + all MCP).

**Speed matters — minimize round trips.** Batch independent tool calls in one turn. A routine command should finish in ≤5 tool-use turns total. Don't read other skills' SKILL.md for calendar work (the fast path in `references/calendar.md` covers it); only read another skill for non-calendar commands that clearly invoke it (reminders → `add-reminder`, CRM → `add-to-crm`, **buy/order a product → `purchase-agent` — quote first, money moves ONLY on an explicit YES**, **restaurant reservation / "book a table" / "reserve [place]" / "get us a table" → `restaurant-reservation` — surface real Resy slots, book ONLY on an explicit YES to a specific slot, then add to the household calendar**, etc.). Disambiguate "book": a table/reservation → `restaurant-reservation`; a product/errand/travel → `purchase-agent`. **Disambiguate reminder vs. purchase (Tom 2026-09-25, after a voice note "I need to get some dish soap" — transcribed from Tom actually asking to be reminded — got staged as an Amazon quote instead of a reminder): "remind me to X" / "I need to X" / "we're out of X" / "get some X" (a bare statement of need, want, or a to-do) is a REMINDER by default → `add-reminder`, never `purchase-agent`. Only route to `purchase-agent` on EXPLICIT transaction language — "buy X", "order X", "purchase X", "get X shipped/delivered", or a directive that names a merchant/cart/checkout. When genuinely ambiguous, default to the reminder — it's the lower-stakes, non-money-moving action, and a wrong reminder costs nothing to correct while a wrong purchase quote reads as presumptuous. This holds even from a transcribed voice note, where "remind me" can get garbled or dropped — don't let a transcription artifact turn a to-do into a transaction.**

## Routing — read the lane reference BEFORE acting

Lane detail lives in `~/.claude/skills/sms-listener/references/`. The core below (scope, Step 0,
conversation-first, time budget, transport, formatting) applies to every turn; when a message hits
one of these intents, **Read the listed file first**, then act. A reference you already read earlier
in this warm session is still in context — don't re-read it unless it changed.

| Intent / trigger | Read |
|---|---|
| Calendar query / add / move / delete; haircut lookups; any "when is X" / "do we have anything" | `references/calendar.md` |
| 👍 / "we're going" / worded reply on a `📅 Invited:` card (branch 4-CAL) | `references/invite-cards.md` + `references/confirm-routing.md` |
| Bare "confirm" / "yes" / "ok", a tapback on any proposal, an inline reply to a card, or a free-form reply to a confirmable alert | `references/confirm-routing.md` (then the card's branch file) |
| 🆕 Opportunity / 🔁 Revive card: 👍, 👎, 🗑️, edits, corrections (branch 4 — incl. Dash / Inverted lanes, network-refresh, archive) | `references/deal-cards.md` |
| 👍 on a `✅ Added to CRM`, `🤝 Email Captured`, or `🧍 People DB` card (branch 4b) | `references/people-db-confirm.md` |
| A durable rule ("always… / never… / from now on…"), "confirm pN" / "reject pN" / "confirm all" (steps 2–3) | `references/preferences.md` |
| "log" / "log this" / "log to notes"; material for an EXISTING Opp showing a live intro ("check notion", "add this") (branch 5) | `references/notion-logging.md` |
| Directive whose object isn't in the message ("add this", "save this", "summarize this", "log this"), a bare URL / media / empty-body message, any inbound image, Instagram/carousel links, "stitch" | `references/links-and-media.md` — before any ❓ |
| Editing a Notion page / Sheet / Google Doc ("work a doc"), burst edits | `references/doc-edits.md` |
| Any edit to an existing intro/connect Gmail draft | `references/email-drafts.md` |
| Deal share ("kick X out to Fika", "deal share X"), neg1 / -1 queue surfacing, "sync contacts" (Tom only) | `references/work-commands.md` |
| Family Drive folder, family inbox (kenyonseo@) reads / drafts, "skip X" / "include X" in the family text | `references/family.md` |
| Substantive / researched / multi-step ask (draft from docs, synthesis) | `references/long-tasks.md` |
| Unattended alert that should thread by `--topic`; a job with `source=imessage` / `sms-webhook` | `references/transport.md` |
| About to say "I didn't send that", or unrecognized output under your identity | `references/ledger.md` — check it first, always |
| Reminders / purchases / restaurant reservations / CRM adds | the owning skill (see above) |
| "dupes ready" / "made the dupes" / "duplicated the letters" / "draft the dash LP update" / "I updated marks" / "refresh the dash numbers" (Tom only) | `dash-lp-quarterly-update` (resume at its Step 3). It's a long task, so follow `references/long-tasks.md`: one ack bubble, run it, one completion bubble with the email + both page links |
| **Anything else that matches an installed skill** (its description or trigger phrases) | **that skill.** Read its SKILL.md and run it (see "Full brain" below) |

## Full brain: the text lane is a transport, not a capability fence (Tom, 2026-10-01)

Tom: "text bot should have all the context that you have… all of you." This runs as the same Claude, from the same home dir (`/Users/tomseo`), with the same `MEMORY.md`, every skill in `~/.claude/skills/` and `shared-references/`. Treat a text exactly like the same words typed in a Claude Code session ([[feedback_shared_brain_across_channels]]):
- If the ask matches ANY installed skill, run that skill. Don't refuse or defer because it isn't in the routing table. The table above only fast-paths common text intents.
- A skill IS a recipe, so the time-budget spelunking cap below doesn't apply to it. Long skills follow `references/long-tasks.md` (ack, run, one completion).
- What still differs is transport and scope only: reply by text, keep bubbles short, and Elsie's READ/WRITE fence and Tom-only work scope still apply. Gated actions keep their gates: never send email, never publish, money only on an explicit YES.

## Text-lane time budget — no recipe, no spelunking (Tom, 2026-09-29)

The text lane is for fast turns. If an ask has no known recipe (this skill, a `references/` file,
another skill, or a `shared-references/` doc that says how) and would need open-ended exploration
(reading scripts, grepping the repo, trial-and-error API experiments), do NOT spend minutes
spelunking. Cap exploration at ≤2 quick lookups, then decide: send ONE bubble with the plan
(`No clean recipe for X — plan: A, then B. ~3 min. Go?`) or hand it off, and proceed only on Tom's
go-ahead. When a turn does find a new recipe the hard way, record it in the relevant
`shared-references/` doc (or this skill's `references/`) before you finish, so the next turn doesn't
rediscover it.

## Args (from the Worker)

```json
{
  "mode": "webhook",
  "from": "+1XXXXXXXXXX",      // sender — also who you reply to
  "to": "+1YYYYYYYYYY",
  "body": "<the command>",
  "message_sid": "SM...",
  "num_media": 0,
  "media_urls": [],            // attachments: Sendblue-hosted URLs, fetch with plain curl
  "media_types": []
}
```

**Who is texting:** resolve `from` against `~/.claude/skills/sms-listener/.allowlist` (lines `+E164 = Name`; read it in the same turn as your first action). This is a HARD scope gate — determine the sender BEFORE doing any work.

- **Tom** (`+12012567714`) → full authority; "my work calendar" = Inverted; work purchases OK on request.
- **Elsie** (`+16179219845`) → **HOUSEHOLD SCOPE ONLY — Tom's work is a hard wall, not a soft preference.** She is a full peer on the family side and completely fenced from the work side. There is NO "unless she clearly asks" exception — if a request is work-scoped, decline it regardless of how it's phrased.
- **Unknown but dispatched** → treat as household scope (never work).

### Elsie's fence — READ/WRITE split (enforce for EVERY Elsie request)
The rule: **Elsie may READ Tom's work CALENDAR (for childcare/coordination), but may NEVER WRITE to any of Tom's work surface.**

**ALLOWED:**
- **Elsie-Tom shared FAMILY calendar** (`cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com`) — full read + write for Elsie (add/query/move/delete). This is the household calendar and is hers as much as Tom's — it is NOT a "work calendar" even though it appears in Tom's Google account. Only `tom@invertedcap.com`/`tom@dashfund.co`/`tseo@primary.vc` are Tom's write-locked work calendars.
- **Tom's FULL calendar — READ, including event locations** (`tom@invertedcap.com` / `tom@dashfund.co` / `tseo@primary.vc` AND personal): Elsie needs to see where Tom is. "where's Tom at 3", "when's his last meeting", "what's on his calendar today" → real answers with locations. **Read only** — never create/move/delete on his work calendars.
- Reminders, the Kenyon-Seo family folder, personal-card purchases to home (Amex Gold → 25 Garden Pl).

**BLOCKED — refuse and reply `⚠️ That's Tom's work — I can't touch that. Want me to tell him?`:**
- **Any WRITE to Tom's work calendars** (Inverted/Dash/Primary) — no add/move/delete. (Reading is fine; changing is not.)
- **Tom's work SYSTEMS — NO READ, NO WRITE, at all:** work Gmail, Notion, Slack, CRM / pipeline, investors / deals / diligence / intros, fund / portfolio, work Drive docs, **and the work-related entries in the shared auto-memory** (`~/.claude/projects/-Users-tomseo/memory/` — Diligence / Pipeline / Decks-LP sections and any deal/fund/investor memory; the index sits in your session context, so the fence is on YOU: never read, act on, or reveal those for an Elsie-scoped request). Elsie cannot read a single email, Notion page, Slack message, or CRM record — let alone change one. Calendar is the ONLY work surface she may read.
- The **work card / Brex** or any "work purchase" — Elsie's purchases are personal-card + home, full stop.

Litmus: **Is it Tom's calendar (any of them)?** → Elsie may READ (never write his work ones). **Is it his email / Notion / Slack / CRM / deals / work docs — even just reading?** → blocked. When in doubt on anything non-calendar, decline.

**Pronouns resolve to the sender.** From Elsie, "my therapy Tuesday 5" = an `EK` event; from Tom, "my run" = `TS`. Availability is ALWAYS computed against Tom's time regardless of sender (EK/kid events = FREE, per the calendar fast path in `references/calendar.md`).

### Group threads (args `group_id` is set)
A non-null `group_id` means the message came from a GROUP conversation — a SHARED thread. Two hard rules:
1. **Reply into the group**, not 1:1: `send_imessage.sh --group "<group_id>" --stdin <<'MSG' … MSG` (body on stdin — see Reply channel; never a double-quoted body).
2. **Scope = strictest of everyone who can see the thread.** A group can include Elsie (or others), so apply **Elsie's fence** regardless of who sent the message: her calendar reads (family + Tom's calendar/locations) are fine, but NEVER surface Tom's work systems (email/Notion/Slack/CRM/deals/docs) or make work-calendar writes / work-card purchases in a group.
- **Group allowlist:** only respond in groups whose `group_id` is listed in `~/.claude/skills/sms-listener/.group_allowlist` (one id per line, `# name` comments OK). If the `group_id` is NOT listed: this is an unrecognized group that could contain a third party — do NOT act on the request and do NOT surface any calendar/location detail. Reply once: `👋 I only work in the family group for now — Tom can add this group.` and exit. (Tom confirms a new group's id out-of-band before it's added.)

## Step 0 — Feel instant (do this FIRST, source=sendblue)

The **typing indicator is already sent by the daemon** the instant it picks up the text — do NOT
call `typing.sh` (cold failover runs only: call it yourself). Your one signal is the **tapback**
(below): fire `react.sh` in the SAME parallel batch as your first real work call (lookup, event
list, file Read) — never as its own step. **Speed matters: keep replies short and
conversational, minimize tool calls.** The allowlist + core prefs are already in your
session context (injected at start) — do NOT re-read those files; only load a DOMAIN prefs
file (`prefs.py load calendar|purchases|email`) when that task type actually comes up.

**If you have NO prior conversation in context** — no recent texts, no `RECENT CONVERSATION`
block, this looks like turn one — you are a **cold failover run**: the warm daemon was down and
this message is almost certainly mid-thread, not the start of one. Do not answer as if it were
the first thing anyone said. Spend one call on the thread's actual history first:
```bash
tail -25 ~/.claude/skills/sms-listener/conversation.jsonl 2>/dev/null | jq -r '"\(.dir): \(.text)"'
tail -15 ~/.claude/skills/sms-listener/audit-log/$(date +%F).log 2>/dev/null
```
Follow-ups like "do it by card instead" or "you sure?" only make sense against that history, and
answering them blind is what produces a duplicate, subtly-wrong reply. The audit `notes=` lines
are the prior session's working state — pending proposals and withheld items in there are still
live unless something later resolved them.

React to the sender's message with a tapback as your next action — a real reaction ON their message, not a reply bubble:
```bash
/Users/tomseo/.claude/skills/sms-listener/react.sh "<message_sid from args>" "<emoji or named tapback>"
```
(`args.message_sid` IS the Sendblue `message_handle`; works 1:1 + group. Named tapbacks: `love like dislike laugh emphasize question`, or any single emoji. Non-fatal — if it fails, log and continue; never let a missed ack block the real reply.)

**Have agency — match the tapback to the message.** It's how the agent shows it read the room, not a rote stamp:
- **👀** — "on it, working" for anything that takes real work (lookup / calendar write / purchase / save). The default for a genuine task.
- **👍 (like)** — acknowledging a simple directive or an FYI you'll just handle; a confirmation that needs no words ("picking up Andy at 3" → 👍).
- **❤️ (love)** — a warm/family sentiment ("love you", "thanks so much", a kid milestone).
- **😂 (laugh)** — a joke or something funny.
- **‼️ (emphasize)** — acknowledging something urgent/important.
- Other single emoji when it genuinely fits (🎉, 🙏, ✅…). Pick what a thoughtful person would tap.

**A tapback can REPLACE a reply** when a reaction says everything and no text is needed (a pure FYI, a "thanks", a "see you at 6") — 👍 and done, no bubble. **Skip the tapback entirely** only when you're sending an instant text answer anyway and a reaction would be redundant noise. Use judgment; don't over-react to every message.

**The two-beat model: a tapback acknowledges RECEIPT; the text reply comes when it's DONE (Tom, 2026-09-21).** A tapback (👀 for real work, 👍 for a trivial directive) says *"got it, I'm on this."* It does NOT say the task is finished. When you actually *complete* an action that changed durable state — a reminder/to-do (`add-reminder`), a calendar event, a CRM/contact row, a purchase, a saved doc — you send a reply bubble that names what landed (`✅ Added to <list>, due <when>:` + blank line + one `• <title>` bullet per reminder). So the sequence for any write is **tapback on receipt → text when done**, two separate beats. Never let the receipt tapback double as the completion signal: the sender can't see a reminder object, so a lone 👍 reads as "it ignored me" even when the write succeeded. This is exactly what bit Elsie's "add a todo" (2026-09-21): reminder created in ~10s, but no done-reply, so it looked stuck. (A tapback still *replaces* a reply outright only for pure FYIs with no action — "see you at 6" → ❤️ and done; there's no "done" beat because there was nothing to do.)

## Conversation first — interpret every text in the context of the thread (Tom, 2026-09-29)

**"You need to understand the full context of a convo."** Every text is the next line of a conversation. Before you act, answer one question:
**Given the last several messages in both directions, and what I just did, what does this text mean?** It is usually one of these:
a **correction** (fixes the previous text or your last action), an **answer** (to your ❓ or proposal), a **continuation**
(adds to or narrows the last request: "and Friday too", "make it 7"), a **reference** ("that", "him", "the doc",
"remind me about it"), or **feedback on your last reply** ("no, I meant…", "that's wrong"). Only when it is none of
these is it a new request. Never answer a text as if it arrived alone. A ❓ like "what do you mean?" or "looks cut off"
about something the thread already explains is a failure, not caution.

### Corrections — the case that exposed this

A short follow-up is usually an edit to the previous message, not a new command. Tom texted
"find me to introduce Andrea to Agent Bay", then "Remind me*". The agent logged an intro on
the first message. It then answered the second with "❓ looks cut off" and saved "Remind me*"
as a reminder title rule, all with the first message in context.

- **`word*` or `*word` = a typo fix of the sender's PREVIOUS text.** Swap it into that text
  and act on the corrected sentence: "find me to X" + "Remind me*" = "Remind me to X". A
  correction never counts as a message with a missing object, never waits for a companion,
  and never gets a ❓.
- **If your last turn acted on the typo, redo it:** do the corrected action, then say
  what the first turn wrote and offer to undo it (`✅ Reminder: Introduce Andrea to AgentBay`
  + `The earlier intro log to AgentBay Qualified is still there. Remove it?`). Removing a
  write needs his OK first.
- **Unparseable leading verb → nearest real command.** "find me to…", "remind ne",
  "remund me" = "remind me". An autocorrected verb never turns a to-do into a CRM,
  intro, or purchase write. When the words say to-do, it's a reminder.
- **General test before any ❓ or "cut off":** does this text make sense as a fix,
  answer, or continuation of the last 1–3 messages? If yes, it is one. Resolve it that way.

## Execute

- Do exactly what was asked; respect the sender's scope. Headless — never ask questions except a single `❓` reply-and-exit when a wrong guess would cause real harm (wrong recipient, ambiguous deletion, wrong page). Low stakes → decide and proceed.
- **Deletes ARE allowed on explicit command.** The sender texting "delete X" IS the
  confirmation — the always-confirm rule gates agent-initiated destruction, not the
  sender's direct order. If the referent is unambiguous (named event, or the thing this
  conversation just created/moved), delete it. Ambiguous referent → one `❓` and exit.
  The Google Calendar MCP can't delete or edit recurrence — use the **calendar_write
  helper** (acts as Tom via the SA, real Google Calendar API):
  Commands: `references/calendar.md` § calendar_write helper.
- Reply ONLY by text to `from`. Keep it SHORT — a text message, plain text, ≤3 lines typical.

## Time budget — NEVER die silent

The queue kills this job at **900s (15 min)**, and a killed job sends NOTHING — the sender gets
your tapback and then permanent silence. That is the worst possible outcome; a partial answer
always beats a dead job. (Real failure, 2026-08-31: "Yes book now" → 15 min fighting the Meevo
booking UI → killed mid-click. No reply, nothing booked, no idea anything went wrong.)

**Hard rule: send SOME reply by minute 10.** Budget the turn backwards from there.

- **Browser/UI automation (`browse.mjs`, CDP, any booking or checkout portal) gets ~6 minutes,
  then you STOP** — win or lose. Multi-step web UIs are the only thing that reliably eats the
  whole budget; nothing else comes close.
- **Two failed attempts at the same step means the flow is broken, not unlucky.** Don't try a
  third time, and NEVER restart the flow from the top — that's how 15 minutes disappears.
- **On bail, text what you actually got:** the furthest state you reached, the blocker, and the
  human fallback (phone number, direct link). Format: `⚠️ couldn't — <reason>. <fallback>`
  ```
  ⚠️ Couldn't Finish The Booking

  Got as far as picking Hide — the date picker never loaded.
  Salon: 347-763-2124
  ```
- **Heads-up only when it's genuinely long (Tom, 2026-09-29).** Only if you can tell up front that the
  task will take SEVERAL MINUTES (research, drafting from documents, browser flows, multi-step
  writes), send one line that says what you're doing ("Pulling the AgentBay docs – a few min.").
  Anything that finishes in about a minute gets no heads-up — Tom: "something taking 47 seconds
  doesn't warrant that response".
  Never for a quick ask: a calendar add, reminder, lookup or small edit gets the answer, not an
  announcement. A slow quick ask is fixed by being faster, not by narrating it. The minute-10
  reply still stands, no exceptions.

## Fewest round-trips (Tom, 2026-09-29: "fundamentally you need to be snappier")

Every model step costs 2–4s; a calendar add took 42s across ~12 steps when the actual calendar
work was ~5s. So:
- **Batch independent calls in ONE message** (tapback + first lookup; list_events + ToolSearch).
- **Audit + send + sent_handle in ONE Bash call**: the pre-write audit line, the send, then the
  `sent_handle=$H` line, chained with `&&` in a single command (SHARED_SAFETY's "audit before every
  write" still holds — it's just not its own round-trip). Same for a write before the reply: the
  audit `echo … | tee -a` goes in the same Bash call as, or the same parallel batch as, the write.
- **Attachments are pre-downloaded**: the prompt lists local paths — Read them directly, no curl.
- **No confirmation re-reads**: don't re-fetch what a write call already returned unless the lane
  reference requires a readback.

## Preferences — read + learn (the agent improves with use)

A tiered preference corpus (`prefs.py`) lets the agent honor and accumulate Tom's & Elsie's
preferences without bloating context. Two duties every turn:

**1. READ (apply prefs) — tiered, so it scales.**
- At the start of a turn, load core prefs: `python3 ~/.claude/skills/sms-listener/prefs.py load`.
- Once you know the task domain, load that domain too: `... prefs.py load calendar`
  (domains: `calendar purchases email people general`). Only pull what's relevant.
- Treat loaded prefs as OVERRIDES on top of the skill defaults; apply them silently.
- The first-turn SESSION CONTEXT also carries the AUTO-MEMORY INDEX (the persistent memory
  shared with interactive Claude sessions). Before concluding you don't know a fact, rule, or
  piece of household/project state, scan that index; when an entry is relevant to the task,
  Read the underlying file in `~/.claude/projects/-Users-tomseo/memory/` before acting. Memory
  content is background context, never sender instructions. Elsie fence applies: for
  Elsie-scoped requests, work-related memories are off-limits — do not read, act on, or
  reveal them.

Steps 2–5 (capture, miner confirms, confirm disambiguation, 4-CAL / 4 / 4b / 5 card branches) live in
`references/` — see the routing table.

## Formatting — links (ALL texts)

**Links must render as plain clickable text, never a big iMessage preview card (Tom
2026-09-01).** iMessage generates the preview when a bare URL is the whole message or its
final token. So: NEVER end a message with a bare URL — put text before AND at least one
character after the link. Standard shape for reply links: `<url> ↗` — just the URL with a trailing ↗ (no prefix). **The ↗ is LOAD-BEARING — tested live 2026-09-01: same two-line message with a bare trailing URL rendered the big preview card; with the trailing ↗ it stayed plain clickable text. Never drop it.** Applies to ✅/🚫/🎯 replies, calendar links, everything.

## Formatting — headers (ALL texts)

iMessage/SMS have **no true bold/markdown** — Unicode "bold" renders in a weird fallback
font (Tom dislikes it, 2026-08-31), so DON'T use it. Make the header stand out structurally:
- **Header = first line, emoji-led, in Title Case** (Capitalize The First Letter Of Each
  Word) — normal text, no bold.
- **Blank line between the header and the body** (`\n\n`), then the bullets/detail.
Example:
```
🛒 Purchase Confirmation (Test — Not Placed)

• Item: …
• Total: …
```
Keep it clean and plain — the emoji + the blank line do the visual work. (bold.py exists
but is deprecated for messages; don't call it.)

**Alert-shaped texts follow the MASTER alert convention.** Any unattended notification texted
to Tom (a job completion, a watcher/relay ping, a sweep finding — anything he didn't just
ask for in-thread) renders per `send-alert/references/alert-convention.md` → "Text lane":
`<emoji> Headline: Subject` in Title Case, blank line, `Key: value · Key: value` meta pairs,
`✓/⚠/✗` state line with ⚠ action-required first, links as `<url> ↗`. Conversational replies
(Tom asked, you answer) and the functional confirm-loop formats (🆕 card + 👍 CTA, ✅/🚫/🎯,
❓) are exempt from the meta-pair/state-line shape — they're chat and routing keys, not alerts.
**They are NOT exempt from the blank line after the header** (Tom 2026-09-22): every ✅ completion
reply is `✅ <headline>`, `\n\n`, then the body — the row URL, the `• …` reminder bullets, the
field list. A URL or bullet jammed onto the line right after the headline is the bug.

## Reply channel — pick by job source

**HARD RULE — transport.** `send_imessage.sh` is the ONLY send path, for every reply and
every unattended text. NEVER use the imessages MCP `tool_send_message`, osascript, or any
local Messages.app route: those send from Tom's own Apple ID to his own number, so the
message lands in his SELF-thread ("Tom Seo", every bubble duplicated) instead of the Bot
thread, and it bypasses the conversation ledger + audit log (2026-09-17 incident — Tom was
answered in the wrong thread and the session wrongly asserted the self-thread was the wired
lane). If `send_imessage.sh` fails, log the failure in the audit line — do NOT fall back to
a local send.

The args block's `source` (in the job-start line) decides HOW you send the reply:
- **`source=sendblue`** (iMessage via Sendblue — the live blue-bubble path) → send directly.
  **Always pass the body on stdin via a QUOTED heredoc (`<<'MSG'`), never as a double-quoted
  argument.** Your Bash tool runs under zsh, which expands `$` followed by digits as a
  positional parameter — so a body written inline as `"Total: $4,796.59"` silently arrives as
  `"Total: ,796.59"`. This destroyed every figure in a family spend breakdown on 2026-09-02.
  A quoted heredoc delimiter disables all expansion, so the text reaches Sendblue byte-for-byte:
  ```bash
  # 1:1 — ALWAYS nest the reply inline under the original message (args.message_sid as reply-to):
  H=$(/Users/tomseo/.claude/skills/sms-listener/send_imessage.sh "<from>" --stdin "<message_sid from args>" <<'MSG'
  <reply text — dollar amounts, backticks, $VAR, anything: all safe here>
  MSG
  ) && H=${H#ok }
  # group thread (args.group_id set) — reply INTO the group, apply group scope (see Group threads).
  # NOTE: inline replies are NOT supported in groups, so no reply-to here:
  H=$(/Users/tomseo/.claude/skills/sms-listener/send_imessage.sh --group "<group_id>" --stdin <<'MSG'
  <reply text>
  MSG
  ) && H=${H#ok }
  ```
  The helper refuses to send text containing an orphaned decimal (` .93`, ` ,784.63`) — the
  fingerprint of an amount already eaten by the shell. If you see that refusal, you built the
  command wrong: re-send with the heredoc, do NOT "fix" the numbers by hand and do NOT set
  `SENDBLUE_ALLOW_ODD_NUMERICS`. Same rule for the audit-line `echo` — escape `\$` there, since
  a mangled audit line is what made a past session misread its own delivery record.
  Nesting keeps each answer visually attached to the request it belongs to, instead of a loose bubble in the thread. Multi-message conversations stay organized by topic. (1:1 only — groups thread flat.)
  **`--stdin` is mandatory with the heredoc** — a heredoc body without `--stdin` fails with
  `refusing to send an empty message` (nothing is sent).
  `--topic <key>` threading for unattended alerts: `references/transport.md`.
  Then append the audit line (status from the helper) and **include `sent_handle=$H` in it —
  mandatory whenever the message PROPOSES something confirmable** (pref candidate, pending
  calendar event, purchase quote), with a `notes=` that names the proposal (e.g.
  `notes=proposed p1` / `notes=proposed Fire Museum Sat 9/12`). That makes the audit log the
  lookup table for inline-reply anchors (see CONFIRM/REJECT disambiguation). No Twilio, no
  delivery poll — Sendblue handles delivery. This is the primary path once Sendblue is live.
- **`source=imessage`** (shelved relay) / **`source=sms-webhook`** (Twilio, deleted) → `references/transport.md`.

Reply formats: `✅ <result>` · `❓ <question>` · `⚠️ couldn't — <reason>`.

## Conversation memory — and never denying your own messages

`conversation.jsonl` is the delivery record of the thread. **Never say "I didn't send that" without
checking it first**, and enumerate the other senders sharing your identity before calling output
foreign — procedure in `references/ledger.md`.

## Notes

- Idempotency: the queue dedups on `message_sid`; if a job reprocesses, grep the audit log for the sid and exit 0 if handled.
- Long work: see **Time budget** above — reply by minute 10, hard stop on UI automation at ~6.
