---
name: outreach-decliner
description: Flip an Opportunity's Status from Qualified/Track/Outreach to Pass (DNM) when Tom's outbound message declines an intro offer or investment opportunity. Uses the shared `classifyOutboundIntent` (Haiku→Sonnet escalation, CacheService-memoized by msg.id so detector reads the same verdict). Mode B (webhook) only — no manual mode, since the signal source is always an outbound email. Runs BEFORE outreach-detector in the handler chain. Does NOT fire post-meeting (Connected/Scheduled/Active use pass-note-sent → Pass (Met) instead).
---

# outreach-decliner

Flip an Opportunity's Status to **Pass (DNM)** when Tom declines a deal before meeting. "DNM" = Did Not Meet. Symmetric to `outreach-detector` but for the decline path; both handlers consume the SAME shared `classifyOutboundIntent` verdict.

## When this fires

Outbound email from Tom that:
- Matches an Opp in Qualified, Track, or Outreach (via the same path cascade as outreach-detector: Contact email, domain stem, Source Thread ID, primary haystack, In-Reply-To parent haystack)
- Returns `verdict: "decline"` with confidence ≥ 0.85 from `classifyOutboundIntent`

Typical scenarios:
- Cold deal inbound at Qualified → Tom replies "Thanks but not a fit for us right now"
- Warm intro offer from a referrer → Tom declines via the Source Thread ID (Path D), Opp at Qualified
- Tom previously initiated outreach (Opp at Outreach), then later declines a follow-up — flips Outreach → Pass (DNM)

## What this does NOT do

- **Does not fire post-meeting.** If the Opp is at Connected/Scheduled/Active, the decline path is Pass (Met) (via `pass-note-sent` after Tom sends a formal pass note), not Pass (DNM). REVIVABLE_FROM = `['Qualified', 'Track', 'Outreach']` only.
- **Does not classify intent from quoted history.** Uses only Tom's own reply body (truncated at `On ... wrote:`). Quoted upstream text would bias the classifier toward the sender's framing.
- **Does not fire on soft deferrals.** "Not right now but keep me posted" → `classifyOutboundIntent` returns `neutral`, no flip. Preserves the Opp at its current state for future action.

## Flow

1. Gate: `SENT && !DRAFT` (handler shared with outreach-detector, intro-outreach, etc.)
2. Match cascade (stops at first hit):
   - **A1** — recipient email = `Opp.Contact`
   - **A2** — recipient email domain stem ⊂ `Opp.Name`
   - **D** — Source Thread ID match on `msg.threadId`
   - **B** — primary haystack (subject + body_head) matches any Opp name
   - **C** — In-Reply-To parent haystack
3. If no match, no-op (log `no-opp-match`).
4. If matched Opp's Status ∉ `{Qualified, Track, Outreach}`, no-op (log `not-flippable`).
5. Call `classifyOutboundIntent(body, msg.id)` (defined in `outbound-intent.js`):
   - Cache check by msg.id → if hit, return memoed verdict (e.g. detector or a prior re-route already classified this msg).
   - Tier 1 Haiku 4.5 (~$0.0003/call) → `{verdict, confidence, reasoning, model}`.
   - If Haiku confidence < 0.85, escalate to Sonnet 4.6 (~$0.001 extra).
   - Cache the final verdict for 6h.
6. If `verdict == "decline"` and `confidence ≥ 0.85`: PATCH `Status = Pass (DNM)` + Slack alert (via `claude` webhook).

Slack alert format (per alert-convention.md — underlined bold colon headline;
transition on the outcome line; no raw confidence score):
```
🚫 <u>**Pass Logged: {Opp name}**</u>
{from status} → Pass (DNM)
{one-sentence reasoning from classifier}
[Open in Notion](…)
```

## Cost profile

- Every SENT matching an Opp in Qualified triggers one Haiku call (~$0.0003).
- ~5-15% of those escalate to Sonnet (~$0.001 extra).
- Rough annual cost at ~10 Opp-related sends/day: **$1-3/year**.

## Related skills

- `outreach-detector` — the symmetric "opt-in" handler (Qualified → Outreach). Shares the match cascade helpers.
- `pass-note-drafter` + `pass-note-sent` — the post-meeting decline path (Outreach/Connected → Pass Note Pending → Pass (Met)).
- `intro-resolution-agent` — handles declines in the intro-target context (moving people between Intros relations), NOT Opp Status flips.
