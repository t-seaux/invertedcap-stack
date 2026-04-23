---
name: add-to-companies
description: >-
  Add a company to Tom's Notion Companies database and run the full enrichment
  pipeline. Dedups by domain/name, creates the row if new, then populates every
  field per the canonical spec at `shared-references/companies-enrichment-spec.md` —
  identity (Name, Category, URL, LinkedIn URL, Overview, HQ, Founded Year,
  Company Type), Revenue, Total Funding, Funding Status, Momentum (rounds +
  Deal Digest mentions), Employee Count, Headcount, HC Commentary — using
  ContactOut + Sales Nav headcount scrape + Deal Digest cache as sources.
  Also sets the page icon from ContactOut's logo. Part of Research Management —
  this is how the company knowledge base gets built up. Trigger phrases: "add
  [company] to companies", "add this company", "log this company", "enrich
  [company]", "create a Companies entry for [X]", "add [domain] to companies",
  "add [LinkedIn company URL]", or any variant indicating Tom wants a company
  logged and enriched in the Companies DB. Also trigger on a bare
  `linkedin.com/company/...` URL or a bare company domain with add/log intent.
  Distinct from `neg1-enricher` (person-level; invokes THIS skill as a
  subroutine per employer/school) and `add-to-crm` (Opportunities pipeline).
  Skill does NOT create People entries, Opportunities entries, or any
  non-Companies artifact.
---

# Add to Companies

Canonical entry point for the Companies DB. Adds a company and runs the full
enrichment pipeline, producing a single cleanly-formatted row that any
downstream skill (pipeline-agent, founder-outreach, first-pass-diligence) can
read.

**Category placement:** Research Management in the Inverted Stack Map.

**Reference contract:** this skill is a thin orchestration wrapper. All field
formats, source hierarchies, and edge-case rules live in:

**`/Users/tomseo/.claude/skills/shared-references/companies-enrichment-spec.md`**

That spec is the single source of truth. Read it before running this skill —
every field written here follows Steps A–E from that document.

## Inputs

One of (any is sufficient; skill resolves the others):
- **LinkedIn company URL** (`linkedin.com/company/<slug>`) — canonical
- **Website domain** (e.g., `thatch.com`)
- **Company name** (e.g., "Thatch")

If only a name is given, resolve the LinkedIn URL via web search first, then
proceed. If the LinkedIn URL is ambiguous (multiple companies share the
name), disambiguate via ContactOut enrichment + industry/size check before
committing.

## Workflow

This skill executes Steps A–E from `companies-enrichment-spec.md` in order:

1. **Step A — Dedup-search** against the Companies DB by normalized domain,
   name fallback. If `Last Enriched` populated → return existing page URL
   (skip). If found with `Last Enriched` empty → proceed to Step B as a
   backfill.
2. **Step B — Create or backfill** via `notion-create-pages` /
   `notion-update-page`. Populate every field per the spec's field table +
   edge-case table. Set icon from ContactOut's `company.logo_url`.
3. **Step C — Sales Nav headcount scrape** for every `Company 🛠️` row (not
   optional — no size-based skipping). Writes Employee Count, Headcount,
   HC Commentary. Skip only for `Fund 💸` / `School 🎓` / Nonprofit / chart
   doesn't render / not authenticated. See
   `shared-references/salesnav-headcount-scrape.md`.
4. **Step D — Deal Digest merge** against the cache at
   `shared-references/deal-digest-cache.json`. Merges into Momentum,
   populates `📰 Deal Digest Mentions` relation, finalizes Revenue.
   See `shared-references/deal-digest-cache.md`.
5. **Step E — Finalize**: write `Last Enriched` = today, return page URL.

## Category routing reminders

(These are baked into the spec's field table but worth surfacing here so the
common cases are obvious.)

- **Publicly-traded financial-services firms** (BlackRock, State Street,
  public asset/alt managers) → `Company 🛠️` with Company Type = `Public
  Company`, Funding Status = `Public`. **Not** `Fund 💸`.
- **Private partnership/fund vehicles** (VC firms, private PE funds, hedge
  fund LPs, family offices acting as pure investors) → `Fund 💸`. Operational
  fields left blank per the edge-case table.
- **Universities / colleges / educational institutions** → `School 🎓`. Only
  Legacy core fields populated; skip Sales Nav and Deal Digest.

## Category-specific field rules

Handled in the spec's edge-case table — summary:

- **`Fund 💸`**: populate identity (Name, Category, Overview, URL, LinkedIn,
  HQ, Founded Year, Company Type) + icon + Last Enriched. Leave Employee
  Count, Revenue, Total Funding, Momentum, Headcount, HC Commentary, Funding
  Status blank.
- **`School 🎓`**: same as Fund — identity-only.
- **`Company 🛠️` + Public, long-public**: Total Funding blank, Momentum
  blank, Funding Status = `Public`. Headcount and HC Commentary still
  populated via Sales Nav.
