---
name: outreach-detector
description: Flip an Opportunity's Status from Qualified or Track to Outreach when Tom sends a message that matches the Opp AND the message reads as opt-in intent (not a decline, not neutral logistics). Mode B (webhook) fires automatically via gmail-webhook on every SENT message and uses the shared `classifyOutboundIntent` (Haiku→Sonnet, CacheService-memoized — outreach-decliner runs first in the chain and pays the LLM cost; detector reads the cache). Mode C (manual) lets Tom log an opt-in after the fact and skips the intent gate. Trigger phrases for manual mode: "I opted in to [company]", "log my opt-in to [company]", "flip [company] to Outreach", "move [company] from Qualified to Outreach", "I reached out to [company]", or any variant confirming Tom initiated a connection. Does NOT fire on demotions, refreshes, or Opps past Outreach (Connected/Scheduled/Active/terminal stay put).
---

# outreach-detector

Flip an Opportunity's Status from **Qualified or Track → Outreach** whenever Tom initiates or affirms a connection attempt on the deal. Status `Outreach` is a unified bucket per `reference_outreach_status_semantics.md` — it covers:

- Replying YES to an investor offering a warm intro (Tom opts in).
- Cold-emailing a founder directly.
- Replying to a founder who inbound-reached Tom.
- Any other outbound email where Tom is pushing the deal forward.

The skill only fires when (a) the Opp's current Status is `Qualified` or `Track` AND (b) the outbound classifies as `verdict: "opt-in"` with confidence ≥ 0.85. Declines are handled by `outreach-decliner` (runs first in the handler chain); neutral logistics leave the Opp at its current state.

## Modes

This skill has **Mode B** (webhook) and **Mode C** (manual). No Mode A (sweep) — the webhook twin is event-driven and idempotent enough that a reconciler hasn't been needed. Revisit if drift emerges.

### Mode B — Webhook (default, auto)

Lives inline in `/Users/tomseo/code/gmail-webhook/outreach-detector.js`. Invoked by `gmail-webhook/Code.js` handlers array on every `SENT && !DRAFT` message.

Flow (match paths tried in order — first hit wins):

1. **Path A1 — Contact email**: query Opps where `Contact contains recipient email`. Most deterministic.
2. **Path A2 — domain stem → Name**: domain from each recipient (skip personal domains), TLD-stripped (bloom.site → `bloom`), query Opps where `Name contains stem`. Catches work-email founders when Contact isn't pre-populated.
3. **Path D — Source Thread ID**: if `msg.threadId` matches an Opp's `Source Thread ID`, that's the deterministic referral-thread match. Handles "Tom replies to a Fwd: thread that add-to-crm tagged at create time."
4. **Path B — primary haystack**: subject + first 500 chars of body via `findOppByHaystackMatch`. Cold outreach where Tom writes the company name.
5. **Path C — In-Reply-To parent haystack**: parent message's subject + first 2000 chars of body. Warm-intro opt-in replies where only the upstream message mentions the company.

Once an Opp is matched:

6. Fetch the Opp page.
7. **Status gate.** REVIVABLE_FROM = `['Qualified', 'Track']`. If Status ∉ that set, log `not-flippable` and stop. Outreach/Connected/Scheduled/Active/terminal all bail here — a flip back is a regression.
8. **Intent gate.** Call `classifyOutboundIntent(body, msg.id)` from `outbound-intent.js`. CacheService memo means outreach-decliner already paid the LLM cost on this msg.id earlier in the handler chain. If `verdict != "opt-in"` OR `confidence < 0.85`, log `not-opt-in` and stop.
9. PATCH `Status = Outreach` + post a Slack alert.

Idempotency: if Tom sends a second message in the same thread after the flip, Status is already Outreach — the handler sees `not-flippable` and no-ops. No message-ID dedup needed.

Slack alert format:
```
📞 **{Opp name}** — opted in, Qualified → Outreach
[Open in Notion]({opp.url})
```

### Mode C — Manual

Tom says something like:
- "I opted in to Bloom"
- "Log my opt-in to Acme"
- "Flip Bloom to Outreach"
- "I reached out to Acme cold"

Execute:

1. **Find the Opp.** Search Notion Opportunities by exact title match on the company name Tom specified. If multiple matches (Fund I vs Fund II variants), pick the one whose Status is currently `Qualified`. If still ambiguous, ask Tom which one.
2. **Verify Status.** Fetch the page and read `Status.status.name`.
   - If `Qualified` → proceed to flip.
   - If already `Outreach` → tell Tom "already Outreach, no-op" and stop.
   - If any other status → tell Tom the current status and ask whether to force-flip. Do NOT flip without explicit confirmation.
3. **Flip Status.** PATCH `{ Status: { status: { name: "Outreach" } } }` via Notion MCP.
4. **Alert.** Send a Slack note to Tom's DM-to-self via the `claude` webhook (see `send-alert` skill). Same format as Mode B but flag it as manual:
   ```
   📞 **{Opp name}** — opted in, Qualified → Outreach (manual)
   [Open in Notion]({opp.url})
   ```
5. **Confirm.** One-line reply to Tom: `Flipped {Opp name} · Qualified → Outreach.`

## What this skill does NOT do

- **Does NOT write a Note to the Opp page body.** Per `feedback_opportunity_page_body_note_only.md`, the body is Note-only and owned by whatever created the Opp. The outreach signal is a state change, not a narrative addition. Tom's actual opt-in reply is in Gmail — that's the authoritative record.
- **Does NOT touch `Latest Outreach`.** That field is being deprecated — don't write to it. If you see it on the page, leave it as-is; Tom will delete it in Notion when ready.
- **Does NOT move people between Intro relation fields** (👓 Qualified → ☎️ Outreach). That's `intro-outreach-agent`'s job; it runs as a sibling handler in the gmail-webhook and operates independently.
- **Does NOT fire on any status except Qualified.** Outreach→Connected is `intro-connected-detect`'s job. Other transitions are silent here.
- **Does NOT require LLM classification in the webhook path.** Pure deterministic code. The job queue is not invoked.

## Testing

Mode B smoke test from Apps Script editor:
```js
_smokeTestOutreachDetectorOnMessage('MESSAGE_ID_HERE');
_smokeTestOutreachDetectorFromSearch('in:sent newer_than:2d bloom');
```

Mode C smoke test: `"I opted in to Bloom"` → skill finds the Bloom Opp, verifies Qualified, flips, alerts.

## Related skills

- `intro-outreach-agent` — moves PEOPLE between Intro relation fields on the Opp. Runs in parallel to this skill on every SENT message. Independent concerns.
- `intro-connected-detect` — the NEXT transition (Outreach → Connected) when a three-way intro lands in Tom's inbox.
- `pipeline-agent` — scheduled sweep of the whole pipeline. Should not duplicate this skill's work; if it sees an Opp in Qualified with recent outbound mail, it can flag but should defer the actual flip to this skill's webhook.
