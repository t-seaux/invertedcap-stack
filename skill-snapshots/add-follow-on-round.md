---
name: add-follow-on-round
description: Create a follow-on round Opportunity card ("FO card") in the Notion CRM for a company already in the portfolio that is raising a new priced round. Manual-only. Mirrors the company's existing FO cards — inherits Fund + identity fields from the existing Opp cluster, links the full 🕰️ Funding History, sets Status=Active (Tom flips to Committed himself on commit), Source=Direct, and leaves Inv @ Round blank until Tom gives a number. Trigger on "create a [Series X] FO card for [company]", "add the [company] [Series X] follow-on", "log [company]'s new round", "follow-on card for [company]", "[company] is raising a [Series X] — add the card", or when Tom hands over a founder's round email/screenshot for a company already in the portfolio and wants the card. Does NOT model returns — if Tom asks to "calc pro forma", hand off to the pro-forma-round skill. Distinct from add-to-crm (new companies / first opportunity) and pro-forma-round (returns modeling, writes nothing).
---

# Add Follow-On Round

Create a new **follow-on round card** in the Notion Opportunities DB for a company Tom already holds, when it raises a new priced round. The card mirrors the company's existing round cards (e.g. `Outmarket (Seed FO)`, `Outmarket (Series A FO)`) so the funding history stays a clean, linked thread.

This skill is **card-only** — it does not model returns. It also **inherits** identity fields from the existing Opp cluster rather than re-enriching, because the company is already in the portfolio.

