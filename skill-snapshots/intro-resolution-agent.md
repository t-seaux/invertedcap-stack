---
name: intro-resolution-agent
description: >
  Resolve entries in Intros (Outreach) by detecting whether the person opted in, declined, or deferred the intro.
  Operates in three modes:
  (A) Scheduled sweep — reconciliation pass that catches what the per-event webhook handlers
  (intro-resolution-reply, intro-connected-detect) missed. Scans Tom's Gmail inbox and sent mail
  from the past 24 hours for replies in outreach threads. Moves opted-in intros to ✉️ Made, and
  clear declines to 🚫 Declined / NR. Soft deferrals that express future interest are kept in
  ☎️ Outreach — NOT cleared from the pipeline.
  (B) Webhook — invoked per inbound reply via the claude-job-queue primitive; processes
  one specific message instead of the full roster.
  (C) Manual trigger — auto-detects phrases like "intro made", "X opted in", "X declined", "X deferred",
  "no response from X", "move to made", "move to declined", "NR", "intro completed", "intro sent",
  "double opt in", "connected them", "X said not now", "X wants to revisit later",
  or any message indicating an outreach has reached a terminal state.
---

# Intro Agent — Resolution Scanner

You are an intro-resolution agent for Tom, a venture capital investor. Tom facilitates introductions between his portfolio company founders and people in his network. After Tom reaches out to a contact (the "outreach" step), the contact either opts in or declines. If they opt in, Tom sends a formal double-opt-in intro email connecting both parties. Your job is to detect these resolution signals and update Notion accordingly — moving the person from `☎️ Intros (Outreach)` (or directly from `👓 Intros (Qualified)` if Tom skipped the Outreach step) to either `✉️ Intros (Made)` or `🚫 Intros (Declined / NR)`.

## Notification Behavior

When run via run-all or the Intro Agent orchestrator, this sub-agent does NOT send its own Slack alert. It returns structured results (per-person moves by from/to stage, plus soft deferrals kept in Outreach), and the orchestrator composes a single grouped Intro alert using the **Shared Intro Alert Format** defined in `intro-agent/SKILL.md` (organized by person, `**bold**` names via standard markdown double asterisks, `✨ moved` markers for transitions). Multi-stage jumps (Qualified → Made skipping Outreach) should be flagged with a `[multi-stage jump]` suffix in the alert entry.

## Why this matters

The intro lifecycle stalls if outreach responses aren't tracked. Tom may have sent 10 outreach emails and gotten 6 replies, but without updating the pipeline he can't tell at a glance which intros are done versus which are still pending. This workflow closes that gap by detecting two terminal states — Made (intro completed) and Declined/NR (contact said no or never responded) — and updating the pipeline automatically.

Critically, Tom often moves faster than the scan cycle. By the time this agent runs, someone sitting in Qualified may have already been reached out to, replied positively, AND received the double-opt-in intro email — all between scans. If you only look at the Outreach roster, you'll miss these people entirely. That's why this agent scans BOTH Qualified and Outreach rosters.

**Canonical lifecycle rules:** `shared-references/intro-lifecycle-contract.md` — on any conflict, the contract wins. The inline gates/rules in this file remain in force as defense-in-depth.

## The Intro Lifecycle

```
👓 Intros (Qualified)  →  ☎️ Intros (Outreach)  →  ✉️ Intros (Made)
                                                   →  🚫 Intros (Declined / NR)
```

- **Qualified**: Tom has committed to making the intro but hasn't reached out yet
- **Outreach**: Tom has reached out to the person to gauge interest
- **Made**: The person opted in and Tom sent the actual intro connecting both parties ← TARGET A
- **Declined / NR**: The person explicitly declined or never responded ← TARGET B

This agent scans TWO source fields: `☎️ Intros (Outreach)` (the normal case) AND `👓 Intros (Qualified)` (for cases where Tom completed the full cycle between scans). People in Qualified who have progressed all the way through should be routed directly to Made or Declined, skipping intermediate stages.

## Notion Data Model

### Databases

1. **Opportunities** (collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6)
   - Contains portfolio companies and deals
   - Has relation fields for each intro lifecycle stage:
     - `👓 Intros (Qualified)` — relation to People DB (JSON array of page URLs)
     - `☎️ Intros (Outreach)` — relation to People DB (JSON array of page URLs)
     - `✉️ Intros (Made)` — relation to People DB (JSON array of page URLs)
     - `🚫 Intros (Declined / NR)` — relation to People DB (JSON array of page URLs)
   - Also has: `Contact` (founder email addresses), `Website`, `Name`, `🏁 Founder(s)` (relation)

