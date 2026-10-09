---
name: inbound-deal-detect
description: "Webhook-triggered classifier for inbound cold deal emails. Fetches the target thread headlessly (`admin_run.py _readThread`, per shared-references/headless-gmail.md) and locates it by `messageId`, runs a deal-vs-not-deal classifier with confidence gating, and on a confident positive delegates to `add-to-crm` for full pipeline entry. Not user-facing — invoked exclusively by `gmail-webhook/deal-scanner.js` via the `claude-job-queue` primitive. Never trigger manually; for ad-hoc CRM creation use `add-to-crm` directly."
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
- `forwardedFromTom` (optional, boolean) — `true` when the message is a `Fwd:` from one of Tom's alias addresses into the watched inbox. Tom's known addresses are `tom@invertedcap.com`, `tom@dashfund.co`, `thomas.seo@outlook.com`, and `tseo@primary.vc` (his Primary Venture Partners address — founders often pitch him there and he forwards those in). The webhook has already swapped `senderEmail`/`senderName` to the inner forwarded sender. The classifier should treat the forwarded body block as the canonical email content and infer Source from the inner `To:` header (see Step 1B). When the inner `To:` is any of Tom's addresses above, the pitch was sent directly to Tom.
- `forwardedFromReferrer` (optional, boolean) — `true` when the message is a `Fwd:` from an external third party (not Tom, not the founder) who is forwarding the founder's email along. The webhook has already swapped `senderEmail`/`senderName` to the inner forwarded sender (the founder candidate). `referrerEmail` / `referrerName` carry the outer envelope (the referrer). See Step 1C.
- `referrerEmail` (optional) — outer envelope email when `forwardedFromReferrer` is true.
- `referrerName` (optional) — outer envelope display name when `forwardedFromReferrer` is true.
- `knownTerminalOpp` (optional, object `{id, name, status, matchedVia?}`) — set by `deal-scanner.js` when the envelope sender (or, via `matchedVia`, another address under the same display name — see `~/.claude/skills/shared-references/revive-gate.md`) is the `Contact` of an existing Opp at a **terminal** status (`Pass (Met)`, `Pass (DNM)`, `Lost`, `NR / Missed`). The webhook skips its Haiku gate for these (a known founder re-engaging is signal by definition) and routes here so the Revive Gate v2 runs. See Step 3 — the update-vs-pitch and confidence gates do NOT apply; the only question is whether the email carries new company signal.
- `knownLiveOpp` (optional, object `{id, name, status, matchedVia?}`) — set by `deal-scanner.js` when the envelope sender's OWN mail (never Tom, never a forward of someone else's pitch) resolves to an Opp at a **live-update** status (`LIVE_UPDATE_STATUSES` = `Track` only — at every other live status Tom is in the thread himself) by Source Thread ID, Contact / rebrand alias, or — when every email + thread lookup misses — the sender's exact full display name in a `-1 (<Name>)` / `NewCo (<Name>)` title (`matchedVia: {kind: "name-title", displayName}`). The webhook skips its Haiku gate and the cold-follow-up gate for these (Greg Reiner, 2026-10-05: a Track founder wrote "found a cofounder, quit Meta, pilot starting – not raising yet, coffee?", Haiku said not-deal, Tom heard nothing: *"This an exist opp. Why didn't u flag"*). Never together with `knownTerminalOpp` (`idd_gate.py` exit 2). See Step 3 — the 📈 live-Opp update path.
- `introConnected`, `introBranch`, `introStage`, `introOldStage`, `introFounderEmail`, `introCorroboration` (optional; only WITH `knownTerminalOpp`) — set by `intro-connected-detect.js` for EVERY corroborated three-way intro on a passed Opp, same round (`introBranch:"reopen"`, `introStage` the stated Stage or `""`) or new raise (`"new-row"`) (`revive-gate.md` § "Intro path"). `introConnected` already proves new company signal — `idd_gate.py` needs no `new_company_signal` then, checks `introBranch` against `revive_branch` (exit 2 on drift), passes the keys to add-to-crm, sets the Stage hint to `introStage` when one was stated and the status to `Connected`. Classify as usual; don't second-guess the Stage.
- `materialUrls` (optional, array of strings) — deck/material URLs the webhook extracted from the email body (Drive, DocSend, Dropbox, Brieflink, Pitch.com, Figma, Canva, Notion.site, raw PDFs). When present, this list is **authoritative**: every URL MUST be passed through to `add-to-crm` so it runs Step 1B (read for thin-body field extraction) and Step 6 (link in Diligence Materials property). Skipping a URL because "the body context didn't seem deck-shaped" is not allowed — the webhook already filtered out company-website links. See the 2026-05-12 Unicorn Snot regression for why this gate moved server-side.

