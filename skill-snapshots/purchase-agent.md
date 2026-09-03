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

0. **An explicit YES text FIRES the order.** (Tom 2026-09-02, after placing a real order
   end-to-end: *"for any online ordering flow I want you to honor the fact that an
   explicit yes text from me fires off the order."*) A YES is the trigger, not an
   acknowledgement to relay back. Staging a cart and then telling him to click it himself
   is a FAILURE of the flow — the entire point is that he can be away from the desk.
   Execute via the guarded placer, which verifies consent against the message record.
   Merchant-agnostic learnings: `references/online-ordering-playbook.md`.

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
**PRIORITIZE DIRECT MERCHANTS** (Tom 2026-09-02: *"you should already know this — we
prioritize direct merchants"*). Within any marketplace, prefer the first-party seller over
a third-party one: **Ships from / Sold by Amazon.com** over a marketplace seller,
Target-owned over Target+. This is a SELECTION rule, not just a disclosure — pick the
direct-seller listing even at a slightly higher price, and only surface a third-party
option when there's no first-party alternative (say so explicitly in the quote).

Why it's a hard rule, not taste: third-party listings silently break the flow. A
`Hanes Tagless BEEFY-T` sold by **AmeriStyle** sat wedged in Tom's Amazon cart from
Aug 31 to Sep 2 because that seller wouldn't ship to his address — and it **blocked the
entire checkout pipeline**, with Amazon refusing to render a Continue control until it
was removed. The same shirt was available from Amazon.com directly, in the same cart.
Returns, delivery estimates, and support all degrade the same way.

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
🛒 Purchase Confirmation: [Merchant]

Transaction Summary
• Address: [Office (365 Bridge) | Home (25 Garden Pl)]
• Card: [Brex (Mastercard *0188) | Amex Gold]
• Arrives: [Fri, Sep 4] @ [9-10a]
• Total: $[grand total] ($[items] − $[promo] + $[shipping/delivery] + $[other fees] + $[tax])
• Link: [cart URL, or the product URL for a single item]

Items ([n]) – $[subtotal]
• [qty]x [item name (variant)] – $[unit]
• [qty]x [item name (variant)] – $[unit]

Reply YES to place as-is ($[total]) or respond to edit cart / delivery time.
```

Rules for that block (Tom 2026-09-02, this exact shape):
- **The Total parenthetical must reconcile to the total.** Every component that moves the
  number gets a term — promos, delivery fee, regional/bag/deposit fees, tax. A breakdown
  whose parts don't sum to the stated total is a bug (a Target quote once showed lines
  summing to $47.19 on a $51.47 charge because regional fees were invisible to the cart API).
- **Source the numbers from the CHECKOUT page, not the cart API** — see
  `target-checkout.md`. Run `review` before sending.
- **NEVER re-derive or re-group the breakdown.** Emit `review`'s line items verbatim:
  items, promo, delivery, regional fees, tax. Do not fold the promo into the item price,
  do not merge delivery + regional into one "fee", do not recompute anything. On
  2026-09-02 a stale daemon session invented `$64.37 item price + $4.28 tax + $17.48 fee`
  for the same order whose real breakdown was `$70.88 − $6.51 + $9.99 + $7.49 + $4.28`.
  The total happened to match, which is exactly what makes it dangerous — a plausible
  wrong breakdown reads as authoritative. If a `review` field is missing, say so; never
  reconstruct it from arithmetic.
- **After editing this format, KICKSTART THE DAEMON** or texts keep going out in the old
  shape: `launchctl kickstart -k gui/$(id -u)/com.invertedcap.sms-agent-daemon`. The warm
  session reads the skill files ONCE at session start and caches them, so edits are
  silently ignored by a running session until it resets (auto after 30 min idle).
- **Don't include the auth-hold line** (Tom 2026-09-02 — cut it as noise). Same-day
  delivery authorizes total + 25% for substitutions, but he's only charged for what
  arrives, so the hold isn't decision-relevant. Mention it only if he asks, or if a
  hold is unusually large / would plausibly bounce.
- **Card digits:** the last-4 comes from Tom's OWN configured card list (below), never
  from scraping the merchant's stored-payment data. Render brand + the configured last-4.
- **One fixed call to action**, verbatim: `Reply YES to place as-is ($[total]) or respond
  to edit cart / delivery time.` Don't enumerate bespoke options (TRIM/swap/etc.) — the
  open "respond to edit" covers them, and a free-text reply is parsed as an edit.
- **The Link bullet goes DIRECTLY under Total — no blank line between them**
  (Tom 2026-09-02: "there shouldn't be a line break between total and link"). Cart URL for
  a multi-item order, product URL for a single item — it's the audit trail: he can open it
  and see exactly what he's approving.
- **NO explanatory prose in the block.** Tom cut a trailing "Ships from and sold by
  Amazon.com. Tax $0 — NY exempts clothing under $110." line as unnecessary. The
  confirmation is a decision surface, not a briefing: if a number needs a caveat it
  belongs in the Total parenthetical, not a sentence. Direct-seller sourcing is the
  DEFAULT (see the direct-merchant rule), so it needs no announcement — only its
  violation does. Flag a marketplace seller inline on the item line
  (`1x Widget (Blue, XL) — $15.00 ⚠️ 3rd-party seller`), never as a paragraph.
- **Address renders short**: `Office (365 Bridge)`, not the full street address.
- **Amazon: state Prime eligibility on the Arrives line** (Tom 2026-09-02) —
  `• Arrives: Thu, Sep 3 (Prime)`. If the item is NOT Prime, say so loudly
  (`(NOT Prime)`), because that's the case that changes the decision: slower delivery and
  possibly a shipping charge. `amazon_api.mjs review` returns a `prime` boolean, read from
  the Prime BADGE ELEMENT — never from the word "prime" in body text, which appears all
  over Amazon's Prime Visa / Prime Business Card promos and would always false-positive.
- **A delivery/pickup window is Tom's call — always show it, never bury it** (Tom
  2026-09-02: "should I be in a position to confirm delivery time"). Don't present a
  scheduled window as settled.
  - **Office snack orders → default to the EARLIEST available slot** and state it plainly
    in the confirmation; Tom edits by replying if he wants a different one. Don't ask
    first — earliest is the right default for the office.
  - **Everything else** → state the selected window and the realistic alternatives.
  - Filter alternatives by destination: an **office** delivery only works in business
    hours, so never offer 8–11pm or weekend slots for 365 Bridge St just because the
    merchant lists them. Same for home — don't schedule perishables into a window when
    nobody's there.
  - Changing the window doesn't change the price → it's an edit, not a re-quote.
  - **RENDER time ranges as `9-10a` / `3-4p`** (Tom 2026-09-02) — a plain HYPHEN, not an
    en dash, and a single `a`/`p`, not `am`/`pm`. This overrides the usual en-dash house
    style for time ranges only; the rest of the block (e.g. `– $8.78`) keeps en dashes.
  - **Map a stated time to the window that CONTAINS it** (Tom 2026-09-02: "if I say I
    want it delivered at 4:30 then you know to pick the 4-5 option"). Merchants sell
    windows, not instants — resolve his natural phrasing to a real slot and echo the
    resolved window back so he can see the mapping:
    - `4:30` / `4:30pm` / `around 4:30` → the slot containing it → **4-5p**
    - `4pm` / `at 4` → the slot STARTING at 4 → **4-5p** (not 3-4p)
    - `9a` / `first thing` → the slot starting at 9 → **9-10a**
    - `after 5` → earliest slot starting ≥5 → **5-6p**
    - `before 5` / `by 5` → latest slot ENDING ≤5 → **4-5p**
    - `asap` / `earliest` → first available slot
    - No slot contains the stated time (e.g. "9am" when slots start at 3pm) → do NOT
      silently pick the nearest. Say what's actually available and let him choose.
    - An explicit request overrides the office-hours filter — if he asks for 8pm at the
      office, book it but flag that nobody may be there to receive it.

**Every one of those lines is required (Tom 2026-09-02).** He greenlights from the text
alone, so the text must carry all key info — amount, items, shipping, arrival ETA. If a
value is genuinely unavailable, say so explicitly (`Shipping: unknown until checkout`)
rather than omitting the line. Never show a bare total with no shipping/tax breakout.

**Pull these from the CART/checkout, not from a product page.** Search and PDP prices can
disagree with what's actually charged (Target: same item $4.29 on search vs $5.29 in the
cart, because the store id differs). The cart is the only authoritative price.
No bold (iMessage renders Unicode bold weirdly). For gifts/travel with options, list up to
3, lead with the recommendation, each with its link.

**Payment + address toggles (Tom 2026-08-30/31):**
- **Payment — THREE cards, Brex is the DEFAULT** (Tom 2026-09-02, replacing the old
  "personal = Amex Gold" rule, which was wrong):

  | Card | Role | When |
  |---|---|---|
  | **Brex \*0188** | **DEFAULT** | everything, unless he specifies otherwise |
  | **Chase Sapphire \*2660** | Tom's personal card | he says personal / his own |
  | **Amex Gold \*2017** | family card | family/household spend |

  His words: *"Brex unless I specify otherwise. Chase is my personal card, Gold is family
  card."* Do NOT infer the card from what's being bought or where it ships — a personal
  T-shirt shipping to the office still goes on **Brex** absent instruction. Ask or default;
  never guess from context.
- **Never select a card by position, and read the NAME ON CARD.** Amazon's payment list
  includes **Prime Visa \*9578 in "Mi Kyung Kim"'s name** — someone else's card sitting
  among Tom's. Match the exact card label every time.
- **Address:** `office` = Inverted Capital, 365 BRIDGE ST STE 8PRO, Brooklyn ·
  `home` = 25 GARDEN PL APT 2, Brooklyn.
- **⚠️ ADDRESS AND CARD ARE INDEPENDENT — do not infer one from the other**
  (Tom 2026-09-02, correcting the old paired default):
  - **Address default = 365 Bridge (office), ALWAYS**, unless he says home/25 Garden.
    His words: *"I have a preference for shipping to 365 Bridge unless I specify home."*
    Packages are received reliably at the office; home delivery is the exception.
  - **Card default = Amex Gold (personal)**. Brex is used ONLY for genuine work
    purchases ("on the work card", "work purchase") — NOT merely because something ships
    to the office. A personal T-shirt delivered to 365 Bridge is still Amex Gold.
- **Elsie's requests count as personal** → Amex Gold (unless she says work).
- The quote ALWAYS states both the address and the card; changing either re-quotes.
- **Never select an address by position.** Amazon's book holds SIX, including other
  people's — Mikyung Kim (Northvale NJ), Steve Seo (Fort Lee NJ), a Santa Barbara and a
  Bridgewater VT address. Match the exact street string ("365 BRIDGE ST" / "25 GARDEN PL").
- Amazon's saved address book contains OTHER people's addresses (past gifts: Jon Terbell,
  Mikyung Kim, Steve Seo, a Vermont + Santa Barbara house) — NEVER select an address by
  position or default; always match the exact street string.

### 3. Confirm gate
- Explicit YES from the requester → execute.
- "No"/silence → log declined/expired (quotes expire after 24h — re-quote, prices drift).
- Modification → adjust, re-quote.

### 4. Execute
Through the merchant's normal checkout using saved payment:
**API-first (standing rule, Tom 2026-09-02).** Prefer a non-browser HTTP/JSON path for
every merchant. Fall back to Chrome only when the merchant makes pure HTTP impossible
(bot walls), and even then use the browser as *transport* — `fetch()` from inside the
page context — not as a UI to click through. Order of preference:

1. **Pure HTTP** (curl/node from a script) — no browser at all. Best case.
2. **Browser-as-transport** — an isolated logged-in CDP profile used only as an
   authenticated HTTP client (`target_api.mjs`, `meevo_api.mjs` pattern). Use when the
   merchant runs PerimeterX/Akamai-class bot detection.
3. **Browser-as-UI** (`click-text`/`click-sel` through checkout) — last resort, for
   merchants with no usable API surface. Brittle; breaks on every redesign.

Per-merchant recipes: Amazon → `references/chrome-checkout.md` (tier 3 today);
Target → `references/target-checkout.md` (tier 2, PerimeterX-walled — verified).
Adding a merchant = probe for an API first, then a profile+port logged in once.
Screenshot or capture the JSON of the final order confirmation as the audit artifact.
- **No-recipe merchant:** get to the final review page, screenshot it, and if the
  last click can't be made reliably, send the requester the cart/checkout link to
  finish — a handed-off purchase is a success, not a failure.
- **Travel:** always the hand-off pattern for the final booking click unless the
  requester pre-approved the exact itinerary+fare in the quote.

### 4b. Block the delivery window on the calendar (ORDERS ONLY)

**Only after an order is actually placed** — a real order number in hand, ledger
`status=ordered`. Never on a quote, never on a YES that hasn't completed. A calendar hold
for an order that didn't happen is worse than no hold.

Create the event on the **Elsie-Tom** shared calendar
(`cd6mc2c68fcpmfif61rhhl51hs@group.calendar.google.com`), spanning the delivery window:

- **Transparency: BUSY** (`transparency: "opaque"` — the Calendar API default). Tom
  settled on this 2026-09-02 after first asking for free: *"the cleanest thing to do would
  be to mark me as busy so I can't be booked then."* Free was self-defeating — a free
  event is still bookable, so "keep me at the office" would have been a suggestion that
  every scheduler ignores. Busy enforces it with no scheduling-side wiring at all.
- **Title:** `📦 Target delivery @ office — be at 365 Bridge`
- **Description:** the order number, item count, total, and the reason — *"Holding this
  window to receive the delivery at 365 Bridge. Remote/phone/video is fine; the block
  exists to prevent in-person meetings elsewhere."*

The cost is real and intended: the window is genuinely unbookable, so keep it TIGHT —
exactly the delivery window, no padding. A 1-hour block is cheap; a half-day is not.

Keep it in sync: if the window is rescheduled, MOVE the event; if the order is cancelled,
DELETE it (per [[feedback_always_confirm_before_delete]], a stale hold is worth removing —
but say so). Deliveries slip, so a hold that silently drifts out of date is a liability.

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
