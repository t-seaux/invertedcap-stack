---
name: add-to-companies
description: >-
  Add a company to Tom's Notion Companies database and run the full enrichment
  pipeline. Dedups by domain/name, creates the row if new, then populates every
  field per the canonical spec at `shared-references/companies-enrichment-spec.md` —
  identity (Name, Category, URL, Overview, HQ, Founded Year),
  Total Funding, Momentum (rounds + Deal Digest mentions), Employee Count,
  Headcount, HC Commentary — using Exa+LLM extraction + Sales Nav headcount
  scrape + Deal Digest cache as sources. Also sets the page icon. Part of
  Research Management — this is how the company knowledge base gets built up.
  Trigger phrases: "add [company] to companies", "add this company", "log this
  company", "enrich [company]", "create a Companies entry for [X]", "add
  [domain] to companies", "add [LinkedIn company URL]", or any variant
  indicating Tom wants a company logged and enriched in the Companies DB. Also
  trigger on a bare `linkedin.com/company/...` URL or a bare company domain
  with add/log intent. Distinct from `neg1-enricher` (person-level; invokes
  THIS skill as a subroutine per employer/school) and `add-to-crm`
  (Opportunities pipeline). Skill does NOT create People entries, Opportunities
  entries, or any non-Companies artifact. ALWAYS runs the full Steps A–E
  enrichment pipeline on every direct invocation — no minimal/partial variant.
  Only skip is the credit-saving rule: if `Last Enriched` is already populated,
  return the existing row without re-enriching.
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

### Mode B (webhook): existing Companies row, URL-populated

When invoked with `{mode: "webhook", page_id: "<notion-page-id>"}`, the row already exists in the Companies DB — Tom (or another skill) created it and populated the `URL` property. The webhook (notion-webhook Cloudflare Worker, `companies-url-populated` trigger) routes here. Do **not** create a new row, do **not** ask Tom anything, do **not** rerun dedup-search.

Behavior:
1. Fetch the existing page by `page_id`. If `Last Enriched` is already populated, log "already enriched" and exit (the worker already gates on this, but re-check in case of races).
2. Read the row's `URL` (website domain) and `Name`. Resolve a LinkedIn company URL via web search for use as an Exa fetch source, then proceed straight into Steps B–E. (LinkedIn URL is no longer persisted to the row — Tom deleted that property 2026-05-06.)
3. Skip Step A's dedup-search entirely — the page_id IS the dedup answer.
4. Apply the unattended-execution guard at `/Users/tomseo/.claude/scheduled-tasks/SHARED_SAFETY.md` — never ask, skip-and-log on missing data, always reach Slack with a result line.
5. On completion, post a Slack alert via `send-alert` using the canonical template below.

### Slack alert template (Mode B + manual)

Use this exact shape via `~/.claude/skills/send-alert/send.sh`. The first line is the header (bold, with the company name + Notion link). The bullets follow on the **very next line** — no blank line in between (md_to_blocks treats a blank line as a `\n\n` paragraph spacer, which renders as visible empty space; that gap is a bug). Use GFM `**bold**`, not Slack mrkdwn.

```
**🪙 add-to-companies — <Company Name> enriched ([Notion](<page_url>))**
- **Category:** <Category emoji + label>
- **HQ:** <metro> (<actual city/region if differs from metro>)
- **Total Funding:** <amount with investors> or "—"
- **Headcount:** <current count> (<Mon 'YY>); 12mo +X% / 24mo +Y%
- **Deal Digest:** <one-line summary; "no mentions in cache" if empty>
```

Rules:
- Header line is GFM bold (double asterisks). The 🪙 emoji is part of the header, not a separate icon-binding.
- Bullets use the GFM dash convention (`- `), not `• `. md_to_blocks renders `- ` as Slack bullets correctly. Do NOT use `*` or `•`.
- The bullet list starts on the line **immediately after** the header line — single `\n` separator, never `\n\n`.
- For Funds / Schools, omit the Total Funding / Deal Digest / Headcount lines (those fields are categorically blank per the spec).
- For Companies that hit Step C / D no-ops (no Sales Nav chart, no Deal Digest mentions), still include the corresponding bullet — use "no mentions in cache" / "Sales Nav skipped (no chart)" rather than dropping the line.

## Workflow