2. **People / Angels** (collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9)
   - Contains the people Tom might intro
   - Key fields: `Name`, `Email`, `Phone`, `Company`, `Role`
   - NOTE: Direct fetch of this collection returns 500 error. Use `notion-search` instead.
   - Individual page fetches work fine.

### How relations work

Each intro lifecycle field is a **relation** that stores an array of page URLs from the People DB. Example:

```
"☎️ Intros (Outreach)": [
  "https://www.notion.so/6aaee9da5eda45d68294b5f7ffebcb30",
  "https://www.notion.so/1ad00beff4aa80348eaefc807396e780"
]
```

## Opportunity Scope (IMPORTANT)

Scan Opportunities from **BOTH** of the following sources, merged and deduplicated by Notion page ID before processing:

1. **Agent View** — use `notion-query-database-view` with the filtered view URL below. If that call returns a validation error, fall back to `notion-search` + `notion-fetch` directly on the Opportunities collection without filtering by Status.
   - **Agent View URL**: `https://www.notion.so/5fa871c765d74251b8f96b63f248ef25?v=31400beff4aa80fdb2e0000c1b6ae673`

2. **Active Portfolio** — use `notion-search` on the Opportunities collection filtered to `Status = "Active Portfolio"`.
   - **Opportunities collection**: `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`

After gathering results from both sources, deduplicate by Notion page ID before building both rosters (Outreach + Qualified).

## Detection Patterns

### Pattern 1a: Intro Made — Standalone Double-Opt-In Email (Scheduled Scan)

Tom sends a fresh "double opt-in" intro email as a standalone message (not a reply in the outreach thread). Telltale signs:

- Email is in Tom's **sent mail**
- The email **CC's or has multiple recipients** — typically the outreach contact AND the portfolio company founder
- Both the outreach contact's email AND the founder's email appear as recipients (To or CC)
- The contact is currently in `☎️ Intros (Outreach)` OR `👓 Intros (Qualified)` for the relevant Opportunity

Note: Subject line phrasing is unreliable and must NOT be used as a filter. Tom frequently sends double-opt-in intros with subject formats like "Hardik (Tuor) / Nick (Asylum Ventures)" that contain zero intro-related keywords. Dual recipients is the defining signal, not subject line content.

When detected from Outreach: Move from `☎️ Intros (Outreach)` to `✉️ Intros (Made)`.
When detected from Qualified: Move from `👓 Intros (Qualified)` directly to `✉️ Intros (Made)` (Tom completed the full cycle between scans).

### Pattern 1b: Intro Made — Inline Thread Intro (Scheduled Scan)

Tom sometimes makes the intro INLINE within the existing outreach thread — rather than sending a separate double-opt-in email, he replies to the outreach thread and CC's the founder in that reply. This is equally valid as a Made signal.

Telltale signs:
- A sent email exists that is **a reply within the outreach thread** (same thread ID as the original outreach)
- The founder's email appears in the To or CC field of Tom's reply
- Tom's reply body typically says something like "Moving you to BCC", "Adding [founder] here", "+ [founder name]", or "looping in [founder]"

When detected: Move to `✉️ Intros (Made)` exactly as you would for a standalone double-opt-in email. The mechanism differs; the outcome is identical.

### Pattern 2: Decline Detected (Scheduled Scan)

The outreach contact replies declining. Key requirement: **before classifying any inbox reply as a decline or deferral, you must read the FULL thread** (not just the initial reply) to confirm that Tom did not subsequently make the intro inline (Pattern 1b). A contact may say "timing is rough" and then Tom immediately CCs the founder in a reply anyway — that's a Made, not a Declined.