Manual-only (Mode C). Tom invokes it in conversation; the round terms come from whatever he hands over (a founder's email, a screenshot, or stated terms).

## Inputs

- **Company** — named by Tom, or inferred from the source material's sender/subject.
- **Round terms** — stage (Series B, etc.), raise amount, post-money (or cap). From the founder's email / screenshot / Tom's words.
- **Lead investor** — if stated (e.g. "SignalFire leading").

If the source is a founder email, capture its full text verbatim for the page body.

## Step 1: Resolve the existing Opp cluster

The company must already exist in the Opportunities DB — this skill adds a round to an existing position, it does not create first opportunities.

1. Search the Opportunities data source `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6` for the company name via `notion-search` (`content_search_mode: "workspace_search"`). If empty, back it with a SQL `Name LIKE '%<name>%'` query via `notion-query-data-sources` before concluding it's absent (semantic search misses exact titles).
2. Fetch the matched card and read its `🕰️ Funding History` relation — that surfaces the **full cluster** of round cards (base + every prior FO). Collect all of them.
3. Pick the **base card** (earliest / the one titled just `<Company>`) as the source of inherited fields.

**If no existing card is found → this is not a follow-on.** Tell Tom and offer to route to `add-to-crm` instead. Do not create a bare FO card for a company with no history to link.

## Step 2: Assemble the new card's fields

**Inherited from the base card (do NOT re-enrich):**
- `Fund` — read off the existing cards and match it (e.g. `Dash 2️⃣`). Never fall back to add-to-crm's `Inverted 1️⃣` default.
- `Description`, `HQ`, `Website`, `Contact` — copy from the base card.
- `🏁 Founder(s)` — copy the base card's founder relation URLs. Never auto-create People rows.
- `icon` — reuse the base card's icon (the company logo) for visual consistency across the cluster.

**From the round terms:**
- `Name` — `<Company> (Series X FO)`. Capitalize the stage fully: `(Seed FO)`, `(Series A FO)`, `(Series B FO)`. This is the follow-on suffix the dedup logic elsewhere expects.
- `Stage` — the round's stage, exact emoji variant from the schema: `Seed 🌾`, `Seed+ 🛣️`, `Series A 🏎️`, `Series B 📈`, `Growth 🚀`. (Series B = `Series B 📈`.)
- `Round Details` — strict format: `$Xm on $Ym post` (or `$Xm on $Ym cap` for a SAFE). Lowercase `m`/`k`. Trust the terms Tom hands over; **if he attaches a term sheet or deck, cross-check the post-money against it.** Leave blank if no $ figure is disclosed.

**Fixed defaults:**
- `Status` — **`Active`**. This is the resting state; Tom moves it to `Committed` himself when he actually commits, and to `Portfolio: Follow-On` when it closes. Never set Committed/Follow-On automatically.
- `Inv @ Round` — **leave blank.** Tom supplies his check size separately; only set it when he gives a number in this or a later turn.
- `Source(s)` — always **Direct**: `https://www.notion.so/0fb9a64034fd46f9934768d590e69dc9`. Follow-ons come straight from the founder; there is no referrer.
- `Support` — the "N/A" entry: `https://www.notion.so/18200beff4aa80bc8344fc48c7b0fdb1`.
- `Followed Up` — `__NO__`.
- `Close Date` — blank.

**Lead / coinvestor (if stated):**
- Search the Coinvestors DB `collection://7d50b286-c431-49f5-b96a-6a6390691309` for the lead (e.g. SignalFire). If found, link it in `Coinvestors`. If not found, **leave blank and surface the gap** — never auto-create a Coinvestor row. Confirm the link back to Tom.

## Step 3: Create the card

The Opportunities DB has a PreToolUse gate (`~/.claude/hooks/gate-opps-creation.sh`) that blocks direct writes. **Before the `notion-create-pages` call, run `touch /tmp/.addcrm-bypass`** (the marker auto-expires in 5 min; no cleanup needed). If the hook denies with "Direct creation of Notion Opportunities-DB rows is gated", you forgot this step.

Create with parent `{"data_source_id": "fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"}` and the fields from Step 2.

**Page body — one section only, source material verbatim:**
```
**Original Email**

[founder's email text verbatim — hyperlinks and paragraph breaks preserved, no `---`/`***` dividers]
```
Swap the header noun by source type: `**Original Text**` (screenshot/SMS), `**Original DM**` (LinkedIn). If Tom only stated terms verbally (no artifact), omit the body.

**URL fidelity:** any URL written into the body must be a literal substring of the source — never reconstruct or guess one.

## Step 4: Confirm

Reply with a concise summary: the new card link, inherited Fund, Stage, Status (Active), Round Details, linked Funding-History count, and the coinvestor link (or the gap). Flag the two things Tom drives manually: `Inv @ Round` is blank until he gives a number, and Status stays Active until he commits.

The `🕰️ Funding History` relation is **dual** — setting it on the new card auto-back-links the new card onto all the prior cards. No need to edit the siblings.

## Pro-forma hook (on request only)

This skill writes the card; it does **not** model returns. If Tom asks to "calc pro forma", "model this out", "what's my MOIC", or similar — before or after the card exists — hand off to the **`pro-forma-round`** skill with the company + round terms. Do not inline the modeling here.

If Tom wants the pro forma **recorded on this card** ("run the PF in the body", "log the PF on the card"), have `pro-forma-round` write it into this Opp's page body via its *"Recording the PF to the Opp card body"* output mode — a dated `## <Company> <Round> PF Draft — <date>` header, a confirmed/NOT-confirmed assumptions block (round size, valuation, option-pool refresh, Tom's check, SOI anchor), the company + fund tables, and the read. The `Outmarket (Series B FO)` card is the canonical example.

## Worked example (Outmarket Series B, 2026-08-15)

Vishal (founder) emailed Tom: Series B, $33.5M at $335M post, SignalFire leading, ~$500K for Tom. Outmarket is Dash 2️⃣ with three prior cards (Pre-Seed SAFE, Seed FO, Series A FO). Result: new card `Outmarket (Series B FO)` — Fund `Dash 2️⃣` (inherited), Stage `Series B 📈`, Status `Active`, Round Details `$33.5m on $335m post`, Source Direct, founders + all three prior cards linked, SignalFire linked as coinvestor. `Inv @ Round` was left blank by the skill; Tom then chose to set it to `$500K` on this card himself, and drives the Active→Committed flip manually.
