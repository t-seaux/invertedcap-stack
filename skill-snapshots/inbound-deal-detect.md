---
name: inbound-deal-detect
description: "Webhook-triggered classifier for inbound cold deal emails. Fetches the target message via `get_thread(threadId)` and locates it by `messageId`, runs a deal-vs-not-deal classifier with confidence gating, and on a confident positive delegates to `add-to-crm` for full pipeline entry. Not user-facing — invoked exclusively by `gmail-webhook/deal-scanner.js` via the `claude-job-queue` primitive. Never trigger manually; for ad-hoc CRM creation use `add-to-crm` directly."
---

# Inbound Deal Detector (Webhook)

Classifies a single inbound Gmail message as a new deal or not, then hands off to `add-to-crm` if it's a deal.

This skill is the deeper-classification half of `gmail-webhook/deal-scanner.js`. The webhook does a cheap deterministic pre-filter (INBOX, no DRAFT, no SENT-unless-self-forward, no noreply) and then a Haiku gate (`classifyWithLLM`) that decides `is_deal` + `is_third_party_forward` + extracts the forwarded founder candidate. On positive, it enqueues a job here. This skill does the expensive part: re-fetch the full email, run the canonical classifier rubric (with attachment list, Source attribution, field extraction), and on positive delegate to `add-to-crm`.

## Args

The local processor invokes this skill with these args (set by `deal-scanner.js`):

- `messageId` (required) — Gmail message ID. Use this to identify the target message inside the thread returned by `get_thread`.
- `threadId` (required) — Gmail thread ID. **Always use this for the fetch** (see Step 1). The MCP toolset has no "get message by API ID" tool, so the messageId alone is not directly fetchable.
- `senderEmail` (required) — pre-parsed by the webhook. For non-forwarded mail this is the outer `From:` header. For self-forwards (see `forwardedFromTom`) and third-party forwards (see `forwardedFromReferrer`), this is the **inner** forwarded `From:` line — the original founder or referrer who sent to Tom, not Tom and not the outer envelope.
- `senderName` (optional) — display name from the same source.
- `forwardedFromTom` (optional, boolean) — `true` when the message is a `Fwd:` from one of Tom's alias addresses (e.g. `tom@dashfund.co`) into the watched inbox. The webhook has already swapped `senderEmail`/`senderName` to the inner forwarded sender. The classifier should treat the forwarded body block as the canonical email content and infer Source from the inner `To:` header (see Step 1B).
- `forwardedFromReferrer` (optional, boolean) — `true` when the message is a `Fwd:` from an external third party (not Tom, not the founder) who is forwarding the founder's email along. The webhook has already swapped `senderEmail`/`senderName` to the inner forwarded sender (the founder candidate). `referrerEmail` / `referrerName` carry the outer envelope (the referrer). See Step 1C.
- `referrerEmail` (optional) — outer envelope email when `forwardedFromReferrer` is true.
- `referrerName` (optional) — outer envelope display name when `forwardedFromReferrer` is true.
- `materialUrls` (optional, array of strings) — deck/material URLs the webhook extracted from the email body (Drive, DocSend, Dropbox, Brieflink, Pitch.com, Figma, Canva, Notion.site, raw PDFs). When present, this list is **authoritative**: every URL MUST be passed through to `add-to-crm` so it runs Step 1B (read for thin-body field extraction) and Step 6 (link in Diligence Materials property). Skipping a URL because "the body context didn't seem deck-shaped" is not allowed — the webhook already filtered out company-website links. See the 2026-05-12 Unicorn Snot regression for why this gate moved server-side.

## Workflow

### Step 1: Fetch the email

Call `mcp__claude_ai_Gmail__get_thread` with the `threadId` arg. The MCP has no "get message by API ID" tool — `threadId` is the only direct fetch path. Inside the returned thread, locate the message whose `id` matches the `messageId` arg; that's the target. Grab:

- Subject
- Plain-text body (strip quoted history below the `On … wrote:` line — only the new content matters for classification)
- Confirmed `From` (verify it matches `senderEmail` arg; if not, log and prefer the actual header)
- Any attachment filenames
- Pass `threadId` through to add-to-crm in Step 4 so it can write `Source Thread ID` on the new Opp page. This enables `outreach-detector` / `outreach-decliner` Path D (deterministic thread-based status flips on Tom's outbound replies).

If the fetch fails (thread not found, target messageId missing inside thread), log to the run log AND append a line to `~/.claude/skills/inbound-deal-detect/audit-log/YYYY-MM-DD.log` via `mkdir -p ~/.claude/skills/inbound-deal-detect/audit-log && echo "[$(date '+%F %T')] FETCH_FAILED: <reason> (messageId: <id>, threadId: <tid>)" | tee -a ~/.claude/skills/inbound-deal-detect/audit-log/$(date '+%F').log`, then exit non-zero so the job lands in `failed/` for retry. The literal `tee -a` shell line must run — never claim the write happened without executing it.

### Step 1B: Forwarded-from-Tom handling

If `forwardedFromTom` is true, the outer envelope is just Tom forwarding the email to himself. The real content lives inside the forwarded block (everything after `Begin forwarded message:` / `---------- Forwarded message ----------`). For classification purposes:

- Treat the forwarded block's `From:`, `To:`, `Subject:`, and body as the canonical email — that's what was actually sent to Tom.
- The classifier should ignore Tom's outer `Fwd:` subject prefix and any cover note Tom wrote when forwarding (usually empty).

For Source attribution and Status default when delegating to `add-to-crm`:

Compare the inner forwarded `From:` email domain against the pitched company's website/domain (from the classifier's `website` field, or inferred from the email body if `website` is empty).

