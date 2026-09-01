---
name: purchase-agent
description: >
  Make a purchase on behalf of Tom or Elsie — household restock, gifts, kids/school
  (class signups, camp registrations, supplies), errands & services (flowers, food,
  dry cleaning), and travel. Trigger on "buy X", "order X", "purchase X", "get X for
  [person]", "book X", "sign Andy up for X", "send flowers to X", or any variant where
  an allowlisted family member wants something bought or booked. HARD GATE: every
  purchase requires an explicit YES from the requester AFTER seeing the exact item,
  price, and merchant — no exceptions, no auto-buy at any price. Works from chat,
  Slack DM, or the SMS line (sms-listener routes here). NOT the agentic-commerce-agent
  skill — that is a market tracker for the AgentBay thesis and never buys anything.
---

# Purchase Agent

Buy things for the family, with money-movement discipline. The requester (Tom or
Elsie) is the decision-maker; this skill is hands, not judgment.

## The four invariants (never bend these)

1. **Quote before charge.** Never let money move until the requester has seen the
   exact item, total price (incl. shipping/tax where shown), and merchant — and
   replied an explicit **YES** (or "yes", "confirm", "buy it", 👍). A vague "sounds
   good" on a vague description is not a quote-confirm. One quote = one purchase;
   changing the item/price re-quotes.
2. **No payment-credential handling.** Never read, type, store, or transcribe card
   numbers/CVVs. Execute only through checkout flows where payment is already saved
   (Tom's Chrome profile autofill, Amazon saved cards, merchant accounts). If a flow
   demands raw card entry, stop and hand the last step to the requester with a link.
3. **Ledger every outcome.** Append every quote, confirm, decline, and completed
   purchase to `~/.claude/skills/purchase-agent/ledger/YYYY-MM.md` (see format
   below). Receipts (order-confirmation pages/emails → PDF) land in the shared
   Drive folder `Kenyon-Seo/Receipts/` (parent id `10BJN5vS5Xld8B3sE29oqrsbchs6K0hVK`;
   create the subfolder on first use).
4. **Requester pays attention to who.** Tom and Elsie are equal requesters. The
   person who asked is the person who confirms — Elsie's request is never held for
   Tom's approval, and vice versa. (If a single purchase exceeds $500, cc the other
   spouse in the confirmation text so nobody is surprised.)

## Flow

### 1. Understand the ask
Item/service, quantity, recipient + deadline (gifts: who is it for, when needed),
delivery address (default: home). Ambiguity that changes what gets bought → one ❓
back to the requester. Ambiguity that doesn't (brand of paper towels) → pick well.

### 2. Research → quote
Find the item. Prefer, in order: (a) merchants where the family already has an
account + saved payment (Amazon first for goods), (b) reputable direct merchants,
(c) anything else only if the requester named it. For gifts/travel, present up to
3 options max, lead with the recommendation. Then send the quote via the channel
the request came from (SMS → short; chat/Slack → can be richer):

Confirmation format — Title Case emoji header, blank line, then the fields (item, total,
ship-to, seller, card, and the **item link** — all required so the requester can audit):
```
🛒 Purchase Confirmation

• Item: [name + size/color]
• Total: $[total]
• Ship to: [home 25 Garden | office]
• Seller: [merchant — flag if 3rd-party]
• Card: [Amex Gold | Brex]   ← name ONLY; NEVER include card digits, not even last 4
• Link: [direct product URL]

Reply YES to place.
```
No bold (iMessage renders Unicode bold weirdly). For gifts/travel with options, list up to
3, lead with the recommendation, each with its link.

**Payment + address toggles (Tom 2026-08-30/31):**
- **Payment:** `personal` = **Amex Gold (…2017)** · `work` = **Brex (…0188)**. On the
  personal Amazon account the Amex Gold is the card ending 2017; do NOT use the Brex
  (…0188, work card) or the auto-selected Chase Sapphire (…2660, burns Chase points) for
  personal — switch to the Amex.
- **Address:** `home` = 25 Garden Pl Apt 2, Brooklyn · `work` = Inverted Capital, 365
  Bridge St Ste 8PRO, Brooklyn. (Amazon's account default is the office — always switch
  to home for personal.)
- **Elsie's requests count as personal** → Amex Gold + 25 Garden (unless she says work).
- **Default is ALWAYS personal + home (Amex Gold → 25 Garden Pl)** — no context
  inference. The work pair (Brex → 365 Bridge) is used ONLY when the requester
  explicitly says so ("on the work card", "ship to the office", "work purchase").
  The quote ALWAYS states the pair being used; changing either re-quotes.
- Amazon's saved address book contains OTHER people's addresses (past gifts: Jon Terbell,
  Mikyung Kim, Steve Seo, a Vermont + Santa Barbara house) — NEVER select an address by
  position or default; always match the exact street string.

### 3. Confirm gate
- Explicit YES from the requester → execute.
- "No"/silence → log declined/expired (quotes expire after 24h — re-quote, prices drift).
- Modification → adjust, re-quote.

### 4. Execute
Through the merchant's normal checkout using saved payment:
- **Browser path (default):** drive Tom's real Chrome (his profile has the logins +
  autofill) via AppleScript/JS injection — see `references/chrome-checkout.md` for
  the per-merchant recipes (Amazon first; add a recipe file per merchant as they
  come up). Screenshot the final order-confirmation screen.
- **No-recipe merchant:** get to the final review page, screenshot it, and if the
  last click can't be made reliably, send the requester the cart/checkout link to
  finish — a handed-off purchase is a success, not a failure.
- **Travel:** always the hand-off pattern for the final booking click unless the
  requester pre-approved the exact itinerary+fare in the quote.

### 5. Close the loop
Reply `✅ Ordered — [item], $[total], arrives [date]` (or `✅ Booked`). Save the
receipt to Drive `Kenyon-Seo/Receipts/`, append the ledger line.

## Ledger format

```
[ISO ts] requester=<Tom|Elsie> item="<desc>" merchant=<name> total=$<amt> status=<quoted|confirmed|declined|expired|ordered|handed_off> ref=<order # or ->
```

## Notes
- SMS entry: sms-listener routes purchase texts here; the quote/confirm loop runs
  over the same thread (warm session remembers the open quote; cold path checks the
  ledger for the most recent `quoted` line from that sender).
- NEVER buy from a quote older than 24h or one the requester didn't see.
- This skill spends real money. When in doubt at ANY step — don't, and ask.