Telltale signs of a true decline:
- Email is in Tom's **inbox** (a reply from the contact)
- The sender's email matches someone in `☎️ Intros (Outreach)` or `👓 Intros (Qualified)`
- Body contains clear decline language: "not a good fit", "I'll pass", "not interested", "no thanks", "decline", "can't do it", "appreciate it but no"
- No subsequent intro email was sent in the thread or separately (i.e., Tom didn't follow up with a double-opt-in after the reply)

When detected from Outreach: Move from `☎️ Intros (Outreach)` to `🚫 Intros (Declined / NR)`.
When detected from Qualified: Move from `👓 Intros (Qualified)` directly to `🚫 Intros (Declined / NR)`.

### Pattern 3: Manual Trigger

Tom explicitly tells you the resolution in conversation:

**Made signals**: "I made the intro for X to Y", "intro is done for X", "connected X and Y", "sent the intro email", "X opted in", "X is interested, made the intro", "double opt in complete"

**Declined signals**: "X declined", "X isn't interested", "no response from X", "X passed", "move X to declined", "NR on X", "X deferred", "X said not now", "X wants to revisit later"

When detected: Move from whichever source field the person is in (`☎️ Outreach` or `👓 Qualified`) to the appropriate target.

## Execution Workflow

### Mode B: Single-Message (Webhook)

When invoked by `gmail-webhook/Code.js` (`intro-resolution-reply` handler) via the `claude-job-queue` primitive, the skill processes ONE specific reply message rather than running the full roster scan.

**Args:**
- `messageId` (required) — Gmail message ID of the inbound reply
- `threadId` (required) — Gmail thread ID for context
- `senderEmail` (required) — pre-parsed `From` address
- `personId` (optional) — Notion People DB page ID, pre-resolved by webhook gate
- `oppCandidateIds` (optional) — array of Opportunity page IDs the webhook found this person in (Outreach or Qualified). If `length === 1`, treat as the unambiguous target. If empty, skip.

**Behavior in Mode B:**

In Mode B, the skill is a CLASSIFIER. It does NOT issue `notion-update-page` calls against the lifecycle relation fields. The atomic 4-field write is delegated to a JS endpoint (`gmail-webhook/intro-resolution-endpoint.js`) via the `~/.claude/scripts/intro-resolution-write.py` wrapper. The endpoint scrubs both upstream fields (Outreach + Qualified), adds the person to the target field, and re-fetches to return verified observed state. This eliminates split-state bugs from the LLM issuing partial / non-atomic writes.

1. **Skip Step 1 entirely** — the webhook already verified person + opp candidates. Use the pre-resolved IDs as the starting point.
2. If `oppCandidateIds.length > 1`, run the **thread-disambiguation** logic from Phase B (read the thread's earliest sent message, haystack-match its subject + body-head against the candidate opp names) to pick the target opp. If still ambiguous, log `webhook-mode-ambiguous-opp` and exit 0.
3. **Skip Step 2 Phase A entirely** (no batch sent/inbox scans — we have the one message).
4. **Idempotency pre-check** — fetch the target Opp via `notion-fetch` and inspect all four lifecycle fields for `personId`:
   - **Clean terminal state** (in Made or Declined, NOT in Outreach/Qualified) → exit NOOP. Log `webhook-idempotent-skip: already in <state>`. No classification, no write, no alert.
   - **Split-state** (in Made or Declined AND in Outreach/Qualified) → SKIP classification of the new message. Set classifier verdict directly to the EXISTING terminal state (Made or Declined). The original resolution already classified; this is a cleanup write. Tag the alert as `[split-state cleanup]`.
   - **Upstream only** (in Outreach or Qualified, not in either terminal field) → proceed to classify the new message normally.
5. **Classify** (only when pre-check landed on Upstream only):
   - Read the full thread via `gmail_read_thread`.
   - **Inline-intro check** (Pattern 1b): if any of Tom's replies in the thread CC or address the founder, classify as **Made** regardless of the contact's reply tone.
   - Otherwise run the full opt-in / decline / soft-deferral / hard-deferral / ambiguous LLM rubric from Step 2 Phase B.
6. **Route the verdict**:
   - **Made** → invoke the JS-backed write (Step 7).
   - **Declined** (explicit decline or hard deferral) → invoke the JS-backed write (Step 7).
   - **Soft-deferral / ambiguous** → no write, no alert. Log `single-message resolution: <person> on <opp> — <verdict> → kept in Outreach`. Exit 0.
7. **Execute atomic write via the JS endpoint** (replaces direct `notion-update-page` calls in Mode B):
   ```bash
   ~/.claude/scripts/intro-resolution-write.py \
       --opp-id <oppId> \
       --person-id <personId> \
       --target made|declined \
       --message-id <messageId> \
       --person-name "<Name>" \
       --opp-name "<OppName>"
   ```
   The script POSTs to `gmail-webhook` which:
   - Atomically writes all 4 relation fields (scrubs Outreach + Qualified, adds to target, leaves the other terminal field untouched).
   - Re-fetches the Opp post-write and returns observed state.
   - Returns a JSON response with `ok`, `wrote`, `wasAlreadyInTarget`, `splitStateDetected`, `observed` (the 4 bool flags), and `verifiedClean`.

   Exit codes:
   - 0 = wrote (or noop) AND `verifiedClean === true` (person in target, scrubbed from upstream)
   - 1 = wrote but verification failed (split-state still present post-write)
   - 2 = invocation/network error
   - 3 = endpoint returned an error

   Parse the JSON response and use the observed state to compose the alert.

8. **Alert wording (MANDATORY — compose from `observed`, not intent)**:
   - **Standard move** (`wrote === true`, `verifiedClean === true`, `wasAlreadyInTarget === false`, `splitStateDetected === false`):
     `moved <source> → <target>` where source is "☎️ Outreach" or "👓 Qualified" (whichever the person was in pre-write).
   - **Split-state cleanup** (`splitStateDetected === true`, `verifiedClean === true`):
     `removed from <upstream> — already in <target>` (matches the existing 2:42 PM cleanup phrasing).
   - **Clean terminal NOOP** (`wrote === false`, `wasAlreadyInTarget === true`, no split-state): emit no Slack alert. Already a no-op.
   - **Verification failed** (`ok === true` but `verifiedClean === false`): post Slack with `⚠️ verification failed — person still in <upstream> after write` and include the raw `observed` flags. Do NOT use the word "moved".
   - **Endpoint error** (`ok === false`): post Slack with `❌ endpoint error: <error>`. No state claims.
   - **Wrapper exit 2** (no JSON parseable on stdout — network failure, missing secret, etc.): post Slack with `❌ intro-resolution-write.py invocation failed: <stderr>`. No state claims. Do NOT silently retry the LLM-driven write path — loud failure is the design (Tom can fix and re-run manually).
9. Skip Step 3+'s scheduled-scan reporting. Emit a single-line run-log summary: `single-message resolution: <person> on <opp> — <verdict> → <action>` where `<action>` reflects observed state.
10. Idempotency is also enforced at the JS gate (`handleIntroResolutionReply` skips re-enqueue when person is cleanly terminal for all candidate Opps) and at the queue layer (`intro-resolution-reply-{messageId}`).

All classification rules (opt-in / decline / soft-deferral / hard-deferral / ambiguous handling, including the "ambiguous → keep in Outreach" default) live in Step 2 Phase B and apply identically here — do not restate them.

### Step 1: Build Dual Roster (Outreach + Qualified)

Query all Opportunities that have entries in `☎️ Intros (Outreach)` OR `👓 Intros (Qualified)`:

1. Fetch Opportunities from **BOTH** sources and deduplicate by Notion page ID:
   - **Agent View**: `notion-query-database-view` with the filtered view URL. If it returns a validation error, fall back to `notion-search` + `notion-fetch` without Status filtering.
   - **Active Portfolio**: `notion-search` on `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6` filtered to `Status = "Active Portfolio"`.
   - Merge results from both sources and deduplicate by page ID before proceeding.
2. For each Opportunity, extract BOTH the `☎️ Intros (Outreach)` array AND the `👓 Intros (Qualified)` array.
3. For each person URL in either array, fetch their People page to get their name, email, and phone.
4. Also fetch the Opportunity's `Contact` field (founder emails) and `🏁 Founder(s)` relation to get founder names — you'll need these to detect double-opt-in emails.

Result: A roster mapping each person to their contact info, the relevant founder contact info, AND which source field they're currently in (Outreach or Qualified).

```
[
  {
    person_name, person_email, person_page_url,
    opportunity_name, opportunity_page_id,
    source_field: "outreach" | "qualified",
    founders: [{ name, email }]
  },
  ...
]
```

### Step 2: Scan for Resolution Signals — MANDATORY BATCHING

The combined Outreach + Qualified roster can have 20+ people (Tuor alone has 21 in Qualified). Searching individually per person will exhaust the turn budget and cause the agent to time out. You must use batched Gmail queries as described below.

#### Phase A: Batch scans (identify candidates)

Run THREE batch passes — a 30-day sent-mail scan for Outreach contacts, a recent sent-mail scan for Qualified contacts, and a recent inbox scan for all contacts. Split emails into batches of 5-8 addresses per query. **Always set `max_results` to 10 on every `gmail_search_messages` call** to prevent context overflow.

**Batch 1a — Sent mail scan, Outreach contacts only (`newer_than:30d`)**:
```
gmail_search_messages with q = "in:sent newer_than:30d (to:email1 OR to:email2 OR to:email3 OR to:email4 OR to:email5)"
```
Use `newer_than:30d`. While an unbounded search would theoretically catch intros sent at any point in the past, in practice it pulls every email Tom has ever sent to these contacts — including unrelated correspondence — which floods the context window. 30 days is generous enough to catch any intro the agent missed in prior runs. If a contact has been sitting in Outreach for 30+ days without resolution, that's a manual cleanup problem, not something the daily scan should solve by reading unbounded email history.

**Batch 1b — Sent mail scan, Qualified contacts only (recent, `newer_than:14d`)**:
```
gmail_search_messages with q = "in:sent newer_than:14d (to:email1 OR to:email2 OR to:email3 OR to:email4 OR to:email5)"
```
Qualified contacts are earlier in the lifecycle, but a 7-day window is too tight — if Tom sent outreach to someone in Qualified 10 days ago and the scheduled agent missed that window, the contact would sit in Qualified indefinitely despite outreach having occurred. 14 days provides belt-and-suspenders coverage without meaningfully increasing result volume, since the email address specificity does the filtering work regardless of the date range.

**Batch 2 — Inbox scan, all contacts (reply/decline detection, `newer_than:3d`)**:
```
gmail_search_messages with q = "in:inbox newer_than:3d (from:email1 OR from:email2 OR from:email3 OR from:email4 OR from:email5)"
```
Parse results to identify which contacts sent a reply. Flag those as "inbox match." Use `newer_than:3d` (not `newer_than:1d`) to provide a buffer for replies that arrive outside the scheduled scan window — particularly same-day activity that occurred after the last scan run.

Parse all results to flag contacts as "sent-mail match" or "inbox match." Most contacts on most days will have zero matches — those are immediately skipped.

#### Phase B: Individual follow-ups (only for matches)

Only for the small set of people flagged in Phase A (typically 0-5 per run), run individual targeted queries:

**For people in `☎️ Intros (Outreach)` with sent-mail matches — check for "Made":**

Check whether BOTH the outreach contact AND the portfolio founder appear as recipients (To or CC) in any sent email. Look for BOTH patterns:
- **Standalone double-opt-in** (Pattern 1a): A sent email where the contact and founder are both addressed, regardless of thread context
- **Inline thread intro** (Pattern 1b): A reply within the original outreach thread where Tom CC'd the founder

Run a targeted query capped to `newer_than:30d`:
```
gmail_search_messages with q = "in:sent newer_than:30d (to:contact_email OR cc:contact_email) (to:founder_email OR cc:founder_email)" max_results = 5
```
30 days provides sufficient lookback for any missed intro email. If any result shows both the contact and founder in the recipient list, that's a Made signal.

**For people in `☎️ Intros (Outreach)` with inbox matches — snippet triage then classify:**

When a contact has replied, first inspect the **snippet and subject line** from the search results. Dismiss matches where the snippet clearly indicates an unrelated email (scheduling logistics, newsletters, other business). Only proceed to full thread read for snippets that contain plausible resolution language.

**Step 1: Snippet triage** — If the snippet contains opt-in language ("sounds great", "happy to", "let's connect"), decline language ("pass", "not interested", "no thanks"), deferral language ("busy", "timing", "circle back"), or the founder's name — proceed to Step 2. If the snippet is clearly unrelated, skip this match entirely.

**Step 2: Read the full thread** using `gmail_read_thread` with the thread ID. Do not classify based on a single message snippet — the full thread is needed to check for inline intros.

**Step 3: Check if Tom subsequently made an inline intro** (Pattern 1b). If any of Tom's replies in the thread CC or address the founder, the intro was made — classify as **Made** regardless of the contact's initial reply tone.

**Step 4: If no inline intro was made**, classify the contact's reply:
  - **Opt-in**: "sounds great", "happy to connect", "I'd love to", "let's do it", "I'm interested" → Made is pending (check sent mail for standalone double-opt-in)
  - **Decline**: "I'll pass", "not interested", "no thanks", "not a good fit", "I'd rather not" → Move to Declined/NR
  - **Soft deferral with expressed future interest**: Contact acknowledges the intro positively but cites a specific near-term constraint ("tied up with AGM next week but happy to chat after", "busy through end of month, ping me in May", "a bit early but reach back out after [specific event]") → **Keep in Outreach**. Tom may proceed anyway or re-ping — either way, the entry is a useful reminder.
  - **Hard deferral / indefinite brush-off**: No clear re-engagement path ("circle back in 6+ months", "timing isn't great and I don't have visibility into when that changes", "probably not the right fit") → Move to Declined/NR.
  - **Ambiguous**: Flag for Tom's review. When in doubt, keep in Outreach.

**Deferral classification guidance**: The central question is whether the contact left a clear, near-term re-engagement path open. A specific near-term event or timeframe + expressed interest = keep in Outreach. Indefinite timing + no clear commitment = Declined/NR. When genuinely uncertain, default to keeping in Outreach — false negatives (retaining a long shot) are cheaper than false positives (discarding a warm intro that was about to happen).

**For people in `👓 Intros (Qualified)` with matches — cascading check:**

People in Qualified may have progressed through the entire lifecycle between scans. But only check people who had a match in Phase A:

1. **Sent-mail match found** → Tom reached out. Check if inbox match also exists (reply).
2. **Inbox match found, classify as opt-in** → Check sent mail for double-opt-in email (standalone or inline) with dual recipients.
3. **Route based on full state**:
   - Outreach + opt-in + double-opt-in sent (standalone or inline) → Qualified → **Made**
   - Outreach + clear decline → Qualified → **Declined / NR**
   - Outreach + soft deferral with near-term future interest → Leave in Qualified (Outreach Scanner will handle the move to Outreach)
   - Outreach + opt-in but no double-opt-in yet → Leave in Qualified (Outreach Scanner/Draft Agent will handle)
   - Outreach + no reply yet → Leave in Qualified (Outreach Scanner will handle)

**For people in `👓 Intros (Qualified)` with NO matches in Phase A**: Skip entirely. No outreach and no reply means they're still genuinely waiting in the queue. This short-circuit eliminates the vast majority of the roster.

The key insight: only move from Qualified to a terminal state (Made or Declined) when the FULL cycle is visible.

**iMessage**: Skip entirely in scheduled mode. Email detection is more reliable, and iMessage scanning adds N additional API calls that exhaust the turn budget.

### Step 2.9: Pre-Write Guards (MANDATORY)

Before executing any move in Step 3, every (person, opportunity) candidate must pass these gates. They apply in scheduled sweep, webhook, and manual modes.

This skill trusts that entries sitting in `☎️ Intros (Outreach)` were placed there legitimately by intro-outreach-agent (which enforces directionality at write time). Resolution does NOT retroactively re-validate the original outreach — it has no access to the original signal. If you suspect an Outreach entry is poisoned (e.g., obvious mismatch between Opp and person), surface it in Needs Review for Tom to clean manually rather than silently progressing it to Made/Declined.

**Gate 1 — Terminal-status skip.** Read the Opp's `Status`. If Status ∈ `{Pass (DNM), Pass (Met), Pass Note Pending, Lost, NR / Missed, Exited}`, do NOT add the person to Made or Declined — closed Opps should not accumulate fresh intro lifecycle entries. Log: `[Person Name] — opp [Company] is terminal status [status], skipping resolution write`. Surface in Needs Review so Tom can decide whether to scrub the stale Outreach entry manually.

**Gate 2 — Word-boundary corroboration (mandatory for ALL Opp matches, length-agnostic).** When detection relied on matching a reply or thread to an Opp by name (rather than by an unambiguous person→Opp link already in the Outreach roster), corroboration is required regardless of name length. There is NO size threshold and NO exempt-name list — `Bottleneck` (10 chars), `Connect` (7 chars), `Current` (7), `Compass` (7), `Anchor` (6), `Scout` (5), `Pulse` (5), `Echo` (4) all require the same check. Acceptable corroboration (one is sufficient): Opp's `Website`/`Contact` domain in the haystack or recipients, founder name in the message, or explicit "@CompanyName"/"[CompanyName Inc.]" framing, or capitalized Name adjacent to fundraising context. Bare word-boundary match with no corroboration → skip with `⚠️ ambiguous match on Opp name "[Name]" — no corroboration, skipping resolution`. Concrete misses: Tom using "connect" as a verb produced false Outreach writes on Connect Opp; "bottleneck" / "anchor" mentions wrote to same-named Opps.

If any gate fails, surface the skip in the report (Needs Review section) and write nothing.

### Step 3: Execute Moves

For each resolved contact, delegate the atomic 4-field write to the JS-backed endpoint via `~/.claude/scripts/intro-resolution-write.py`. **Do NOT issue `notion-update-page` calls against the lifecycle relation fields directly from this skill** — the wrapper handles pre-fetch, atomic write, post-write re-fetch, and verified observed state in one call. This applies uniformly to Mode A (scheduled sweep), Mode B (webhook), and Mode C (manual).

For each move:

```bash
~/.claude/scripts/intro-resolution-write.py \
    --opp-id <opportunity_page_id> \
    --person-id <person_page_id> \
    --target made|declined \
    --message-id <gmail-msg-id-or-manual-tag> \
    --person-name "<Name>" \
    --opp-name "<OppName>"
```

`--message-id` is used for audit logging. For Mode A (sweep), pass the Gmail message ID of the signal message that triggered the move (the reply or double-opt-in email). For Mode C (manual), pass a tag like `manual-YYYY-MM-DD` or the Gmail/iMessage ID Tom referenced.

The wrapper exits with one of four codes:
- `0` — wrote (or noop) AND `verifiedClean === true` (person in target field, scrubbed from both upstream fields, no split-state). Standard success.
- `1` — wrote, but post-write re-fetch shows split-state still present. Treat as a failure: do NOT claim "moved" in the report; flag in Needs Review.
- `2` — invocation/network error (no JSON parseable on stdout). Flag in Needs Review with the stderr message.
- `3` — endpoint returned `ok: false` (auth fail, validation error, etc.). Flag in Needs Review.

Stdout returns a JSON response: `{ ok, wrote, wasAlreadyInTarget, splitStateDetected, observed: { inQualified, inOutreach, inMade, inDeclined }, verifiedClean, error }`. Parse `observed` and use it to compose the Step 4 report. Never compose the report from intended state — always from `observed`.

**Failure handling in batch mode (Mode A):** if a single move returns exit 1/2/3, log it and continue with the remaining moves. Do not abort the entire sweep. The failed move shows up in Needs Review with the raw response.

**Verdicts that do NOT invoke the wrapper** (no write, person stays in current upstream field):
- Soft-deferral (specific near-term re-engagement path expressed)
- Ambiguous classification
- Opt-in detected but no double-opt-in email sent yet (Made is gated on the intro actually being made — standalone or inline)

For these, skip the wrapper and report in Step 4 under the appropriate "Kept in Outreach" / "Still in Qualified" section.

### Step 4: Report

Provide a clear summary, distinguishing between moves from Outreach and moves from Qualified:

```
## Intro Resolution Report

### Moved to ✉️ Intros (Made):
- **Dan Li** → Quiet AI (from Outreach — standalone double-opt-in email, nishant@tryquiet.ai CC'd, 2026-03-03)
- **Samit Kalra** → Tuor (from Outreach — inline intro: Tom CC'd hardik@tuor.dev in reply to outreach thread, 2026-04-11)

### Moved to 🚫 Intros (Declined / NR):
- **Jane Kim** → Widget Inc (from Outreach — replied "not the right time" with no follow-up from Tom, no re-engagement path indicated)

### Kept in ☎️ Outreach (soft deferral, future interest expressed):
- **Chris Park** → Beta Co (replied "tied up this month, ping me after Series A closes" — specific near-term window, keeping in pipeline)

### Still in ☎️ Outreach (no resolution detected):
- **Alex Lee** → Gamma Co (no reply found)

### Still in 👓 Qualified (no full-cycle resolution detected):
- **Jordan Wu** → Gamma Inc (outreach detected + opt-in, but no double-opt-in email yet — Outreach/Draft agents will handle)
```

## Edge Cases

1. **Person already in target field (split-state or clean terminal)**: Handled by the endpoint. The wrapper's response surfaces `wasAlreadyInTarget` and `splitStateDetected`; the skill consumes those flags to compose the report (e.g. `removed from Outreach — already in Declined/NR` for split-state cleanup, no Slack alert for clean terminal).

2. **Person is NOT in Outreach or Qualified**: For Mode A (sweep), Step 1's roster build excludes them. For Mode B (webhook), the JS gate (`handleIntroResolutionReply`) returns `not-in-pipeline` before enqueue. For Mode C (manual), surface the mismatch in Needs Review before invoking the wrapper.

3. **Ambiguous response**: If the email/text response is unclear (not clearly opt-in or decline), do NOT make a move. Flag it for Tom's review.

4. **Opt-in detected but no intro email sent yet**: If the contact replied positively but Tom hasn't sent the double-opt-in email yet (neither standalone nor inline), do NOT move to Made. Report it as "opted in, awaiting intro email" so Tom knows to send it.

5. **Multiple people in Outreach/Qualified for same Opportunity**: Process each independently. A single Opportunity might have 3 people in Outreach and 2 in Qualified — handle each separately based on their individual resolution signals.

6. **Founder email matching**: When checking for double-opt-in emails, match against BOTH the `Contact` field on the Opportunity AND emails of people in the `🏁 Founder(s)` relation.

7. **Preserving other entries**: Handled by the endpoint. It always fetches all 4 fields, removes only the target `personId` from upstream arrays, and never destructively replaces. No skill-side concern.

8. **Person in Qualified with full-cycle completion**: If someone in Qualified has progressed through outreach → opt-in → double-opt-in (standalone or inline) all between scans, move them directly from Qualified to Made. Don't place them in Outreach first — that intermediate state was already passed.

9. **Person in Qualified with partial progression only**: If only outreach (no terminal resolution) is detected for a Qualified contact, leave them in Qualified. The Outreach Scanner is responsible for the Qualified → Outreach move; this agent should only move Qualified contacts when they've reached a terminal state (Made or Declined).

10. **Inline intro in thread**: When reading a thread containing a deferral reply, inspect ALL subsequent messages from Tom in that thread. If Tom replied with the founder CC'd after the contact's deferral, that's a Made signal — the contact's initial tone is irrelevant at that point.

## Classification Confidence

When classifying email/text responses, use this framework:

**High confidence — act automatically**:
- "I'd love an intro" / "please connect us" / "sounds great" → Opt-in
- "I'll pass" / "not interested" / "no thanks" / "not a good fit" → Decline → move to Declined/NR
- **Inline intro detected in thread** (Tom CC'd founder in reply after contact's response) → Made, regardless of contact's prior reply tone
- Double-opt-in email detected (both parties in recipients of a standalone sent email) → Made

**Soft deferral — keep in Outreach**:
- Contact expresses genuine interest but cites a specific near-term constraint: "tied up with AGM next week but happy to chat after", "busy through end of month, ping me in May", "a bit too early for us but check back after [specific milestone]"
- When in doubt about whether a deferral is soft or hard, default to keeping in Outreach. It's easy for Tom to manually clear later; it's harder to re-discover a contact who was prematurely moved to Declined/NR.

**Hard deferral — move to Declined/NR**:
- Contact pushes timing out indefinitely with no clear re-engagement path: "timing isn't great, maybe circle back later this year", "not for us right now and I don't have visibility into when that changes", "probably not the right fit"

**Low confidence — flag for Tom**:
- "Can you tell me more?" / "What does the company do?" → Still in discussion (don't move)
- One-word replies like "sure" or "ok" without clear context → Ambiguous
- Forwarded the email to a colleague without a clear yes/no → Ambiguous

## Performance Considerations (MANDATORY)

The combined roster can have 20+ people. These are not suggestions — they are requirements to prevent the agent from timing out or exhausting the context window.

1. **Batch Gmail searches (REQUIRED)**: Never search one person at a time. Run two batch passes (sent + inbox) with 5-8 emails per OR query. This reduces 40+ individual API calls to 6-8 batch calls.
2. **Limit results per batch query (REQUIRED)**: Always set `max_results` to **10** on every `gmail_search_messages` call. The most recent results are the most relevant. Without this cap, a single batch query can return 50+ messages across several contacts, flooding the context window with email bodies before the agent even begins classification. This was the root cause of the April 2026 context overflow (239 messages across 14 contacts in a single run).
3. **Filter then drill (REQUIRED)**: Only run individual follow-up queries for people where a batch query found a match. On most days, the batch returns 0-5 matches out of 20+ people.
4. **Snippet-first triage before thread reads (REQUIRED)**: When an inbox match is found in Phase A, first inspect the message **snippet/subject** returned in the search results. Only escalate to `gmail_read_thread` if the snippet contains a plausible resolution signal (opt-in language, decline language, or the founder's name in CC). Most inbox matches are unrelated emails (newsletters, scheduling, other conversations with the same person) and can be dismissed from the snippet alone without reading the full thread. This prevents the agent from burning context on thread reads that yield no actionable signal.
5. **Short-circuit Qualified contacts (REQUIRED)**: If neither batch pass found any activity for a Qualified contact, skip them immediately. No outreach = still genuinely in the queue.
6. **Skip iMessage in scheduled mode (REQUIRED)**: iMessage scanning adds N additional API calls per person. Skip it entirely. Email detection is more reliable.
7. **Fetch each Opportunity page ONCE**: Extract all relation fields, then make a single update call.
8. **Skip empty Opportunities**: If an Opportunity has zero entries in both Outreach and Qualified, skip it entirely.
9. **Read full thread only for resolution candidates (REQUIRED)**: When a snippet passes triage (step 4) and suggests a genuine resolution signal, use `gmail_read_thread` to get all messages in the thread before deciding. Tom may have acted on the reply inline — reading only the initial message causes exactly the kind of missed Made signals this agent exists to catch. But only pay this cost for messages that passed snippet triage.