### Dash lane (`mail_source: "dash-local"`)

Since 2026-10-01 the gmail-webhook routes tom@dashfund.co mail through the same deal-scanner, so
this skill also receives Dash jobs: args carry `mail_source:"dash-local"`, `rowid` (= the Dash
Gmail message id, same value as `messageId`) and `fund:"Inverted 1️⃣"` (new deals are never Dash).
Deltas — everything else is identical:
- **Step 1 fetch:** `python3 ~/.claude/scripts/dash_mail.py get "<rowid>"` (body, headers,
  `message_id`, `threadId`) — NOT `admin_run.py` (Inverted-only, per `headless-gmail.md` H4).
  Attachments: `dash_mail.py attachments "<rowid>" <dir>`.
- **Step 1D sender gate:** also drop anything sent from `@dashfund.co` (Tom / Ryan internal mail).
- **Step 4 args:** pass `mail_source`, `rowid`, `fund` through to add-to-crm and OMIT
  `threadId` / `gmailMessageUrl` — add-to-crm's Dash lane stages the card with them (Step 4T).
- Ledger/audit: `.proposed` lines go to this skill's `.proposed`, tagged `dash rowid=<rowid>`;
  pre-cutover Dash ledgers are archived read-only in `dash-history/`.

## Workflow

### Step 1: Fetch the email

**⛔ Headless Gmail — follow `shared-references/headless-gmail.md` H1.** Fetch with `cd ~/code/gmail-webhook && python3 admin_run.py _readThread <threadId>` and locate the message whose `messageId` matches the arg; never conclude Gmail is unavailable from the MCP's absence (2026-09-26 Rexi miss). Grab:

- Subject
- Plain-text body (strip quoted history below the `On … wrote:` line — only the new content matters for classification)
- Confirmed `From` (verify it matches `senderEmail` arg; if not, log and prefer the actual header)
- Any attachment filenames
- Pass `threadId` through to add-to-crm in Step 4 so it can write `Source Thread ID` on the new Opp page. This enables `outreach-detector` / `outreach-decliner` Path D (deterministic thread-based status flips on Tom's outbound replies).

If the fetch fails (thread not found, target messageId missing inside thread), log to the run log AND append a line to `~/.claude/skills/inbound-deal-detect/audit-log/YYYY-MM-DD.log` via `mkdir -p ~/.claude/skills/inbound-deal-detect/audit-log && echo "[$(date '+%F %T')] FETCH_FAILED: <reason> (messageId: <id>, threadId: <tid>)" | tee -a ~/.claude/skills/inbound-deal-detect/audit-log/$(date '+%F').log`, then end your final output with `JOB_FAILED: FETCH_FAILED <reason>` per `shared-references/headless-gmail.md` H3 (prose "exit 1" does nothing). The literal `tee -a` shell line must run — never claim the write happened without executing it.

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

**Direct-inbound override (run FIRST).** Before trusting `forwardedFromReferrer`, apply the Step 1B domain discriminator: compare the inner forwarded `From:` domain against the pitched company. **If they match, this is NOT a referral — it is a cold inbound founder email that reached Tom directly** (the founder wrote to one of Tom's addresses and Tom self-forwarded it; the "referrer" the webhook flagged is really Tom's own alias, e.g. `tseo@primary.vc`). Treat exactly like Step 1B's match case: **Source = Direct, Status default = `Connected`** — ignore `referrerEmail`/`referrerName`. Only fall through to the referrer attribution below when the inner sender's domain does NOT match the pitched company. **Exception – a real third-party referrer:** when `referrerEmail` is NOT one of Tom's addresses (e.g. an investor forwarding a founder's email that was sent to *them*), the domain match only proves who the founder is, not that they wrote to Tom → referrer attribution below (Source = referrer, Status = `Qualified`). `idd_gate.py` enforces this (Inviscid via Laura Bock, 2026-10-08).

For Source attribution when delegating to `add-to-crm` (referrer case only — inner domain does NOT match the company):