This skill executes Steps A–E from `companies-enrichment-spec.md` in order:

1. **Step A — Dedup-search** against the Companies DB by normalized domain,
   name fallback. If `Last Enriched` populated → return existing page URL
   (skip). If found with `Last Enriched` empty → proceed to Step B as a
   backfill.
2. **Step B — Exa enrichment + create/backfill.** Resolve LinkedIn URL + website
   via WebSearch if not given (LinkedIn URL is used as an Exa fetch source only —
   not persisted to the row). Fetch via Exa (LinkedIn page first, website
   fallback). Pass Exa text to Claude (Haiku) with the extraction prompt from
   the spec — returns hq, founded_year, overview, total_funding.
   Apply HQ metro rollup. Then `notion-create-pages` or `notion-update-page`
   per the spec's field table. Set icon per the source hierarchy in the spec.
   **Before any `notion-create-pages` call into the Companies data source
   (`7d50b286-c431-49f5-b96a-6a6390691309`), run `touch /tmp/.addcompanies-bypass`** —
   a PreToolUse hook at `~/.claude/hooks/gate-db-creation.sh` blocks direct
   writes unless this marker is fresh (≤5 min). Marker auto-expires, no cleanup
   needed. `notion-update-page` is not gated.
3. **Step C — Sales Nav headcount scrape** for every `Company 🛠️` row (not
   optional — no size-based skipping). Writes Employee Count, Headcount,
   HC Commentary. Skip only for `Fund 💸` / `School 🎓` / Nonprofit / chart
   doesn't render / not authenticated. See
   `shared-references/salesnav-headcount-scrape.md`.
4. **Step D — Deal Digest merge** against the cache at
   `shared-references/deal-digest-cache.json`. Merges into Momentum,
   populates `📰 Deal Digest Mentions` relation.
   See `shared-references/deal-digest-cache.md`.
5. **Step E — Finalize**: write `Last Enriched` = today, return page URL.

## Category routing reminders

(These are baked into the spec's field table but worth surfacing here so the
common cases are obvious.)

- **Publicly-traded financial-services firms** (BlackRock, State Street,
  public asset/alt managers) → `Company 🛠️`. **Not** `Fund 💸`.
- **Private partnership/fund vehicles** (VC firms, private PE funds, hedge
  fund LPs, family offices acting as pure investors) → `Fund 💸`. Operational
  fields left blank per the spec.
- **Universities / colleges / educational institutions** → `School 🎓`. Only
  identity fields populated; skip Sales Nav and Deal Digest.

## Category-specific field rules

Handled in the spec's edge-case table — summary:

- **`Fund 💸`**: populate identity (Name, Category, Overview, URL, HQ) + icon
  + Last Enriched. Leave Founded Year, Employee Count, Total Funding, Momentum,
  Headcount, HC Commentary blank.
- **`School 🎓`**: same as Fund — identity-only.
- **`Company 🛠️` + Public, long-public**: Total Funding blank, Momentum
  blank. Headcount and HC Commentary still populated via Sales Nav.
- **`Company 🛠️` + Acquired**: Employee Count, Headcount, HC Commentary,
  Total Funding, Momentum blank.

## Important rules

- **Speed over confirmation** — run the pipeline, return the URL.
- **Never re-enrich if `Last Enriched` is populated** — credit-saving rule.
- **Dedup by domain first, name second.**
- **No hallucination** — blank > fabricated. If Exa returns nothing for a
  field, leave blank.
- **Always set icon** — iconless row is a bug.
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
2. Step B: WebSearch resolves LinkedIn URL → Exa fetches LinkedIn page → LLM
   extracts overview, HQ="San Francisco", founded_year=2021, total_funding="$40m".
   Create row with identity fields + Total Funding = `$40m (Index Ventures, ...)`.
3. Step C: Sales Nav → 12mo/24mo headcount series; writes `Employee Count`,
   `Headcount`, `HC Commentary`.
4. Step D: Sep 2025 digest → `$400m post` folded into round entry as Momentum;
   relation set.
5. Step E: icon set, Last Enriched written. Returns page URL.

## Behavior Rules

### Public asset managers → Company 🛠️, not Fund 💸

Publicly-traded asset managers / financial-services firms (BlackRock, State Street, Blackstone-the-public-entity, etc.) default to `Company 🛠️` in the Companies DB — NOT `Fund 💸`.

