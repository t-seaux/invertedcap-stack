---
name: add-to-companies
description: >-
  Add a company to Tom's Notion Companies database and run the full enrichment
  pipeline. Dedups by domain/name, creates the row if new, then populates every
  field per the neg1-enricher spec — identity (Name, Category, URL, LinkedIn URL,
  Overview, HQ, Founded Year, Company Type), Revenue, Total Funding, Funding
  Status, Momentum (rounds + Deal Digest mentions), Employee Count, Headcount,
  HC Commentary — using ContactOut + Sales Nav headcount scrape + Deal Digest
  cache as sources. Also sets the page icon from ContactOut's logo. Part of
  Research Management — this is how the company knowledge base gets built up.
  Trigger phrases: "add [company] to companies", "add this company", "log this
  company", "enrich [company]", "create a Companies entry for [X]", "add
  [domain] to companies", "add [LinkedIn company URL]", or any variant indicating
  Tom wants a company logged and enriched in the Companies DB. Also trigger on
  a bare `linkedin.com/company/...` URL or a bare company domain with add/log
  intent. Distinct from `neg1-enricher` (which enriches a -1 Scanner row for
  a person and treats Companies enrichment as a side effect) and `add-to-crm`
  (which creates an Opportunities pipeline entry for a deal). Skill does NOT
  create People entries, Opportunities entries, or any non-Companies artifact.
---

# Add to Companies

Adds a company to the Companies database (Notion data source
`7d50b286-c431-49f5-b96a-6a6390691309`) and runs the full enrichment pipeline,
producing a single cleanly-formatted row that any downstream skill
(pipeline-agent, founder-outreach, first-pass-diligence) can read.

This is the standalone entry point for building the company knowledge base.
`neg1-enricher` uses the same enrichment logic as a side effect when it
resolves a person's Work History + primary employer — both skills share the
field-format spec codified in `neg1-enricher/SKILL.md` (Step 3b field table).

## Reference files (must read at invocation)

- `~/.claude/skills/neg1-enricher/SKILL.md` — canonical field-format spec
  (Step 3b table + Steps 3c-3g). Every field written by this skill follows
  that spec. Do not duplicate format rules here.
- `~/.claude/skills/shared-references/salesnav-headcount-scrape.md` —
  Sales Nav recipe for Employee Count, Headcount, HC Commentary.
- `~/.claude/skills/shared-references/deal-digest-cache.md` —
  Deal Digest cache format, staleness check, rebuild procedure.

## Notion target

**Companies Database** — Data Source ID: `7d50b286-c431-49f5-b96a-6a6390691309`
(Database ID: `dbe72423ba5840b78e9c4cf718156230`)

## Inputs

One of (any is sufficient; skill resolves the others):
- **LinkedIn company URL** (`linkedin.com/company/<slug>`) — canonical
- **Website domain** (e.g., `thatch.com`)
- **Company name** (e.g., "Thatch")

If only a name is given, resolve LinkedIn URL first via web search, then
proceed. If the LinkedIn URL is ambiguous (multiple companies share the name,
as with "Thatch"), disambiguate via ContactOut enrichment + industry/size
check before committing.

## Workflow

### Step 1: Dedup

Search the Companies DB for an existing row using the standard resolver
(identical to `neg1-enricher` Step 3a):

- **Primary key**: normalized root domain. Strip `https://`/`http://`/`www.`,
  trailing slash, and path. `abnormal.ai/` ≡ `https://www.abnormal.ai` ≡
  `abnormal.ai`.
- **Fallback**: fuzzy name match ("Google DeepMind" matches "DeepMind",
  "Meta" matches "Facebook/Meta").

If found with `Last Enriched` populated → return the existing page URL. Do NOT
re-enrich (credit-saving rule). If found with `Last Enriched` empty → proceed
to Step 3 backfill for that row; skip Step 2.

### Step 2: ContactOut enrichment (new-row path)

Call `contactout_enrich_company` (or fallback to `contactout_enrich_domain`)
with the resolved domain. Capture the full response — identity, logo, rounds,
investors, funding_status, size, headquarter, etc.

### Step 3: Create or backfill the Companies row

Use `notion-create-pages` (for new) or `notion-update-page` (for backfill of
existing enrichment-empty row).

**Populate every field per the `neg1-enricher` Step 3b field table.** That
table is the single source of truth for Name, Category, Overview,
userDefined:URL, LinkedIn URL, Company Type, HQ, Founded Year, Funding Status,
Revenue (hierarchy), Total Funding (text with investor list), Momentum
(merged round + digest timeline), Employee Count (text with as-of month),
Headcount (two-line bolded format), HC Commentary (intra-window color),
Last Enriched.