- **`Company 🛠️` + Acquired**: Employee Count, Headcount, HC Commentary,
  Revenue, Total Funding, Momentum blank; Funding Status = `Acquired`.

## Important rules

- **Speed over confirmation** — run the pipeline, return the URL.
- **Never re-enrich if `Last Enriched` is populated** — credit-saving rule.
- **Dedup by domain first, name second.**
- **No hallucination** — if ContactOut returns nothing for a field, leave
  blank.
- **Return the page URL** on success.

## Interaction with other skills

- **`neg1-enricher`**: invokes THIS skill as a subroutine per distinct
  employer + school when enriching a person. Collects the returned page
  URLs for the Company / Work History / School(s) relations on the -1
  Scanner row.
- **`pipeline-agent`**: may invoke THIS skill when creating a new
  Opportunity to ensure the company exists in Companies DB before linking.
- **`first-pass-diligence`**: reads Companies rows directly; doesn't
  invoke this skill.

## Example

**User:** `add thatch.com to companies`

**Claude:**
1. Step A: no existing Thatch row.
2. Step B: ContactOut → Series B, Index lead, $40M, 174 employees, overview,
   etc. Create row with identity fields + Funding Status = `Series B`.
3. Step C: Sales Nav → 25-month headcount series (45 → 199); writes
   `Employee Count = 199 (April 2026)`, `Headcount` two-line, `HC Commentary`
   intra-window color.
4. Step D: Sep 2025 digest → `$400m post` folded into Apr 2025 round;
   `~$4m ARR` standalone Sep 2025 traction entry. Relation set.
5. Step E: icon set, Last Enriched written. Returns page URL.

## Behavior Rules

### Public asset managers → Company 🛠️, not Fund 💸

Publicly-traded asset managers / financial-services firms (BlackRock, State Street, Blackstone-the-public-entity, etc.) default to `Company 🛠️` in the Companies DB — NOT `Fund 💸`.

**Why:** The standard Category heuristic routes "Asset Management / Hedge Fund / Investment Management" industries to `Fund 💸`. That misfires on public companies Tom tracks as companies in their own right. Confirmed 2026-04-22 when Tom corrected BlackRock from Fund 💸 to Company 🛠️ (Public Company).

**How to apply:** When industry is Asset Management / Investment Management / PE / VC AND the entity is publicly-traded (Company Type = `Public Company`), use `Company 🛠️` with Funding Status = `Public`. Reserve `Fund 💸` for private partnership/fund vehicles (VC firms, private PE funds, hedge fund LPs). If in doubt, ask.

### Revenue & Total Funding enrichment hierarchy

Same rule applies to both `Revenue` and `Total Funding`. Strict order:

1. **Deal Digest data first** — scan the `Momentum` field and entries in the `📰 Deal Digest Mentions` relation. For Revenue: ARR/MRR/revenue. For Total Funding: cumulative raised-to-date (rare in digests). Format: `$X ARR (MMM 'YY Deal Digest)` for Revenue; bare `$Xm`/`$Xb` for Total Funding.
2. **ContactOut estimate second** — call `contactout_enrich_domain`.
   - Revenue: top-level `revenue` field. Format: `$Xm (ContactOut Est.)` / `$Xb (ContactOut Est.)` — lowercase `m`/`b`.
   - Total Funding: `funding.total_funding_usd` when populated. If null, sum `funding.rounds[].money_raised_usd`. Format: bare `$Xm` / `$Xb` (no `(ContactOut Est.)` suffix).
3. **WebSearch third (Total Funding only)** — ContactOut often returns `total_funding_usd: null` with round-level amounts also null for companies with publicly disclosed totals (Navan ~$2.6B, Omada ~$528M). When that happens, WebSearch Crunchbase/PitchBook/press for "[Company] total funding raised". Format: bare `$Xm` / `$Xb` with trailing source marker, e.g. `$2.6b (Crunchbase)`. **Do NOT use WebSearch for Revenue** — ContactOut is the ceiling.
4. **N/A fallback** — if none of the above returns the figure, set `N/A`. Do not leave blank. Obvious-N/A short-circuits (skip WebSearch): government agencies, pre-VC-era private firms (DRW, Lloyd's), wholly-acquired subsidiaries (Refinitiv/LSEG, Setter/Thumbtack).

**What does NOT count as a revenue signal:** funding round data ("Pre-Seed; $1.3m raised", "Series A", "raised $80m") — that's capital, not revenue. Ambiguous figures without a unit ("$200k" alone with no MRR/ARR label) — not usable.

**Edge cases:**
- Public companies where ContactOut returns `N/A` (e.g. Jack Henry — NASDAQ) still get `N/A` per this rule.
- ContactOut returns empty record entirely (404-like): treat as no-data → N/A.
- Stamp `Last Enriched` to today's date on any row touched during an enrichment pass.

**Pagination gotcha:** `notion-query-database-view` returns max 100 results per call with `has_more: true` when more exist — but has NO pagination param. If the DB has >100 rows, a single call silently truncates. Mitigate with a server-side filter (`Revenue: Is empty`), walk via multiple `notion-search` calls, or confirm `has_more` is false before trusting results.

### Always run Headcount scrape for Company 🛠️ rows

Always run the Sales Navigator headcount scrape and populate `Employee Count`, `Headcount`, `HC Commentary` for every `Company 🛠️` row — including public companies and large-cap enterprises. Do NOT skip because "the company is too large" or "HC is less load-bearing" or "it's not a portfolio target."

**Why:** If the row is `Company 🛠️`, the chart is part of the standard enrichment output — not optional. Size- or importance-based discretion is off the table.

**How to apply:** On every create/backfill of a `Company 🛠️` row, execute the Sales Nav recipe at `shared-references/salesnav-headcount-scrape.md` unless one of the spec's explicit skip conditions hits (`Fund 💸`, `School 🎓`, Nonprofit, chart doesn't render, auth error). "Large public company" is NOT a skip condition.