**Why:** The standard Category heuristic routes "Asset Management / Hedge Fund / Investment Management" industries to `Fund 💸`. That misfires on public companies Tom tracks as companies in their own right. Confirmed 2026-04-22 when Tom corrected BlackRock from Fund 💸 to Company 🛠️ (Public Company).

**How to apply:** When industry is Asset Management / Investment Management / PE / VC AND the entity is publicly-traded, use `Company 🛠️`. Reserve `Fund 💸` for private partnership/fund vehicles. If in doubt, ask.

### Total Funding enrichment

Source: Exa text + LLM extraction. The LLM returns `total_funding` (e.g., `"$40m"`) and `total_funding_evidence` (supporting quote). If Exa returns nothing, leave Total Funding blank — do not write `N/A`.

Format: `$Xm (Investor1, Investor2, ...)` — dollar amount from LLM extraction; investor names from the Exa text if mentioned (cap at 4–5, prefer tier-1 names). If no investors visible, just `$Xm`.

Leave blank for `Fund 💸` / `School 🎓`.

**Public, long-public companies:** leave Total Funding blank (no tracked rounds).

**Acquired companies:** leave Total Funding blank.

### Always run Headcount scrape for Company 🛠️ rows

Always run the Sales Navigator headcount scrape and populate `Employee Count`, `Headcount`, `HC Commentary` for every `Company 🛠️` row — including public companies and large-cap enterprises. Do NOT skip because "the company is too large" or "HC is less load-bearing" or "it's not a portfolio target."

**Why:** If the row is `Company 🛠️`, the chart is part of the standard enrichment output — not optional. Size- or importance-based discretion is off the table.

**How to apply:** On every create/backfill of a `Company 🛠️` row, execute the Sales Nav recipe at `shared-references/salesnav-headcount-scrape.md` unless one of the spec's explicit skip conditions hits (`Fund 💸`, `School 🎓`, Nonprofit, chart doesn't render, auth error). "Large public company" is NOT a skip condition.

### Headcount enrichment via Sales Nav Growth Insights

Data source: **LinkedIn Sales Navigator → Company page → Growth insights tab**. Exa does not return time-series employee counts.

**Format (preserve exactly):**
```
**12mo (Apr '25-'26):** 667 → 840 (+26%)<br>**24mo (Apr '24-'26):** 509 → 840 (+65%)
```
Thousands separators (commas), Unicode `→` arrow (not `->`), `<br>` between lines.

**Fetch procedure:**
1. **Get numeric Sales Nav company ID.** Fetch `https://www.linkedin.com/company/{vanity}/` from an authenticated tab, regex `fsd_company:(\d+)` on the HTML. If 429, wait 60+ seconds; retry one-at-a-time.
2. Navigate to `https://www.linkedin.com/sales/company/{id}`. Wait ~8 seconds for Growth insights to render.
3. Extract via regex against `document.body.innerText`:
   - `(\d[\d,]*)\s*\n?total employees` — current count
   - `(\d+)%\s*\n?1y growth`
   - `(\d+)%\s*\n?2y growth`
4. Back-solve: `prev12 = round(current / (1 + g1y/100))`, `prev24 = round(current / (1 + g2y/100))`.
5. Write `Employee Count`, `Headcount`, and `HC Commentary` in one `update_properties` call.

**HC Commentary format:** 2–3 sentences describing trajectory in plain prose. Don't restate anchor numbers. Capture round-linked inflections, intra-window acceleration, parent-vs-subsidiary scope caveats.

**Fallback:** If Sales Nav lacks Growth insights, set `Headcount` to `N/A`. Don't leave blank.

**Gotchas:**
- **Silent-write failures:** `notion-update-page` sometimes returns success but the property change doesn't persist. Spot-check via `notion-fetch` on 2–3 targets after a batch; retry any that didn't land.
- **Parent-vs-subsidiary:** Sales Nav often shows only the entity carrying the vanity slug, not full corporate headcount. Document mismatch in HC Commentary.
- Negative growth (layoffs): preserve as-is.

### Acquired companies → blank operational fields

When a company is acquired, it no longer operates standalone. Leave Employee Count, Total Funding, Momentum, Headcount blank. Preserve Overview, HQ, URL, Category, Founded Year. Stamp Last Enriched.