Do not restate format rules in this skill — read them from the referenced
spec file.

### Step 4: Sales Nav headcount scrape

If Category is `Company 🛠️`, run the Sales Nav scrape per
`shared-references/salesnav-headcount-scrape.md`:

1. Resolve Sales Nav company ID via keyword search; verify match via industry
   + employee count before committing (names collide frequently).
2. Navigate to the company page, scrape aria-labels for the monthly headcount
   series.
3. Compute and write `Employee Count`, `Headcount`, `HC Commentary` per the
   neg1-enricher field table.

Skip Sales Nav for `Fund 💸` / `School 🎓` / `Nonprofit` / `Public Company`
categories (either not load-bearing or the category explicitly leaves these
fields blank per the spec's edge-case table).

### Step 5: Deal Digest merge

Read the cache at `shared-references/deal-digest-cache.json`. Check staleness
and rebuild per the cache spec if needed. Look up the normalized company
name, merge matching digest mentions into `Momentum` (valuation merged into
matching-round entries; traction entries as their own timeline lines per
`neg1-enricher` Step 3e merge rules). Populate `📰 Deal Digest Mentions`
relation with matched page URLs.

Skip for `Fund 💸` / `School 🎓` (digest tracks operating companies only).

### Step 6: Logo + icon

Set the page icon to ContactOut's `company.logo_url` (pass as `icon` on
`notion-create-pages`, or via `notion-update-page` with `icon` param). If
ContactOut has no logo, try scraping the `og:image` from the public LinkedIn
company page via WebFetch as a fallback. If still nothing, leave the icon
unset.

### Step 7: Finalize

- Write `Last Enriched` = today's date.
- Return the Notion page URL to the user (with a one-line summary: name,
  category, primary facts surfaced).

## Fund / School / Nonprofit handling

Per Tom's instruction: **for `Fund 💸` and `School 🎓` categories, do NOT
populate operational enrichment fields** (Employee Count, Revenue, Total
Funding, Momentum, Headcount, HC Commentary, Funding Status). Populate only
identity (Name, Category, Overview, URL, LinkedIn URL, HQ, Founded Year,
Company Type) + icon + Last Enriched. Leave the rest blank.

For `Nonprofit` (Company Type on a `Company 🛠️` row): same rule — operational
fields blank per the edge-case table.

## Important rules

- **Speed over confirmation** — do not ask for confirmation before creating.
  Just run the full pipeline and return the page URL.
- **Never re-enrich if `Last Enriched` is populated** — credit-saving rule
  shared with `neg1-enricher`. Manual re-enrichment is a separate ask.
- **Dedup by domain first, name second** — different slugs with the same
  canonical domain are the same company.
- **No hallucination** — if ContactOut returns nothing for a field, leave the
  field blank. Never fabricate.
- **Return the page URL** on success. Caller (user or another skill) will use
  it to relate / link / navigate.

## Example

**User:** `add thatch.com to companies`

**Claude:**
1. Dedups — no existing Thatch row.
2. Calls `contactout_enrich_company` with `thatch.com` → full payload
   (Series B, Index lead, $40M, 174 employees per ContactOut, overview, etc.).
3. Creates Companies row with identity fields + Funding Status = `Series B`.
4. Runs Sales Nav scrape: resolves ID 79132365 (verified via healthcare
   industry + current employee count), scrapes 25-month headcount series
   (45 → 199), writes `Employee Count = 199 (April 2026)`, `Headcount`
   (bolded 12mo/24mo lines), `HC Commentary` (intra-window color).
5. Queries Deal Digest cache: Sept 2025 mention found (`Series B by Index;
   $4M at $400M`). Merges — `$400m post` folded into Apr 2025 round entry;
   `~$4m ARR` as standalone Sep 2025 traction entry. Relation set.
6. Logo: sets icon from ContactOut logo URL.
7. Last Enriched set. Returns page URL.

**User invokes with LinkedIn URL:** `add https://linkedin.com/company/thatch-com`
Same flow — LinkedIn URL resolves to `thatch.com` domain via ContactOut.

## Interaction with other skills

- **neg1-enricher**: when processing a person's Work History or primary
  employer, invokes the same Companies enrichment logic as a side effect.
  Both skills converge on identical row state. If Tom wants a company
  logged WITHOUT a person attached, this skill is the entry point.
- **pipeline-agent**: when creating a new Opportunity from inbox scanning,
  may invoke this skill to ensure the company exists in Companies DB before
  linking from Opportunities.
- **first-pass-diligence**: reads Companies row state directly; doesn't
  invoke this skill.