- **Inner sender's domain matches the pitched company** (e.g. `john@highroad.capital` for HighRoad) → the founder is the actual sender; Tom is just bouncing his own inbound to his watched inbox. **Source = Direct**, Status default = `Connected`.
- **Inner sender's domain does NOT match the pitched company** (e.g. `lurein@givecard.io` forwarding John Daniels's HighRoad pitch — a 2-level forward where a third party referrer's intro offer to Tom got re-forwarded across Tom's aliases) → the inner sender is a referrer offering an intro, with the founder living in a deeper-nested forwarded block. **Source = <inner sender's display name>** (resolved by looking up `senderEmail` in the People DB — relation URL goes in `Source(s)`; do NOT default to "Direct"). Status default = `Qualified` — the intro is offered but not yet made; the founder is NOT in the thread with Tom yet. If Tom has already replied opting in to the intro in this thread, escalate Status to `Outreach`.

The `To:` field is not authoritative for referrer detection — Tom often forwards to himself across his own aliases, which makes a `To: tom@*` match meaningless. Always discriminate on the inner sender's domain ↔ pitched company domain.

### Step 1C: Forwarded-from-referrer handling

If `forwardedFromReferrer` is true, the email arrived from a third-party referrer who forwarded a founder's email to Tom (e.g. an LP, a friend, or a past founder passing along an intro). The outer envelope is the referrer; the real content lives inside the forwarded block. The webhook has already set `senderEmail`/`senderName` to the inner forwarded sender (the founder candidate) and `referrerEmail`/`referrerName` to the outer envelope.

For classification: same body-unwrap as Step 1B — treat the forwarded block's `From:`/`To:`/`Subject:`/body as the canonical email and ignore Tom's outer cover note.

For Source attribution when delegating to `add-to-crm`:

- **Source = `referrerName`** (the outer envelope's display name). If `referrerName` is empty, use the local-part of `referrerEmail`.

### Step 2: Classify

Apply the classifier rubric below. Return a single JSON object — **no markdown fences, no commentary**:

```json
{
  "is_deal": true | false,
  "confidence": "high" | "medium" | "low",
  "reason_if_not_deal": "string",
  "company": "string",
  "description": "string",
  "founder_first_name": "string",
  "founder_li_url": "string",
  "website": "string",
  "round_details": "string",
  "stage": "string"
}
```

**A "new deal" is:**
- A founder pitching their startup for investment
- A referral intro of a startup seeking investment
- A meeting request from a founder to discuss fundraising

**NOT a new deal (`is_deal=false`):**
- Founder update on an existing portfolio company (check-ins, quarterly updates, status reports) — these go through `investor-update`
- Networking, social plans, dinner invites, casual catch-up
- SaaS marketing, newsletters, product announcements, cold sales
- Recruiting / job postings / hiring asks
- LP communications / fund updates / allocator outreach
- **Hierarchy for investment-related entities**: (1) If the sender is **explicitly raising venture capital** — asking Tom to write an equity/investment check into their company or vehicle — that IS a deal regardless of entity type (VC firm, family office, fund, asset manager, anything). (2) Only if you cannot determine they are raising venture capital, apply these exclusions: other venture capital firms reaching out (co-invest pitches, deal flow sharing, fund intros); LP-type entities such as fund of funds, family offices, and endowments.

**Field formatting:**
- `round_details` — **strict format, two valid shapes only**: `"Raising $Xm"` / `"Raising $X-Ym"` for unfinalized rounds (no terms set), OR `"$Xm on $Ym post"` / `"$Xm on $Ym cap"` for rounds with terms set. Lowercase `m`/`k`. **Return empty string `""` if neither a dollar amount nor a cap/post is explicit in the source.** Never write timing-only or qualitative text like `"Kicking off seed this week"`, `"Raising soon"`, `"Active round"`, `"Closing this month"` — those are not round details and will be rejected by `add-to-crm`. If the source has a deck attached, the deck is part of the source — `add-to-crm`'s Step 1B reads decks for round disclosure before writing the field. Empty here is fine; the downstream skill will fill in from the deck if available.
- `stage` — one of `Pre-Seed`, `Seed`, `Seed+`, `Series A`, `Series B`, `Growth`, `Angel` — infer from round size or explicit mention.
- Use empty strings for fields you cannot extract.
- Prefer `"low"` confidence for ambiguous cases — false negatives are cheaper than false positives here.

### Step 3: Gate on classification

- `is_deal: false` → log `not-deal` with the reason and exit 0.
- `is_deal: true` AND `confidence: low` → log `low-confidence-skip` and exit 0. (Tom would rather miss a deal than create a noisy entry.)
- `is_deal: true` AND `confidence: medium`/`high` AND `company` is empty → log `no-company-extracted` and exit 0.
- `is_deal: true` AND `confidence: medium`/`high` AND `company` populated → proceed to Step 4.

### Step 4: Delegate to `add-to-crm`

Read `~/.claude/skills/add-to-crm/SKILL.md` and run it with the email as input. The `add-to-crm` skill already handles:

- Duplicate detection against the Opportunities DB (including the **Protected Status Guard** for terminal statuses — Active Portfolio, Portfolio: Follow-On, Exited, Committed)
- Hard-block on prior `Pass *` statuses
- ContactOut / web enrichment for HQ, website, contact email, logo
- Page creation with full property mapping
- Materials handler for any deck/memo attachments

Pass these inputs explicitly so `add-to-crm` doesn't have to re-classify:

- The full email body (subject, sender, plaintext body)
- The classifier's extracted fields as **hints** (company, description, founder name, founder LI URL, website, round_details, stage). These are starting points — `add-to-crm`'s enrichment step may override them with better data from ContactOut / company website.
- **Explicit `status` directive**: `"Connected"` if Step 1B / 1C resolved Source = Direct (founder is the actual sender), `"Qualified"` if Source = <referrer> (intro offered but not made). `add-to-crm` MUST honor this directive — do not let downstream inference flip it back to `Connected` for a referrer-forward case.
- **Explicit `source` directive**: either `"Direct"` (use the canonical Direct People DB page) OR `{ email: <senderEmail>, name: <senderName> }` for a referrer. `add-to-crm` resolves the referrer against the People DB and populates `Source(s)`. If the referrer is not in the People DB, leave `Source(s)` blank and surface the gap in the Slack alert — do NOT auto-create.
- The Gmail message URL (`https://mail.google.com/mail/u/0/#inbox/{messageId}`) for the Source Email field, and the `messageId` itself so it can probe attachments via the Gmail Attachment Saver per the skill's Step 1B.
- The Gmail `threadId` for the new `Source Thread ID` property on the Opp.
- **Explicit `materialUrls` directive** (if non-empty): pass the array verbatim to `add-to-crm`. The skill treats this as the authoritative list of deck/material URLs that MUST be processed by Step 1B (read for thin-body enrichment) and linked via Step 6 (Diligence Materials property). The webhook ran the regex; the skill doesn't get to second-guess what counts as a deck URL.

**Unattended mode:** This skill is invoked by an automated webhook job — do **not** ask Tom for confirmation at any point. If `add-to-crm`'s "Step 4: Present extracted fields to user" would normally pause for confirmation, skip the pause and create the page directly with the classifier-extracted fields. If a duplicate is found in a protected terminal status, log `protected-status-skip` with the existing page ID and exit 0 — do **not** prompt.

### Step 5: Report to Slack

After `add-to-crm` returns (success or skip), post a single Slack alert via the `send-alert` skill (read `~/.claude/skills/send-alert/SKILL.md`).

**Alert format (one of):**

- New entry created — `🆕 **<Company>** — "<subject>" — <one-liner description>. <Notion page link>`
- Duplicate (in pipeline, not protected) — `🔁 **<Company>** — already in pipeline (<status>). <existing Notion page link>`
  - **Exception: suppress this alert silently (exit 0, no Slack post) when the existing Opp's status is any active state: Outreach, Connected, Scheduled, Active, or Committed.** These are deals already in motion — the 🔁 is noise. Only post 🔁 for stale states (Qualified, Track, Pass Note Pending) where a resurface is worth knowing.
- Hard-block (prior pass) — `⛔ **<Company>** — prior pass on file. <existing Notion page link>`
- Protected status — `🛡️ **<Company>** — already <status> (protected). <existing Notion page link>`

Header line: `📥 NEW DEAL — YYYY-MM-DD HH:MM ET`. One alert per webhook job — never batch.

### Step 6: Exit

Exit 0 on any successful path (created, skipped, deduped). Exit non-zero only on infrastructure errors (Gmail fetch failed, Notion API down, `add-to-crm` raised). The processor moves non-zero jobs to `failed/` and posts a Slack failure alert from the queue layer.

## Notes

- **Idempotency:** the webhook keys the job by `messageId` (`idempotencyKey: 'inbound-deal-detect-' + messageId`), so re-deliveries from Gmail Pub/Sub are deduped at the queue layer. The skill itself does not need its own dedup beyond `add-to-crm`'s existing duplicate check.
- **Founder-sender exclusion is now this skill's job.** The webhook used to skip emails whose sender matched a portfolio founder, but the heuristics (Contact-field substring, People→Founder relation) misfired in both directions — referrers tripped the substring check, and Opps with no Founder relation leaked through. Removed 2026-05-06. The classifier's not-deal rubric ("Founder update on an existing portfolio company") now gates this entirely; rely on it instead of pre-screening on the sender.
- **No People DB row creation** for the founder (per Tom's standing rule) — `add-to-crm` already honors this.
