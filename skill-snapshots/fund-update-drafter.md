---
name: fund-update-drafter
description: Draft an outbound fund / portfolio update — the email Tom sends an LP, prospective LP, or peer summarizing a fund's portfolio companies and noteworthy developments (per-company blocks + fund-level stats, optionally covering predecessor Dash funds). Trigger on "draft portfolio update", "draft fund update", "draft a portfolio / fund update", "fund update for [person]", "portfolio update for [person]", "give [X] an update on fund 1 / the portfolio", a bare "fund update" or "portfolio update" where Tom is asking to GENERATE one, or a forwarded email where someone asks Tom for an update on his fund(s). Disambiguation: Tom forwarding/pasting a founder's update INTO the system = investor-update; Tom asking to produce an update going OUT = this skill. NOT the investor-update skill (which processes INBOUND updates FROM portfolio companies) and NOT LP letters (long-form quarterly essays — that's writing-style/letters-and-memos via memo/letter flows). Manual-only. Output is a Gmail draft — an HTML-formatted reply on the requester's thread — never a send.
---

# Fund / Portfolio Update Drafter

Draft the outbound email Tom sends when someone asks for an update on his fund(s). Born from the Jim Lim reply (Sep 2026); refine here as the template evolves.

## Step 0 — Scope

- Default fund: **Inverted 1**. Include the **Dash predecessor section** only when the recipient asked about Dash too (as Jim did) or Tom says so.
- Read the thread being replied to: note anything the recipient said about **citing/forwarding** (drives the confidentiality line, Step 4).

## Step 1 — Pull portfolio data (Notion)

**Opportunities DB** — data source `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6` (schema: `add-to-crm/references/schema.md`):

```sql
SELECT "id", "Name", "Status", "Description", "Website", "Current OS%",
       "date:Close Date:start" AS close_date
FROM "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"
WHERE "Fund" LIKE 'Inverted 1%'
  AND "Status" IN ('Committed', 'Active Portfolio', 'Portfolio: Follow-On', 'Exited')
ORDER BY date(close_date)
```

(Fund select values carry emoji — `Inverted 1️⃣`, `Dash 2️⃣` — so match with `LIKE '<name>%'`.)

- **Fold follow-on rows** (`(… FO)` suffix / `Portfolio: Follow-On`) into the parent company's block — one block per company, keyed on the initial-investment row. Follow-ons surface inside the block (Next Round or Highlights: "insiders pre-empted the Seed", "Inverted followed on twice").
- `Invested` month = initial row's Close Date. `Current Ownership` = `Current OS%` (a fraction: 0.08 → 8%); the initial row already reflects post-follow-on current ownership.
- **Description is verbatim** from the Description field — never rewritten. If it's stale vs. a pivot (the Quiet AI → Quiet Software case), flag to Tom instead of silently editing either the field or the email.

**Company Updates DB** — data source `collection://bf491fb9-214f-456e-921b-5194b8187f2a` (conventions: `shared-references/company-updates-db.md`). Pull recent rows (last ~3–4 months per company) for Summary / Traction / Update Type.

- ⚠️ **Resolve by Company relation, not name matching** — update rows survive renames under old names (`Quiet AI` rows vs the `Quiet Software` Opp). A name-only sweep silently drops a company's history.
- Formal (founder-written) content anchors the highlights; calls add color — same precedence as the Company Updates rolling rules.

## Step 2 — Company blocks

One block per company, close-date order. Exact shape (labels bold through the colon; company name underlined with the domain as a direct-href link; header line italic):

```
Company Name (domain.com)
Invested: Mon YYYY | Current Ownership: X%
• Description: <verbatim Notion Description field>
• Highlights: <1–3 sentences>
• Next Round: <one line>
```

**Highlights** — editorialized momentum framing, not a metrics dump. Open with Tom's read ("Seeing early signs of commercial momentum:", "Stellar early momentum:") then the 1–2 proof points with numbers from the formal updates. Very-early companies get one honest line ("Very early in distribution", "Closed last week (Aug 24) – the Fund's newest investment"). Reference a fund-deck page when one exists ("see p.4 in attached").

**Next Round** — status + timing: "Planning for Seed in early 2027", "N/A – insiders pre-empted the Seed round based on early momentum", "N/A – early". Tom's forward-looking color is welcome here ("my prediction is this one could catch heat given founder profile in 2027").

## Step 3 — Email scaffold

```
<Name> – of course! See below and apologies in advance for the email length.
<Headline framing: the single most exciting update, named up top with why.>

Best,
Tom

Inverted Capital I                                    ← underlined header
Fund Size: $Xm (of which ~$Ym closed) | Vintage: YYYY | Called: X% | Returns: X.Xx MOIC, X.Xx DPI

• <active count>; portfolio-construction target (~18 over 3-year deployment)
• Graduation proof point + pro forma image (place the image AFTER the full
  bullet sentence — never mid-sentence)
• Expected graduations with dates
• "Company overviews and updates below"

<Company blocks, Step 2>

–
Predecessor funds: Dash Fund I, Dash Fund II        ← only when in scope

Dash Fund I
Fund Size / Vintage / Called / Returns line
• Fund-returner narrative bullets (EvenUp)

Dash Fund II
Fund Size / Vintage / Called / Returns line
• Bench of potential returners → the emerging returner narrative (Outmarket)
• Position-sizing tables: Company Level / Fund Level / Terminal Value
```

The Dash tables come from the **pro-forma body on the relevant FO Opp page** (e.g. Outmarket (Series B FO)) — reuse them, don't rebuild. Any new priced-round math requires the pro-forma cap table (hard rule: `feedback_priced_round_math_requires_cap_table.md`); SAFEs are never marked up.

**Voice:** `writing-style/letters-and-memos/STYLE.md` register — sober even when results are good, top-down assertion, en dashes, no em dashes, direct-href links (per `feedback_writing_mechanics.md`).

## Step 4 — Verification gates (each caught a real error in the first run)

1. **Table internal consistency** — per row, MOIC × cost = FMV at both ends of every arrow; per-tranche rows sum to the Blended row. (Caught the Outmarket Series A FO row showing the wrong tier's FMVs.) If the source table in Notion is wrong, fix it there first so the artifact and email agree.
2. **Header-vs-table cross-check** — every stat in a Fund Size / Returns header line must match its table (caught DPI 0.1x vs 0.05x).
3. **Dates sanity** — forward-looking claims ("graduate in late 2026 / early 2027") checked against today's date.
4. **Ownership** — only from `Current OS%`; never derived arithmetic.
5. **Claim freshness** — anything in Highlights not traceable to a logged update (e.g. "closing first customer at $50–75k") gets flagged to Tom as unverified before send.
6. **Typo sweep** — read the full draft; report each fix with its exact location (section + bullet) so Tom can apply it in his mail client.
7. **Confidentiality + completeness** — if the recipient may cite/forward, recommend one line drawing the citable/private boundary (especially around unclosed rounds with named leads); confirm every substantive question in their email is answered.

## Output — Gmail draft

Create a **Gmail draft as a reply on the requester's thread** (Gmail MCP `create_draft` with the thread id; NEVER `update_draft` — it sanitizes signature HTML, per `feedback_gmail_connector_create_only.md`). Do not send.

HTML formatting rules for the body:

- Company header: underlined company name with the domain as a direct-href `<a>` (never a bare URL); `Invested: Mon YYYY | Current Ownership: X%` line italic.
- Labels (`Description:` / `Highlights:` / `Next Round:` and fund-header stat names) bold through the colon; blocks as `<ul>` bullets.
- Fund names underlined + bold; the fund stat line italic.
- Position tables as real HTML `<table>`s matching the Company Level / Fund Level / Terminal Value shapes.
- En dashes throughout; `–` divider line before the predecessor-funds section.
- Append Tom's canonical signature per `shared-references/gmail-signature.md` (API drafts land signature-less otherwise).

Any image (e.g. the seed pro-forma graphic) can't be attached via the API path — leave a clearly-marked `[INSERT: <image name>]` placeholder on its own line and tell Tom where to drop it.

After creating the draft, reply in chat with the draft link plus anything Tom still owes the draft (image placeholders, unverified-claim flags from gate 5). When Tom shares a revised/composed version for review (screenshots), run the Step 4 gates and report findings ranked: real numbers errors first, judgment flags second, typos last with exact locations.