### Headcount enrichment via Sales Nav Growth Insights

Data source: **LinkedIn Sales Navigator → Company page → Growth insights tab**. ContactOut does not return time-series employee counts; WebSearch/Crunchbase/PitchBook don't publish the 1y/2y breakdown structurally.

**Format (preserve exactly):**
```
**12mo (Apr '25-'26):** 667 → 840 (+26%)<br>**24mo (Apr '24-'26):** 509 → 840 (+65%)
```
Thousands separators (commas), Unicode `→` arrow (not `->`), `<br>` between lines.

**Fetch procedure:**
1. **Get numeric Sales Nav company ID.** Vanity URLs like `/sales/company/stripe` don't redirect — must pass numeric ID. Fetch `https://www.linkedin.com/company/{vanity}/` from an authenticated tab (`credentials: 'include'`), regex `fsd_company:(\d+)` on the HTML. If 429, wait 60+ seconds; retry one-at-a-time with ~1.5s delays. If "company unavailable" redirect, the vanity is wrong — fall back to `https://www.linkedin.com/sales/search/company?query=(keywords%3A<name>)`.
2. Navigate to `https://www.linkedin.com/sales/company/{id}`. Sales Nav renders client-side — do NOT `fetch()`; the HTML is a shell.
3. Wait ~5 seconds for Growth insights to render (lazy-loaded).
4. Extract via regex against `document.body.innerText`:
   - `(\d[\d,]*)\s*\n?total employees` — current count
   - `(\d+)%\s*\n?1y growth`
   - `(\d+)%\s*\n?2y growth`
5. Back-solve starting counts: `prev12 = round(current / (1 + g1y/100))`, `prev24 = round(current / (1 + g2y/100))`.
6. **Update Employee Count, Headcount, AND HC Commentary together** — every Headcount write MUST be accompanied by an HC Commentary write. Same `update_properties` call. If SN total differs from Notion Employee Count by more than rounding, update EC to match SN (SN is fresher than ContactOut). Format: `"{count} (April 2026)"`.

**HC Commentary format:** 2–3 sentences describing trajectory in plain prose, referencing absolute numbers and context. Sentence 1: shape of growth with start→end values and deltas. Sentence 2: the "why" (funding, public status, acquisition, restructuring). Optional sentence 3: caveat if Sales Nav scope is narrower than real corporate headcount (parent-only vs. full corporate).

**Fallback:** If Sales Nav page lacks Growth insights (micro-companies, some subsidiaries), set `Headcount` to `N/A`. Don't leave blank.

**Gotchas:**
- **Silent-write failures:** `notion-update-page` sometimes returns success but the property change doesn't persist. Spot-check via `notion-fetch` on 2-3 targets after a batch; retry any that didn't land.
- **Parent-vs-subsidiary:** Sales Nav often shows only the entity carrying the vanity, not full corporate headcount (Moody's ~3,522 vs. ~14–16K full org). Document mismatch in HC Commentary.
- Negative growth (layoffs): preserve as-is, don't "correct" counterintuitive numbers.
- Young companies (<2 years): g2y may be 0% or equal to current. Keep as-is.

### Acquired companies → N/A operational fields

When `Funding Status = "Acquired"`, the company no longer operates standalone. Populating operational fields with scraped numbers (which reflect parent/merged entity) produces misleading signal.

**Set to `N/A`:** Employee Count, Revenue, Total Funding, Momentum, Headcount.

**Leave alone:** Overview (historical context useful), HC Commentary (may carry legitimate post-acquisition analysis), Founded Year, HQ, URL, LinkedIn URL, Category, Company Type (identity, not operational).

Also stamp `Last Enriched` to today's date.

**Trigger:**
- Automatic: any row where `Funding Status = "Acquired"`.
- Manual from Tom: "X was acquired — mark it". Also set `Funding Status = "Acquired"` as part of the same write.