- **Source = `referrerName`** (the outer envelope's display name). If `referrerName` is empty, use the local-part of `referrerEmail`.

### Step 1D: Plain inbound, no forward flags — envelope-sender gate (MANDATORY)

When BOTH `forwardedFromTom` and `forwardedFromReferrer` are false, the envelope sender of the target message (= the webhook's `senderEmail`) is the person who actually wrote to Tom. Run the same domain discriminator directly on THEM before writing any directive:

- **Envelope sender's domain matches the pitched company** (or the sender is the named founder writing from a personal address) → `sourceDirective: "Direct"`, `statusDirective: "Connected"`.
- **Envelope sender's domain does NOT match the pitched company** → the sender is a referrer/sharer (an investor bouncing a deal blast, an "any interest?" note, an intro offer) **even when the webhook failed to set `is_third_party_forward`** — the webhook's Haiku gate is a hint, not the authority; this gate is. `sourceDirective: { email: <envelope senderEmail>, name: <envelope senderName> }`, `statusDirective: "Qualified"`. Escalate per `~/.claude/skills/shared-references/opp-status-sets.md` § "Status inference for an inbound" (the ONE rule — observed events only: referrer facilitating after Tom's opt-in → `Outreach`; Tom saying he'll reach out himself stays `Qualified`; founder actually in the thread with Tom → `Connected`).

**HARD RULE (New Issue IQ incident, 2026-08-21):** in this path, `senderEmail`/`senderName` in the add-to-crm args must remain the ENVELOPE sender. Never substitute a founder email or name extracted from the body — the founder's contact belongs in the classifier fields (`founder_*`, and downstream in the Opp's Contact), never in `senderEmail` or `sourceDirective`. A quoted thread below the sender's note (Outlook-style `From:` blocks, Diadem-style deal blasts) does NOT make the quoted author the sender. `sourceDirective: "Direct"` may only be written when the envelope-domain match above is affirmatively proven; if you cannot prove it, it is a referrer. That morning's failure: Luis Valdich (viminvestors.com) forwarded a Diadem blast about New Issue IQ; the run swapped `senderEmail` to `john@newissueiq.com` and stamped Direct/Connected — wrong on both counts (correct: Source = Luis's People row, Status = Qualified, intro merely offered).

### Step 2: Classify

Apply the classifier rubric below. Return a single JSON object — **no markdown fences, no commentary**:

```json
{
  "is_deal": true | false,
  "is_update_not_pitch": true | false,
  "confidence": "high" | "medium" | "low",
  "reason_if_not_deal": "string",
  "companies": [
    {
      "name": "string",
      "description": "string",
      "founder_first_name": "string",
      "founder_li_url": "string",
      "website": "string",
      "round_details": "string",
      "stage": "string",
      "material_urls": ["string", ...],
      "material_urls_ambiguous": true | false,
      "investor_referrer": true | false,
      "sender_is_founder": true | false,
      "forward_roles": {"referrer_is_founder": true | false, "referrer_email": "string", "referrer_name": "string"},
      "status_escalation": "" | "Outreach" | "Connected" | "Scheduled"
    }
  ],
  "new_company_signal": true | false,
  "live_update": {"news": "string", "update": "string", "wants_meeting": true | false}
}
```

The last four fields are the judgment inputs `idd_gate.py` (Step 3) cannot compute: `investor_referrer` = the sender is a VC/investor pitching this company (medium-gate signal 4); `sender_is_founder` = the sender is the named founder writing personally (free-mail, or another custom domain when the sender's first name is the founder's – 2026-10-05); `forward_roles` = REQUIRED when the args carry `forwardRoleBasis: "domain-diff"` (a 2-level forward whose roles the scanner only guessed from domains): read `forwardChain` + the body – `referrer_is_founder: true` when the outer forwarder is the founder themselves (old → new address), else the real referrer's `referrer_email` / `referrer_name`; omit otherwise; `status_escalation` = an OBSERVED event per opp-status-sets.md § "Status inference for an inbound" (Tom opted in → `Outreach`; founder in the thread → `Connected`), else `""`; `new_company_signal` = REQUIRED whenever `knownTerminalOpp` OR `knownLiveOpp` is set (the revive / live-update question below), omit otherwise. `live_update` = REQUIRED when `knownLiveOpp` is set and `new_company_signal` is true, omit otherwise: `news` = ONE line of what's new, facts only from this email (no period, no Opp name — the card adds both), `update` = 1–2 sentences for the Opp body, facts only from this email (who/what/when as written; no inference, no adjectives the founder didn't use), `wants_meeting` = the email asks Tom for a meeting / call / coffee. En dashes, never em dashes (the code swaps any `—` it gets).

**`companies` is always an array.** A single-company pitch (the common case) returns a one-element array. A multi-company digest (e.g. David Talpalar's "A Few Interesting Ones" — one email pitching 4 distinct startups) returns one element per company. The sender of a multi-company digest is the Source/referrer for ALL of them (no founder can write a digest about 4 separate startups); Step 4 applies that convention without needing to be told.

**Default to single-company.** Most inbound emails pitch ONE company. Only return `companies.length >= 2` when the body **explicitly enumerates distinct companies with separate identities** — each one has its own name, separate website OR separate founder, and separate fundraising context (round size, stage, or distinct pitch sentence). Signals that indicate multi-company digest:
- Multiple distinct company URLs (different domains, not subpaths of one)
- Multiple distinct founder names attached to different products
- The sender frames the email as a list ("a few interesting ones", "wanted to flag these companies", "three startups in your space", a bulleted list of companies)
- Each company has its own 1–3 sentence blurb in the body

Signals that are NOT multi-company (return single-element array):
- A founder mentioning their previous company in their own pitch ("ex-CTO of Stripe, now building Acme")
- A founder mentioning competitors ("we're like Plaid for X")
- An investor mentioning their portfolio in passing ("similar to my investment in Mercury")
- A single founder pitching a multi-product platform

When in doubt, return one company. False fan-out creates ghost Opps Tom has to clean up — false collapse just means he uses `batch-add-to-crm` manually.

**`material_urls` partitioning:** the args dict's top-level `materialUrls` arrives flat from the webhook (every deck/memo URL extracted from the body). When emitting `companies`, partition that list per-company — assign each URL to the company it visually clusters with in the body (e.g. a `simpleproduct.dev/share/...` memo URL belongs to the Simple Product entry, not Lucius). URLs whose target company is ambiguous go in the FIRST listed company's bucket — Tom will reassign manually if wrong — AND set `material_urls_ambiguous: true` on that company's entry (default `false`). Step 4 copies the flag into that company's `classifierHints` so downstream `add-to-crm` surfaces the ambiguity in its Slack alert. Companies with no associated URL get an empty array.

**Update-vs-pitch sub-question (answer BEFORE setting `is_deal`):** Is this a founder UPDATE (progress report, intro ask, feedback request) or a NEW PITCH (seeking investment)? Only a NEW PITCH classifies as a deal. If the email is an update, set `is_update_not_pitch: true` and `is_deal: false` — Step 3 logs the skip and never enqueues. Set `is_update_not_pitch: false` otherwise.

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
- **Hierarchy for investment-related entities**: (1) If the sender is **explicitly raising venture capital** — asking Tom to write an equity/investment check into their company or vehicle — that IS a deal regardless of entity type (VC firm, family office, fund, asset manager, anything). (2) An investor or VC firm introducing or pitching a **specific named company for investment** IS a deal — co-investment offers on a named deal ("want to chat with X, now raising their seed"), "our portfolio company X is raising", a single-company share pitching the startup itself. An explicit round size/terms is NOT required — the intro of one named investable company is the signal; round details or a deck merely raise confidence. The sender is a referrer, exactly like a third-party referral forward (see Source attribution in Step 4). (3) Only when neither (1) nor (2) applies, exclude: reciprocal deal-flow/pipeline sharing (a dealflow LIST shared for reciprocal sourcing, "send us anything raising and we'll do the same" — that's `log-deal-share` territory), fund intros and co-invest relationship building with no specific company raising; LP-type entities such as fund of funds, family offices, and endowments.

**Field formatting (apply per-company within the `companies` array):**
- `round_details` — per `~/.claude/skills/shared-references/round-details-format.md` (the ONE spec). Return `""` if no concrete $ figure is explicit in the source; add-to-crm reads the deck downstream.
- `stage` — one of `Pre-Seed`, `Seed`, `Seed+`, `Series A`, `Series B`, `Growth`, `Angel` — ALWAYS one of these, never empty: an explicitly named round (email or deck) wins; otherwise infer from the whole context (company age, first-ever raise), not round size alone — a new company's big first round is Pre-Seed. Stage / HQ / Description provenance = `~/.claude/skills/shared-references/deal-card-provenance.md` (ONE spec, both lanes): no WebSearch / aggregator snippets.
- Use empty strings for fields you cannot extract.
- Prefer `"low"` confidence for ambiguous cases — false negatives are cheaper than false positives here.

### Step 3: Gate on classification

> **As code (2026-10-04).** Steps 3 and 4's mechanics run in `idd_gate.py` — write the Step 2 JSON to `/tmp/idd-cls-<messageId>.json` and this job's args verbatim to `/tmp/idd-args-<messageId>.json`, then:
> ```bash
> python3 ~/.claude/skills/inbound-deal-detect/idd_gate.py gate --classifier /tmp/idd-cls-<messageId>.json --args /tmp/idd-args-<messageId>.json
> ```
> - **exit 0** → it wrote one `/tmp/addcrm-args-<messageId>-<slug>.json` per surviving company (paths in stdout `files`); run Step 4's helper on each. Do NOT hand-write or edit those files.
> - **exit 1** → gated skip: log stdout `log` verbatim; run blurb capture; post the Step 5 alert only if `slack_alert` is true; exit 0.
> - **exit 2** → bad input (missing `messageId`; `knownTerminalOpp` or `knownLiveOpp` set without a boolean `new_company_signal`; `knownLiveOpp` + signal without a usable `live_update`; `knownLiveOpp` together with `knownTerminalOpp`) → `JOB_FAILED: IDD_GATE <log>`.
> - **exit 3** → 📈 live-Opp update: it wrote ONE `/tmp/live-update-<messageId>.json` (paths in `files`). Run `python3 ~/.claude/skills/inbound-deal-detect/live_update.py apply --plan <file> --apply`, log stdout verbatim, exit 0 (non-zero from live_update.py → `JOB_FAILED: LIVE_UPDATE <error>`; exit 1 = a benign skip — status moved / already applied — log it, exit 0). Never enqueue add-to-crm, never hand-write the plan, never text or write Notion yourself.
>
> The code owns: the gate order below, the medium 2-of-4 evidence count, empty-name drops, the revive routing, the Source/Status directive (Direct only when PROVEN — sender domain == company website domain, or a free-mail sender with `sender_is_founder`; digests are always referrer; `senderEmail` is copied verbatim from the envelope, never from the body — New Issue IQ), and the name slug. The prose below is the WHY and the judgment you feed it.

- **`knownTerminalOpp` present → the revive path, BEFORE any of the gates below (Tom, 2026-09-22, Solid Credit).** A founder whose Opp Tom already closed out is writing in again. Ignore `is_deal` / `is_update_not_pitch` / `confidence`; ask only: does the email carry **new company signal** — a raise starting or planned, a deck or materials (attachment or `materialUrls`), a pivot, traction, or an ask for Tom's time *about the company*? If YES → enqueue `add-to-crm` (Step 4) with the extracted fields plus `knownTerminalOpp` passed through verbatim; add-to-crm's Protected Status Guard runs the Revive Gate (`revive-gate.md`: new raise → new linked row on 👍; same round → enrich the existing row now + 🔁 card for the status flip). Log `terminal-contact-revive-route opp=<name> status=<status>`. If NO (pure scheduling, social, thanks) → log `terminal-contact-no-signal` and exit 0. Never mint a new Opp for this sender.
- **`knownLiveOpp` present → the 📈 live-Opp update path, BEFORE any of the gates below (Tom, 2026-10-05, Greg Reiner).** A founder Tom has met, whose Opp sits at `Track`, is writing in. Ignore `is_deal` / `is_update_not_pitch` / `confidence` ("not raising right now" is exactly the news Tom wants). Ask only: does the email carry **new company signal** — left a job, found a cofounder, a pilot / launch, traction, a raise planned or starting, materials, or an ask for Tom's time *about the company*? If YES → fill `live_update` (Step 2); `idd_gate.py` exits 3 and `live_update.py` does all of it: (a) prepends a dated `## Update — YYYY-MM-DD` section to the Opp body (1–2 sentences + an `Email` link), (b) backfills `Contact` with the sender's email when Contact is blank / `N/A` AND the Opp is a `-1` / `NewCo` row titled with the sender's name (named-company Opps: the webhook's canonical Contact writer owns those), (c) texts Tom ONE 📈 card (`+12012567714`) — headline `📈 Opp Update: <Opp>`, a `⚠ Wants to meet – once it's on the calendar it moves to Scheduled` line when the email asks to meet, `Status: <status>`, the news line, `✓ Added to the Opp`, the email link. If NO (pure scheduling, social, thanks) → log `live-contact-no-signal opp=<name> status=<status>` and exit 0. **NEVER change Status, NEVER create an Opp, never add-to-crm.** A meeting that gets booked moves `Track → Scheduled` on its own via `calendar-scheduled-detect` (it matches attendees against `Contact`, which is why (b) matters). Blurb capture still applies on the no-signal skip.
- **Blurb capture (every skip exit in this step):** before exiting, run `~/.claude/skills/shared-references/blurb-capture.md`. If the sender maps to an existing Opp AND the email carries a founder / intro-er blurb, enqueue `log-company-blurb` headless. That skill logs the blurb verbatim and alerts Tom. (Thermis, 2026-09-24: Emily's forwardable blurb was classified not-deal and dropped.)
- `is_deal: false` → log `not-deal` with the reason and exit 0.
- `is_update_not_pitch: true` → log `update-not-pitch-skip` and exit 0, regardless of confidence. Founder updates route through `investor-update`, never through add-to-crm.
- `is_deal: true` AND `confidence: low` → log `low-confidence-skip` and exit 0. (Tom would rather miss a deal than create a noisy entry.)
- `is_deal: true` AND `confidence: medium` → secondary evidence gate: a medium-confidence verdict proceeds only if at least TWO of these four signals are present in the extraction: (1) a named founder AND company name, (2) an explicit round amount (`round_details` non-empty), (3) a founder LinkedIn URL (`founder_li_url` non-empty), (4) an identified investor referrer pitching the company (sender is a VC/investor whose domain is a fund, not the pitched company — hierarchy rule (2) in the rubric; investor intros often omit round size, so this signal substitutes for it). For multi-company emails, apply per-company and drop entries that fail. If no company clears the gate, downgrade to skip — log `medium-confidence-evidence-skip` listing which of the signals were missing — and exit 0.
- `is_deal: true` AND `confidence: medium`/`high` AND `companies` is empty OR every entry's `name` is empty → log `no-company-extracted` and exit 0.
- `is_deal: true` AND `confidence: medium`/`high` AND at least one `companies[].name` populated → proceed to Step 4. Drop any per-company entry whose `name` is empty before looping.

### Step 4: Enqueue one `add-to-crm` job per company

This skill runs on **Haiku** (per Tom's model tier framework — it's a classifier). The downstream `add-to-crm` work — Notion dedup queries, ContactOut/web enrichment, page creation with full property mapping, materials handling — is Sonnet-class. Splitting the two via the queue keeps each tier doing what it's good at and was the structural fix for the Evalion 2026-06-01 failure mode (Haiku narrated `(would execute)` instead of running add-to-crm inline).

Do NOT read `add-to-crm/SKILL.md` or attempt to execute its steps inline. Instead, write a typed args JSON file per company and invoke the canonical helper `~/.claude/scripts/enqueue-addcrm.sh` — it wraps the args in the queue envelope, computes the idempotency key itself (`-<slug>` of `classifierHints.company`; the file's `idempotencySuffix` is not trusted), and POSTs to `/enqueue`. The helper exists so Haiku doesn't have to template a mixed-shape JSON inline (bools, arrays, mixed-shape `sourceDirective`).

#### Source attribution for multi-company digests

Before the loop, decide the `sourceDirective` shape. The rule applied per-company:

- **Single-company email** (`companies.length === 1`) AND not a forward → check WHO is pitching before defaulting:
  - Sender is the founder (sender domain matches the pitched company, or the sender is the named founder writing from a personal address) → `"Direct"`.
  - Sender is an investor/VC pitching someone ELSE's company (hierarchy rule (2): co-invest intro, "our portfolio company X is raising" — sender domain is a fund, founder is a named third party) → `{ email: senderEmail, name: senderName }`, Status default = `Qualified` (the intro is offered but the founder is not yet in the thread with Tom). If Tom has already replied opting in, escalate to `Outreach`; if the founder is on the thread, `Connected`.
- **Multi-company digest** (`companies.length >= 2`) → `{ email: senderEmail, name: senderName }`. The sender of a digest is by definition a referrer: no founder writes a digest pitching 4 separate startups. Apply this even when `forwardedFromTom`/`forwardedFromReferrer` are false — the multi-company shape itself is the signal.
- **Forwarded email** (any company count, `forwardedFromTom` true with inner-domain mismatch per Step 1B, OR `forwardedFromReferrer` true) → `{ email: referrerEmail, name: referrerName }` for every per-company job. Step 1B/1C source resolution overrides the multi-company rule above.
- **Self-forward, inner-domain matches pitched company** (Step 1B), single-company → `"Direct"`, Status default = `Connected`. (Multi-company self-forwards where the inner sender matches one of the companies — exceedingly rare — still use the digest rule above; mark that one company `"Direct"` if you can and the rest `{ email: senderEmail, ... }`.)

#### Loop

For each company in `companies[]` (use the array index `i`, zero-based):

1. `idd_gate.py gate` (Step 3) has already written `/tmp/addcrm-args-<messageId>-<slug>.json` per company with the shape below — reference only; never hand-template it. Use exact JSON types — booleans unquoted, arrays as arrays, `sourceDirective` either a string OR an object:

```json
{
  "webhookMode": true,
  "messageId": "<messageId arg, verbatim>",
  "threadId": "<threadId arg, verbatim>",
  "gmailMessageUrl": "https://mail.google.com/mail/u/0/#inbox/<messageId>",
  "senderEmail": "<senderEmail arg, verbatim — webhook already swapped for forward cases>",
  "senderName": "<senderName arg, verbatim>",
  "classifierHints": {
    "company": "<companies[i].name>",
    "description": "<companies[i].description>",
    "founder_first_name": "<companies[i].founder_first_name>",
    "founder_li_url": "<companies[i].founder_li_url>",
    "website": "<companies[i].website>",
    "round_details": "<companies[i].round_details>",
    "stage": "<companies[i].stage>",
    "material_urls_ambiguous": <companies[i].material_urls_ambiguous — true only when an ambiguous URL was bucketed here>
  },
  "statusDirective": "<Connected | Qualified | Outreach | Scheduled — per Step 1B/1C; default Qualified for multi-company digests>",
  "sourceDirective": "Direct" | { "email": "...", "name": "..." },
  "materialUrls": ["<companies[i].material_urls — per-company slice>"],
  "forwardedFromTom": false,
  "forwardedFromReferrer": false,
  "idempotencySuffix": "",
  "batchContext": null
}
```

  - **As code:** `enqueue-addcrm.sh` now IGNORES any `idempotencySuffix` in the file and recomputes `-<slug>` from `classifierHints.company` via `idd_gate.py slug` (the only exception is add-to-crm-detect's literal `-cmd`); no company + no `-cmd` → exit 2.
  - **`idempotencySuffix` is ALWAYS `"-<slug>"` derived from the company NAME — never from the loop index, and never omitted.** Slug = company name lowercased, non-alphanumeric runs collapsed to a single hyphen, trimmed to 40 chars (`"Acme AI, Inc."` → `"acme-ai-inc"`). Apply this identically whether there is one company or ten, so the key is `add-to-crm-<messageId>-<slug>` in every case.
  - For `companies.length >= 2`: additionally set `batchContext` to `{ "total": <companies.length>, "index": <i> }` so `add-to-crm` can surface "(2 of 4 from David Talpalar digest)" in its Slack alert. `batchContext` is presentation only — **it must never feed the idempotency key.**

  > **Why the key is name-derived (changed 2026-08-04).** The suffix used to be the array index (`-0`, `-1`, …), with the single-company case using a bare `add-to-crm-<messageId>` and no suffix. Both are LLM-output-dependent, so the key space shifted between runs and the queue's dedup silently stopped protecting anything:
  > - Run 1 classifies 1 company → enqueues bare `add-to-crm-<msgId>`, then dies. Run 2 classifies 2 → enqueues `-0` and `-1`. Neither collides with the bare key, so **the first company gets two `add-to-crm` jobs → two Opportunities.**
  > - Run 1 classifies `[Playground, Watchful]` and enqueues `-0`=Playground, then dies. Run 2 orders them `[Watchful, Playground]` → `-0` now dedups against Playground's old key, so **Watchful is silently dropped and never logged**, while `-1`=Playground enqueues a second time.
  >
  > A name-derived slug is stable across re-classification regardless of count or ordering, which is what makes step 3's "a retry of this skill won't double-enqueue the successful ones" actually true — and step 3 explicitly permits partial fan-out with a later retry, so it depends on that being true. **Migration note:** this changes the key shape, so a message enqueued under an old bare/index key before 2026-08-04 and re-routed afterwards could enqueue once more. Changed while the queue was empty (`counts: queued=0`), and `add-to-crm`'s Source-Thread-ID dedup gate now backstops it regardless.

  For referrer cases (forwarded-from-Tom-with-inner-mismatch, forwarded-from-referrer), include `referrerEmail` / `referrerName` at the top level — the webhook passed them as args.

2. Invoke the helper:

```bash
~/.claude/scripts/enqueue-addcrm.sh /tmp/addcrm-args-<messageId>-<slug>.json
```

The helper reads `$CLAUDE_JOB_QUEUE_SECRET` from env (injected by `processor.py:_skill_env()`) and POSTs the envelope to `https://claude-job-queue.tom-182.workers.dev/enqueue`. Idempotency key is computed by the helper as `add-to-crm-<messageId>-<slug(classifierHints.company)>`.

3. Check the result. The helper exits:
   - **0** + response body `{"enqueued": true, ...}` → success, this company's downstream `add-to-crm` job will run on its own tick.
   - **0** + response body `{"enqueued": false, "reason": "dedup"}` → an `add-to-crm` job for this `messageId<suffix>` was already enqueued (e.g. on a previous retry of this skill). That's fine — log `add-to-crm-already-enqueued company=<name> i=<i>` and continue the loop.
   - **Non-zero** (2 or 3) → infrastructure error (missing secret, malformed args, HTTP non-200). **Continue the loop for remaining companies, but exit non-zero from this skill at the end** so the processor moves the job to `failed/` and posts a Slack failure alert. Partial fan-out (some enqueued, some failed) is acceptable — the queue's dedup means a retry of this skill won't double-enqueue the successful ones.

4. After the loop, log a single summary line: `fan-out-complete companies=<N> enqueued=<E> already=<A> failed=<F>`.

`add-to-crm` owns the outcome notification: a NEW deal → a 🆕 TEXT card to Tom (add-to-crm Step 4T; the row is created only on his 👍, then its Step 8 Slack alert fires — Tom 2026-09-24, twin of the Dash lane); duplicate/protected → its Step 8 Slack alert. This skill does NOT post a Slack alert when it enqueues successfully — the alerts come from the follow-on jobs (one per company). For multi-company digests, `add-to-crm` reads `batchContext` and tags its alert with the batch position so Tom can correlate the N Slack messages back to the digest email.

**Never hold a confident deal on your own.** If you think `idd_gate.py`'s Source/Status directive is wrong, still enqueue (add-to-crm Step 4T texts Tom the 🆕 card, where he corrects Source/Status before 👍) and add one Slack note naming the disagreement. A self-invented "held for review" Slack post skips the text opt-in entirely (Inviscid 2026-10-08).

### Step 5: Report to Slack (skip-paths only)

This skill posts a Slack alert ONLY for paths where it does NOT enqueue `add-to-crm` (i.e., the Step 3 gate filtered the message out). Use the `send-alert` skill (read `~/.claude/skills/send-alert/SKILL.md`).

For the gated paths, post:

- `is_deal: false` — suppress silently (exit 0, no Slack post). The webhook's deterministic pre-filter + Haiku gate already filtered most non-deals; the second-pass classifier saying "not a deal" is mundane and noisy if alerted.
- `is_deal: true, confidence: low` — suppress silently (exit 0).
- `is_deal: true, no company extracted` — `🏢 <u>**New Deal Classifier: <subject>**</u> — ⚠ high-confidence positive but couldn't extract company name; manual triage needed. <gmail message URL>`

Successful-enqueue path: no Slack post here. `add-to-crm` owns the outcome notification when it processes the follow-on job: a 🆕 TEXT card for a new deal (Step 4T — created on Tom's 👍), a 🛡️ Slack alert for portfolio-protected outcomes, or — when the match is a **terminal/pass Opp** — the Revive Gate v2 path (`shared-references/revive-gate.md`): add-to-crm enriches the existing row at detection (materials chip with its own Slack alert, dated body update, Stage/Round Details from the deck) and texts Tom a 🔁 revive card for the status flip. There is no ⛔ Slack post for that case, and this skill posts nothing either way. The 📈 live-Opp update (Step 3, `knownLiveOpp`) notifies by TEXT only (`live_update.py`) — no Slack post.

### Step 6: Exit

Exit 0 on any successful path (enqueued, gated/skipped, dedup-rejected at queue layer). Exit non-zero only on infrastructure errors (Gmail fetch failed, `/enqueue` POST returned non-200, missing `CLAUDE_JOB_QUEUE_SECRET`). The processor moves non-zero jobs to `failed/` and posts a Slack failure alert from the queue layer.

## Notes

- **Cold follow-ups never arrive (webhook gate, 2026-09-22):** `deal-scanner.js` drops a sender's repeat messages in a thread Tom never replied in (verdict `cold-followup-skip`). Forwards and threads with an Opp take their existing paths. Carve-outs (a `Pass (Met)` founder always arrives; rebrand alias matching) are defined in `~/.claude/skills/shared-references/revive-gate.md` § "Who reaches this gate".
- **Idempotency (this skill):** the webhook keys the job by `messageId` (`idempotencyKey: 'inbound-deal-detect-' + messageId`), so Gmail Pub/Sub re-deliveries are deduped at the queue layer. The skill itself does not need its own dedup beyond `add-to-crm`'s existing duplicate check.
- **Idempotency (fan-out):** each per-company `add-to-crm` job is keyed `add-to-crm-<messageId>-<slug>` with the name-derived slug defined in Step 4 (never the loop index, never bare). A retry that re-enqueues already-enqueued companies dedups at the queue, and the loop continues for the rest.
- **Founder-sender exclusion is now this skill's job.** The webhook used to skip emails whose sender matched a portfolio founder, but the heuristics (Contact-field substring, People→Founder relation) misfired in both directions — referrers tripped the substring check, and Opps with no Founder relation leaked through. Removed 2026-05-06. The classifier's not-deal rubric ("Founder update on an existing portfolio company") now gates this entirely; rely on it instead of pre-screening on the sender.
- **No People DB row creation** for the founder (per Tom's standing rule) — `add-to-crm` already honors this.
