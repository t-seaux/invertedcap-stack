---
name: inbound-deal-detect
description: "Webhook-triggered classifier for inbound cold deal emails. Fetches a single Gmail message by ID, runs a deal-vs-not-deal classifier with confidence gating, and on a confident positive delegates to `add-to-crm` for full pipeline entry. Not user-facing — invoked exclusively by `gmail-webhook/deal-scanner.js` via the `claude-job-queue` primitive. Never trigger manually; for ad-hoc CRM creation use `add-to-crm` directly."
---

# Inbound Deal Detector (Webhook)

Classifies a single inbound Gmail message as a new deal or not, then hands off to `add-to-crm` if it's a deal.

This skill is the deeper-classification half of `gmail-webhook/deal-scanner.js`. The webhook does a cheap deterministic pre-filter (INBOX, no DRAFT, no SENT-unless-self-forward, no noreply) and then a Haiku gate (`classifyWithLLM`) that decides `is_deal` + `is_third_party_forward` + extracts the forwarded founder candidate. On positive, it enqueues a job here. This skill does the expensive part: re-fetch the full email, run the canonical classifier rubric (with attachment list, Source attribution, field extraction), and on positive delegate to `add-to-crm`.

## Args

The local processor invokes this skill with these args (set by `deal-scanner.js`):

- `messageId` (required) — Gmail message ID.
- `senderEmail` (required) — pre-parsed by the webhook. For non-forwarded mail this is the outer `From:` header. For self-forwards (see `forwardedFromTom`) and third-party forwards (see `forwardedFromReferrer`), this is the **inner** forwarded `From:` line — the original founder or referrer who sent to Tom, not Tom and not the outer envelope.
- `senderName` (optional) — display name from the same source.
- `forwardedFromTom` (optional, boolean) — `true` when the message is a `Fwd:` from one of Tom's alias addresses (e.g. `tom@dashfund.co`) into the watched inbox. The webhook has already swapped `senderEmail`/`senderName` to the inner forwarded sender. The classifier should treat the forwarded body block as the canonical email content and infer Source from the inner `To:` header (see Step 1B).
- `forwardedFromReferrer` (optional, boolean) — `true` when the message is a `Fwd:` from an external third party (not Tom, not the founder) who is forwarding the founder's email along. The webhook has already swapped `senderEmail`/`senderName` to the inner forwarded sender (the founder candidate). `referrerEmail` / `referrerName` carry the outer envelope (the referrer). See Step 1C.
- `referrerEmail` (optional) — outer envelope email when `forwardedFromReferrer` is true.
- `referrerName` (optional) — outer envelope display name when `forwardedFromReferrer` is true.

## Workflow

### Step 1: Fetch the email

Read the full message via Gmail MCP (`mcp__claude_ai_Gmail__search_threads` for thread context, or read the single message by ID). Grab:

- Subject
- Plain-text body (strip quoted history below the `On … wrote:` line — only the new content matters for classification)
- Confirmed `From` (verify it matches `senderEmail` arg; if not, log and prefer the actual header)
- Any attachment filenames
- **`threadId`** — Gmail's conversation identifier (same field as `messageId` in the Gmail API response). Pass through to add-to-crm in Step 4 so it can write `Source Thread ID` on the new Opp page. This enables `outreach-detector` / `outreach-decliner` Path D (deterministic thread-based status flips on Tom's outbound replies).

If the fetch fails, log to the run log and exit non-zero so the job lands in `failed/` for retry.

### Step 1B: Forwarded-from-Tom handling

If `forwardedFromTom` is true, the outer envelope is just Tom forwarding the email to himself. The real content lives inside the forwarded block (everything after `Begin forwarded message:` / `---------- Forwarded message ----------`). For classification purposes:

- Treat the forwarded block's `From:`, `To:`, `Subject:`, and body as the canonical email — that's what was actually sent to Tom.
- The classifier should ignore Tom's outer `Fwd:` subject prefix and any cover note Tom wrote when forwarding (usually empty).

For Source attribution when delegating to `add-to-crm`:

- If the forwarded `To:` is one of Tom's alias addresses (`tom@dashfund.co`, `thomas.seo@outlook.com`, `tom@invertedcap.com`) → **Source = Direct** (founder pinged Tom directly at his alternate inbox).
- Otherwise the forwarded `To:` is a referrer who passed the email along to Tom → **Source = <Referrer Name>** (the referrer's name, derived from the forwarded `To:` field; if unavailable, use the email-local-part).

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
- Other investment firms (VC funds, hedge funds, family offices) pitching coinvestment or fund intros — peer firms, not startups

**Field formatting:**
- `round_details` — `"Raising $3m"` or `"Raising $3-5m"` for unfinalized rounds; `"$3m on $12m post"` or `"$3m on $12m cap"` for finalized rounds. Lowercase `m`/`k`.
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
- The Gmail message URL (`https://mail.google.com/mail/u/0/#inbox/{messageId}`) for the Source Email field, and the `messageId` itself so it can probe attachments via the Gmail Attachment Saver per the skill's Step 1B.
- The Gmail `threadId` for the new `Source Thread ID` property on the Opp.

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
