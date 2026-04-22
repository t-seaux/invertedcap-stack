---
name: neg1-enricher
description: >-
  Enrichment + evaluation primitive for pre-founder (-1) candidates. Takes a LinkedIn URL,
  builds a fully-enriched -1 Scanner row in Notion (ContactOut person + per-employer Company
  Search, online presence research, Companies DB relations), then applies Tom's signal
  framework rubric to score the candidate and write the verdict (Eval Score + Eval Breakdown
  + Signals + Working Description + Claude Rec + Eval Summary). Pure spike-based MAX, no
  Intentionality gate (per FRAMEWORK_PRD.md v0.4). One-shot company enrichment (Company
  Search per distinct employer, skipped if Last Enriched is already populated — credit-
  saving). Supports a `--score-only` mode that re-runs rubric application against existing
  row data without re-fetching ContactOut (used for rubric drift checks per PRD §6.7).
  Manual trigger only. Scheduled enrichment of unenriched rows is handled by pipeline-agent
  Task 6, which invokes THIS skill as its primitive. Trigger phrases: "-1 scan [URL]", "-1
  scanner", "add to scanner", "add to -1 scanner", "add to sourcing", "source these", "scan
  these profiles", "enrich these LI URLs", "add these to -1", "scanner", "enrich [LI URL]",
  "score [name]", "score the new -1 entries", "run founder scoring", "score this profile",
  "rescore [name]", "re-score [name]", "score-only", or any variant involving LinkedIn URLs
  paired with sourcing/enrichment/scoring intent. Also trigger when the user pastes a batch
  of LinkedIn URLs without further context — the default action for a bare list of LinkedIn
  URLs is to run this skill. Distinct from "add to contacts" (People DB), "add to crm"
  (Opportunities pipeline), and "founder-outreach" (drafting only — runs after this skill
  produces a Reach Out verdict). Part of pipeline management.
---

# -1 Enricher

Enrichment + evaluation primitive for pre-founder candidates. Takes a LinkedIn URL, produces a fully-enriched and **fully-evaluated** -1 Scanner row that Tom can act on. Per Tom's mental model: enrichment is "gather info/context, then apply my frameworks and intelligence to further enrich" — scoring IS a form of enrichment (it enriches the row with a rubric-derived verdict). Drafting the cold email lives in `founder-outreach` and only runs *after* Tom flips Claude Rec to ✅.

## Notion Targets

- **-1 Scanner Database** — Data Source ID: `32c00bef-f4aa-80a5-923b-000b83921fa3`
- **Companies Database** — Data Source ID: `7d50b286-c431-49f5-b96a-6a6390691309`

## Workflow

### Step 1: Enrich Each Profile

For each LinkedIn URL, call `contactout_enrich_linkedin_profile` with `profile_only: false`. This returns the full profile payload — name, headline, location, seniority, company details, experience history, education, skills, languages, publications — along with email addresses (personal and work).

Process URLs sequentially when handling a batch.

### Step 2: Determine the Primary Company and Role

From the enriched profile's experience array, identify the **primary current role** — the person's actual full-time day job.

**Filter out these as non-primary (secondary activities):**
- Advisory / Advisor roles
- Board member / Board of Directors positions
- Venture Scout / Scout roles
- Mentor / EIR (Entrepreneur in Residence)
- Angel investor side activities
- Part-time, volunteer, or informal affiliations

**Filter out study abroad programs entirely:**
- Semester abroad, year abroad, exchange programs, and similar short-term study abroad affiliations at foreign universities should be **ignored completely** — they should not appear in School(s), Work History, Field(s) of Study, or any other field.
- Common signals: the education entry mentions "semester abroad," "study abroad," "exchange program," "visiting student," or has a duration of ~1 semester (3–6 months) at a university that differs from the person's degree-granting institution.
- These are temporary programs, not meaningful affiliations. Do not create Companies database entries for study abroad schools, and do not include study abroad fields of study in the Field(s) of Study field.

**How to identify the primary:**
- Look for `is_current: true` entries first
- Among current roles, prefer the one with the longest tenure or that reads as full-time
- If the headline leads with a role, that's typically the primary
- The exception: people whose full-time job IS investing (GP at a fund, VC Partner, family office principal) — their investing role is the primary

### Step 3: Resolve Companies in Notion

The **Company** field (primary employer, limit 1), **Work History** field (all employers), and **School(s)** field (all schools from education) are all **relations** to the Companies database. For every company or school that needs to be linked:

**3a. Dedup-search** — use `notion-search` with `data_source_url: "collection://7d50b286-c431-49f5-b96a-6a6390691309"`.
- **Primary key: URL (normalized domain).** Search by the `URL` field. Normalize both sides for comparison: strip `https://`/`http://`/`www.`, trailing slash, and any path (`abnormal.ai/` ≡ `https://www.abnormal.ai` ≡ `abnormal.ai`). Domain-equivalent match is authoritative — different names with same domain are the same company.
- **Fallback: Name fuzzy match.** If no URL hit (or no URL available), fall back to name matching. Be forgiving ("Google DeepMind" matches "DeepMind", "Meta" matches "Facebook/Meta").
- If found, capture the row's `Last Enriched` value — determines whether Step 3c backfill is needed.

**3b. Create if not found** — use `notion-create-pages` with parent `data_source_id: "7d50b286-c431-49f5-b96a-6a6390691309"`.

For **Company**-category entries, call `contactout_enrich_company` to pull the full Company Search response (falling back to `contactout_enrich_domain` if it misses). Populate the fields below. For **School**-category entries, only populate the Legacy core fields (use `contactout_enrich_domain` as before) — skip all Enrichment fields. For **Fund**-category entries, populate both tiers.

| Companies Field | How to Populate |
|----------------|----------------|
| **Name** | Company name from ContactOut data |
| **Category** | "School 🎓" if the entity is a university, college, or educational institution. "Fund 💸" if the company's industry involves Venture Capital, Private Equity, Investment Management, Hedge Funds, Asset Management, or if the entity is clearly an investment fund. Otherwise "Company 🛠️". |
| **Overview** | **Always populate for every new entity.** For the primary company, use `company.overview`. For work history companies and schools, use `description` from `contactout_enrich_domain` (or `overview` from Company Search). If nothing returned, leave blank — never fabricate. |
| **userDefined:URL** | Company website — this is **also the canonical dedup key**. For primary company: `company.website`. For work history: `https://{domain}` from `experience[].domain`. Omit if unavailable. Dedup compares normalized root domain (strip protocol/www/path). |
| **Total Funding** | Text field. Format: `$Xm (Investor1, Investor2, ...)` with lowercase `m`/`b`/`k`. Aggregate of disclosed funding + major investors in parens. **Free-lane data hierarchy** — dollars and investors come from different sources because no single free source gives both comprehensively. **Dollar amount**: (1) PDL `/v5/company/enrich` → `total_funding_raised` (most reliable aggregate; see `shared-references/peopledatalabs.md`) → (2) ContactOut `company.funding.total_funding_usd` (fallback) → (3) sum of ContactOut `funding.rounds[].amount_usd` (last resort). **Investor list**: ContactOut's latest round's `funding.rounds[0].investor_names` — this captures ~80-90% of the "who backed this" signal because major leads typically follow on in later rounds (verified on Thatch: Index-led Series B's `investor_names` included Index, a16z, GC, ADP Ventures, PeopleTech, SemperVirens, The General Partnership — effectively the full history's major participants). Cap at 4-5 names, comma-separated, de-duped, prefer tier-1 firms. Leave blank if no investors known. Leave whole field blank for Funds / Schools / Nonprofits (categorically N/A). Examples: Thatch → `$84m (Index Ventures, Andreessen Horowitz, General Catalyst)`; PDL $12m no investor data → `$12m`; ContactOut aggregate $1.3B with known leads → `$1.3b (Sequoia, a16z, Lightspeed)`. **Paid upgrade path** if free-lane coverage feels insufficient: Sacra (subscription + MCP already wired), Harmonic.ai (enterprise), Crunchbase API (enterprise). |
| **LinkedIn URL** | Company's LinkedIn URL from Company Search `linkedin_url`, or from the `experience[].company.linkedin_url` on person enrichment. |
| **Employee Count** | Text field. Format: `{count} ({Full Month} {YYYY})` — count followed by as-of month in parens, full month name + 4-digit year (e.g., `April 2026`, not `Apr '26`). **Primary: Sales Nav final series point** (populated by Step 3d — the most-recent month's headcount from the scraped chart; the month in parens is that series point's month). **Fallback: ContactOut's `size` / `employees`** only if Sales Nav scrape fails (auth error, no chart, or entity is Fund 💸 / School 🎓 which skips Sales Nav). In the fallback case, use the current enrichment month in parens. Example: Thatch → `199 (April 2026)`. Reason for Sales Nav primary: ContactOut's value lags — observed 174 vs. Sales Nav's current 199 for Thatch (April 2026), ~14% understated. Sales Nav refreshes monthly. |
| **Founded Year** | From `founded_at` (year only, integer). |
| **Company Type** | **1:1 with ContactOut's LinkedIn-derived `type`** — store verbatim. Valid values: `Privately Held`, `Public Company`, `Nonprofit`, `Educational Institution`, `Government Agency`, `Partnership`, `Self-Employed`, `Sole Proprietorship`. Leave blank if ContactOut returns nothing (don't force a guess). Note: LinkedIn has no "Acquired" type — acquisition status lives on `Funding Status` instead. |
| **HQ** | Single-select, rolled up to **metro area** (US) or **country** (international). From ContactOut's `headquarter` or `country`, map the city/region to the corresponding select option. US metro rollup examples: Brooklyn/Jersey City/Stamford → `New York`; Berkeley/Oakland/Palo Alto/Mountain View/San Jose/South Bay → `San Francisco`; Santa Monica/Pasadena/Orange County → `Los Angeles`; Cambridge/Somerville → `Boston`; Bellevue/Redmond → `Seattle`; Northern VA/MD suburbs → `Washington DC`; Boulder → `Denver`; Lehi/Provo → `Salt Lake City`; Durham/Chapel Hill/RTP → `Raleigh`. International rollup to country: London → `United Kingdom`; Tel Aviv → `Israel`; Berlin → `Germany`; Paris → `France`; Amsterdam → `Netherlands`; Toronto → `Canada`; Mumbai/Bangalore → `India`. If the city doesn't match any explicit option → `Other`. If ContactOut returns no location → `Unknown`. No-HQ / distributed companies → `Remote`. Valid options: see Companies DB schema for the full list. |
| **Revenue** | Text field. Source hierarchy: **Deal Digest wins over ContactOut** when both have a value. Format depends on source: **Deal Digest:** `$Xm ARR (Mon 'YY Deal Digest)` — pick the most recent digest mention that carries an ARR number (date format: 3-letter title-case month + apostrophe + 2-digit year + space + `Digest`, e.g., `Sep '25 Deal Digest`). **ContactOut fallback:** `$Xm (ContactOut Est.)` — ContactOut's `revenue` field converted to millions with lowercase `m`/`b`/`k`, with the word `Revenue` and `Est.` source-tag to flag that it's an aggregator estimate (Zoominfo/Apollo-sourced, less reliable than digest). Use only when no Deal Digest ARR mention exists. Examples: Thatch with Sep 2025 digest → `$4m ARR (Sep '25 Deal Digest)`; company with only ContactOut data → `$1m Revenue (ContactOut Est.)`. Leave blank if neither source has a value. Lowercase units throughout. |
| **Funding Status** | **1:1 with ContactOut's `funding.funding_status`** — store verbatim. Valid values: `Pre-Seed`, `Seed`, `Angel`, `Series A`–`Series G`, `Series H+` (consolidated), `Convertible Note`, `Debt Financing`, `Corporate Round`, `Private Equity`, `Secondary Market`, `Grant`, `IPO`, `Post-IPO Equity`, `Post-IPO Debt`, `Acquired`, `Bootstrapped`, `Public`, `Unknown`. **Leave blank for `Fund 💸`, `School 🎓`, and Nonprofit entities** — funding_status is categorically inapplicable, and Tom's "Hide when empty" view config handles the display. For Operating companies (`Company 🛠️`): if ContactOut returns a value matching an option, use verbatim; if it returns a round type not in the list, use `Unknown` (and flag for spec extension); if ContactOut returns nothing, use `Unknown`. Same blank-when-inapplicable rule applies to `Total Funding ($m)` and `Funding Rounds`. |
| **Momentum** | Plain-text timeline of funding rounds and Deal Digest traction mentions. **Newest first** (descending by date). One entry per line, no bullet prefix, no bold, no blank lines between. Date format: `{Mon YYYY}:` where Mon is 3-letter title case (`Jan`, `Feb`, `Mar`, `Apr`, `May`, `Jun`, `Jul`, `Aug`, `Sep`, `Oct`, `Nov`, `Dec`). Two entry types: (a) **Round entries** — `{Mon YYYY}: {Round Type}[ led by {lead}]; ${amount}m[ on ${valuation}m post]`. Round type + date + amount + lead from ContactOut `funding.rounds[]`; post-money valuation pulled from any Deal Digest mention that references this round (valuation is a round attribute, not time-sensitive — merge regardless of digest date). Omit the `led by` clause if no lead known; omit the `on $Xm post` clause if no valuation. (b) **Traction (Deal Digest) entries** — `{Mon YYYY}: ~$Xm ARR (Deal Digest)` or similar — used when a digest mention carries ARR, growth rate, or other time-indexed traction data. Keep traction and rounds as SEPARATE entries unless the traction number was explicitly captured at the same time as the raise (±1 month). **Amount formatting:** always lowercase `m`, `b`, `k` (e.g., `$40m`, `$1.2b`, `$500k`). >= $1b → "$X.Xb", >= $1m → "$Xm" or "$X.Xm" if decimals matter, >= $1k → "$Xk". Null `amount_usd` → "undisclosed". **Separator on round lines:** use semicolon (`;`) between the `led by {lead}` clause and the amount clause — NOT en dash or hyphen. Semicolons also separate clauses within traction lines (e.g., `$26m ARR; targeting $100m '26E`). **Multiple investors** are comma-separated, not slash or dash (e.g., `Charles Schwab, Google Ventures` — never `Charles Schwab / Google Ventures`). **Source tag on every line (required):** every Momentum line ends with the data source in parens. `(ContactOut)` for round lines where all fields came from ContactOut; `(Deal Digest)` for traction lines sourced from a digest; `(ContactOut, Deal Digest)` for round lines where the core round fields came from ContactOut but valuation/detail was merged from a digest. Example: `**Apr 2025:** Series B led by Index Ventures; $40m on $400m post (ContactOut, Deal Digest)` — the $40m + Series B + Index + date came from ContactOut; $400m post came from a later Deal Digest mention, folded in per Step 3e merge rules. **Round Type normalization:** if ContactOut returns `"Series unknown"` or a placeholder type, render as `Unknown`. **Decomposition of digest bullets:** a single digest bullet often carries both round data and traction data (e.g., `Thatch: Series B by Index; $4M at $400M` contributes valuation `$400m post` → folded into Apr 2025 round entry, AND traction `~$4m ARR` → standalone Sep 2025 entry). Split accordingly. "Deal Digest" is capitalized in the suffix. **If `funding.rounds[]` is empty AND no Deal Digest mentions exist, leave blank** — never write literal `[]`. **Bold the date prefix** with colon inside bold (`**Sep 2025:**`, `**Apr 2025:**`). **Multi-period traction** (arrow progressions, YoY multipliers, forward targets): **always one line per digest mention**, tagged with the digest's own date (when Tom saw it). Preserve the shape of the original bullet — arrows, multipliers, target clauses all stay. Use `→` for progressions. Examples: `**Mar 2026:** $1m → $7m → $26m ARR in '25A; targeting $100m '26E (Deal Digest)` (Campus); `**Sep 2025:** $30m ARR, 10x YoY (Deal Digest)`. **When the digest provides explicit anchor dates for each point in a progression**, include them as parentheticals attached to each number — still one line: `**Mar 2026:** $1.7m (Dec '25) → $3.5m (Jan '26) → $5.6m (Mar '26) ARR (Deal Digest)`. Multiplicative growth attaches to the point-in-time number on the same line. Forward targets share the line with the most recent actual. **Preserve relative dates verbatim** ("last year", "LTM", "in Q1") — do NOT translate to absolute years; the digest date prefix provides the anchor. **Drop thin mentions** ("doing well", name-only, pure punditry) — but keep raise signals ("raising", "raising B", "raising A later this year"). **Consolidate same-month digest mentions** only when they carry identical info (e.g., Xbow's Feb 18 and Feb 25 both saying "$1.1b by DFJ" → one entry). If two same-month digests carry different content, keep both (both tagged `Mon YYYY:`). **Same round mentioned across multiple digests over time** → ONE round entry, tagged to the **first digest that reported it**, enriched with richer detail (raise amount, investor names, revised valuation) from later mentions. The later digest retains its own entry ONLY for traction data (ARR, growth rate, targets) — not for re-stating the same round. Example (Xbow): round first surfaced Feb 2026 at `$1.1b; DFJ`; Mar 2026 digest added `$120m` raise amount + ARR progression → the $120m is folded into the Feb 2026 round entry, the ARR progression becomes a separate Mar 2026 traction entry. Example (Thatch): `**Sep 2025:** ~$4m ARR (Deal Digest)\n**Apr 2025:** Series B led by Index Ventures; $40m on $400m post` |
| **Headcount** | Two-line text field with growth numbers at a glance. Format: `**12mo (Mon 'YY-'YY):** {hc_start} → {hc_end} (+X%)\n**24mo (Mon 'YY-'YY):** {hc_start_24} → {hc_end} (+X%)`. Each window on its own line, separated by a single `\n` newline (not semicolon, not blank line). Period label (`12mo` / `24mo`) comes first, followed by the date range in parens. When start and end months match (the common case), collapse year range to `Mon 'YY-'YY` — don't repeat the month. Use plain hyphen (`-`) between years, no spaces, apostrophe on both. Use arrow `→` for the numeric progression. Include `+` sign for growth; use `-` for contraction (e.g., `-12%`). Compute X% as `(end - start) / start * 100`, round to integer. `end_date` is always the most recent Sales Nav data point (today's month); `start_date` is that month minus 12 or 24 months. Example: `**12mo (Apr '25-'26):** 98 → 199 (+103%)\n**24mo (Apr '24-'26):** 45 → 199 (+342%)`. Edge case — if start and end months differ (shouldn't happen with monthly Sales Nav data but possible with truncation), use full form with en dash: `**12mo (Apr '25 – Mar '26):** ...`. If Sales Nav doesn't cover a full 12mo or 24mo window, omit that line and flag in HC Commentary. Leave blank if scrape fails entirely. |
| **HC Commentary** | Short prose adding intra-window color that Headcount's anchor numbers can't convey. **Do NOT restate the Headcount anchor numbers** (no "45 → 199 over 25mo" opener — that duplicates the 24mo row). Focus on: (a) round-linked inflection points (e.g., `Series B closed Apr '25 at 98 employees`) — useful for tenure-overlap reasoning in founder-outreach Mode 1; (b) intra-window acceleration or deceleration (e.g., `+56% in H2 '25`, `flat Jun–Aug '25 before reacceleration`); (c) current hiring velocity (e.g., `adding ~12/month since Jan '26`). Dates use `Mon 'YY` format for consistency with Headcount. Example: `Series B closed Apr '25 at 98 employees; hiring accelerated through H2 '25 (98 → 153, +56% in 6mo). Adding ~12 employees/month since Jan '26`. Leave blank if no non-trivial intra-window events or velocity signals to report. |
| **📰 Deal Digest Mentions** | Relation — JSON array of Notion page URLs for every Deal Digest page that mentions this company. Populated alongside Momentum writes. See Step 3e. |
| **Last Enriched** | Today's date (ISO-8601). This is the credit-saving marker — its presence means "don't re-enrich." |

Set the page **icon** to the entity's logo URL from ContactOut. Primary company: `company.logo_url`. Work history: `experience[].logo_url` or Company Search `logo_url`. Schools: `contactout_enrich_domain` → `logo_url`. Pass as `icon` on `notion-create-pages`.

**Funding-field edge-case handling.** Use this table when Total Funding ($m), Funding Rounds, and Funding Status could each go multiple ways:

| Company state | Total Funding | Momentum | Funding Status |
|---|---|---|---|
| Public, pre-IPO rounds tracked | aggregate of pre-IPO round amounts | prose timeline of pre-IPO rounds + digest mentions | `Public` |
| Public, long-public (no rounds tracked) | blank | blank | `Public` |
| Private startup with disclosed rounds | aggregate of rounds | merged rounds + digest timeline | matching series (`Seed`, `Series A`, etc.) |
| Private startup, top-line only | ContactOut's `funding.total_funding_usd` fallback | digest-only entries if any, else blank | ContactOut's `funding_status` or `Unknown` |
| Bootstrapped | blank | digest-only entries if any, else blank | `Bootstrapped` |
| Acquired | pre-acquisition total | pre-acquisition rounds + digest timeline | `Acquired` |
| Fund 💸 (VC/PE firm) | blank | blank | blank (categorically N/A) |
| School 🎓 | blank | blank | blank |
| Nonprofit | blank | blank | blank |

**3c. Backfill existing rows** — if Step 3a found a row but `Last Enriched` is empty:
- This row was created before the enrichment architecture existed. Backfill once.
- Call `contactout_enrich_company` (or fallback to domain) and populate only the **new** fields: LinkedIn URL, Employee Count, Founded Year, Company Type, HQ, Revenue, Total Funding, Funding Status, Momentum, Headcount, HC Commentary, 📰 Deal Digest Mentions, Last Enriched.
- **Do NOT overwrite** existing Name, Category, Company Overview, URL, Total Funding, icon — Tom may have curated these.
- Skip backfill for School-category rows.

**3d. Compute Headcount + HC Commentary (+ set Employee Count)** — scrape LinkedIn Sales Navigator's monthly headcount chart. Full recipe in `shared-references/salesnav-headcount-scrape.md`.

1. Resolve the company's Sales Nav ID: navigate Chrome to `https://www.linkedin.com/sales/search/company?keywords=<name>` and scrape `a[href*="/sales/company/"]` from the first result. Verify the match with a quick industry/description check before committing — multiple companies share names (e.g., there are 5+ "Thatch" entities; only one is the healthcare platform).
2. Navigate to `https://www.linkedin.com/sales/company/<id>` (8s wait). Click "View headcount growth" if the chart isn't inline (3s wait).
3. Scrape all `aria-label` attributes matching `^(\d[\d,]*)\s+employees?\s+in\s+(\w+)\s+(\d{4})`. Parse into a `{YYYY-MM: count}` dict.
4. **Set `Employee Count`** to `{count} ({Full Month} {YYYY})` — the most-recent month's value with its month in parens, full month name + 4-digit year (e.g., `199 (April 2026)`). Replaces ContactOut's stale aggregator number.
5. Compute 12mo growth: `(hc_now - hc_12mo_ago) / hc_12mo_ago * 100`, round to integer. Same for 24mo. If the exact lookback month isn't present, use the nearest month within ±2 and flag truncation in HC Commentary.
6. Write `Headcount` field: `**12mo (Mon 'YY-'YY):** {hc_start} → {hc_end} (+X%)\n**24mo (Mon 'YY-'YY):** {hc_start_24} → {hc_end} (+X%)`. Two lines, each starting with period label then date range in parens. Arrow `→` between counts. Omit a line if data is too thin.
7. Write `HC Commentary` as prose — intra-window color only (do NOT restate Headcount anchor numbers). Focus on round-linked inflection points, acceleration/deceleration windows, and current hiring velocity. Leave blank if nothing non-trivial to add.

**Skip conditions:**
- Category is `Fund 💸`, `School 🎓`, or Nonprofit → leave all three fields blank.
- Sales Nav returns no chart (small company, private, not yet backfilled) → leave blank.
- Chrome not authenticated or redirect to `/upsell` → surface error, do not retry, leave blank.
- `Company Type` is `Public Company` — HC is available but less load-bearing; still populate if the chart renders.

Rate-limit: cap at ~1 company per 10 seconds when batching.

**Comprehensive-population rule**: when enriching a row (fresh or re-enrichment), populate EVERY field the ContactOut response supports, including identity fields (Founded Year, HQ, Company Type, Overview, URL, LinkedIn URL). Only skip a field if it's already populated in Notion — never overwrite. Leaving known gaps (e.g., Amazon's Founded Year blank when ContactOut has it) is a bug.

**3e. Merge Deal Digest mentions into Momentum (+ set relation)** — query the cached inverted index of Deal Digest bullets, then interleave mentions with funding rounds in the `Momentum` field. Full cache spec in `shared-references/deal-digest-cache.md`.

1. Read `/Users/tomseo/.claude/skills/shared-references/deal-digest-cache.json`.
2. Check staleness: rebuild the cache if `built_at` is >7 days old OR if the count of Deal Digest pages currently in the Notes DB exceeds `len(digest_pages)` in the cache. Follow the rebuild procedure in the cache spec. Do NOT skip this step — a stale cache silently drops recent mentions.
3. Normalize the company name (lowercase, strip whitespace, strip trailing periods, strip parenthetical suffixes). Look up in `company_mentions`. If miss, try substring fallback (4+ char overlap).
4. Write `📰 Deal Digest Mentions` relation as JSON array of the distinct `digest_url` values from all matched mentions, sorted by `digest_date` ascending.
5. **Build the merged `Momentum` timeline** (see the Momentum field-table entry for the full format spec — bullet list, newest first, `MON YYYY` dates, lowercase `m`/`b`/`k`):
   - Collect all funding rounds from ContactOut `funding.rounds[]`.
   - Collect all Deal Digest mentions for this company from the cache.
   - **Decompose each digest bullet** into its attributes: identify any post-money valuation (attach to the corresponding round's entry — match by round series/date from bullet context), and any ARR / growth / traction number (emit as its own standalone entry).
   - **Merge rule:** a traction number captured **within ±1 month** of a round's date can be folded into that round's line if it reads as "ARR at close." Otherwise keep traction as a separate `(deal digest)` entry with the digest's own date. Default to separate — folding is the exception.
   - Sort all entries (rounds + standalone traction) by date, **newest first**.
   - Render as a bullet list with `- ` prefix per line, no bold, no blank lines between.
6. Dates: `Mon YYYY` format, 3-letter title-case month (`Apr 2025`, `Sep 2025`).
7. If `funding.rounds[]` is empty AND no digest mentions → leave `Momentum` blank (don't write `[]` or `"[]"`).
8. **Finalize `Revenue`** — hierarchy: if any Deal Digest mention for this company carries an ARR number, pick the MOST RECENT one and overwrite Step 3b's ContactOut value. Format: `$Xm ARR (Mon 'YY Deal Digest)`. If no digest ARR mention, keep Step 3b's `$Xm (ContactOut Est.)` format. If neither has an ARR, leave blank. Digest ARR beats ContactOut ARR even if ContactOut's number is bigger — Tom trusts the digest more.

**Skip conditions:**
- Category is `Fund 💸` or `School 🎓` → leave both relation and Momentum blank (Deal Digest tracks operating companies only).

**3f. Skip re-enrichment** — if Step 3a found a row with `Last Enriched` already populated, SKIP Steps 3b/3c entirely. Just use the existing page URL. This is the credit-saving rule. Note: re-running HC Growth (3d) and Deal Digest (3e) on refresh is fine — they don't consume ContactOut credits.

**3g. Collect the page URL** returned by search or create. This URL is what gets passed into the relation fields on the -1 Scanner entry. All relation URLs must be full `https://www.notion.so/` URLs, not bare UUIDs.

For **Work History**, resolve every distinct company from the person's experience array **including the primary company as the first entry** (so its logo appears at the top of the Work History relation chips). Deduplicate by domain before searching/creating.

### Step 4: Create or Update the -1 Scanner Entry

**Create path** (URL-driven, manual invocation from user): use `notion-create-pages` with parent `data_source_id: "32c00bef-f4aa-80a5-923b-000b83921fa3"`.

**Update path** (pipeline-agent Task 6 invokes this skill with an existing page ID where `Experience Summary` is empty): use `notion-update-page` with the same field population logic. Do NOT create a duplicate.

Set the page **icon** to `profile.profile_picture_url` from the ContactOut response if available. If the API returns no profile picture URL, leave the icon unset. Note: ContactOut profile image URLs may not always render correctly in Notion — this is a known CDN limitation, but attempt it anyway.

| -1 Scanner Field | How to Populate |
|-------------------|----------------|
| **Name** | `profile.full_name` from the API |
| **LI** | The LinkedIn URL provided as input |
| **Company** | The primary company's Notion page URL as a JSON string (single relation) |
| **Role** | The person's title at their primary company |
| **City** | Map `profile.location` to the metro area. Berkeley/Oakland/Palo Alto/Mountain View/Sunnyvale → "San Francisco". Brooklyn/Queens/Manhattan/Bronx → "New York". Arlington VA/Bethesda MD → "Washington DC". Use the core metro city name, not suburbs or neighborhoods. City is a **single select** field — if the metro area doesn't exist as an option yet, create a new one (Notion handles this automatically when you pass a new value). |
| **Email** | Prefer `profile.personal_email` (first entry). If no personal email is available, fall back to `profile.work_email` (first entry). If neither exists, omit. |
| **School(s)** | Relation to the Companies database. For each school in `profile.education`, search the Companies DB (same as company resolution — search first, create if not found). When creating a new school, set Category to "School 🎓" and URL to the school's website. Pass the collected page URLs as a JSON array. |
| **Field(s) of Study** | Comma-separated, each with degree in parens. Normalize degree abbreviations to broad categories: B.S./B.A./BA/BS → "Bachelor's", M.S./M.A./MA/MS → "Master's", Ph.D./PhD → "PhD", M.B.A./MBA → "MBA", J.D./JD → "JD". Example: "Economics (Bachelor's), Computer Science (Master's)". Include whatever is available — if degree is missing, just the field; if field is missing, just the degree. |
| **LI Profile Summary** | `profile.summary` from the API. If empty/null, omit the field entirely. |
| **Function** | Single select inferred from the person's current title at their primary company. Map using these rules (check in order, first match wins): **Founder** — Founder, Co-Founder, CEO (only when at their own company or an early-stage startup they started). **GTM** — Sales, Business Development, Marketing, Growth, Partnerships, Revenue, Account Executive, SDR/BDR, Customer Success, Demand Gen, "Founding Growth". **Engineering** — Software Engineer, SWE, Developer, SRE, DevOps, Infrastructure, Backend, Frontend, Full Stack, ML Engineer, Data Engineer, CTO, "Founding Engineer". **Product** — Product Manager, PM, Product Lead, VP Product, CPO. **Design** — UX, UI, Product Designer, Visual Designer, Brand, Creative Director. **Talent** — Recruiter, People Ops, HR, Talent Acquisition, Head of People. **Finance** — CFO, Controller, FP&A, Accounting, Treasury. **Operations** — COO, Chief of Staff, Business Operations, Strategy & Ops. **Data** — Data Scientist, Data Analyst, Analytics, BI. **Legal** — General Counsel, Legal, Compliance. If ambiguous or none of the above match, omit — never guess. |
| **Online Presence** | Files property holding all discovered URLs — populated by Step 4.5 (online-presence research). See dedicated section below. |
| **Work History** | JSON array of Notion page URLs for every company in the person's experience history, **including the primary company as the first entry** (so its logo appears at the top). The primary company also appears in the Company field — that's intentional. |
| **Experience Summary** | Entries separated by a **blank line** (double line break), no bullets, with bold company-name prefix through the colon. **Consolidate multiple roles at the same company into a single entry** — if someone held two positions at "Corgi" and "Corgi (YC S24)", group them: `**Corgi:** Chief of Staff (10/2025–Present); Founding Growth (12/2024–10/2025): AI insurance`. Match company names fuzzy (ignore parenthetical suffixes like "(YC S24)"). For distinct companies, format as: `**Company Name:** Title (Start–End): summary text`. Bold runs from company name through the first colon (after the company name). Include all experience entries regardless of whether they have a summary — if an entry has no summary, just show the company, title, and dates. |
| **Last Enriched** | Today's date (ISO-8601). Set on the -1 Scanner row when enrichment completes — this is the person-level marker distinct from the Companies DB `Last Enriched` on each employer. Always populate on create AND on any re-enrichment pass. |

### Step 4.5: Research Online Presence

Populate the `Online Presence` Files property with every discoverable URL for this person. Two phases:

**Phase A — ContactOut harvest.** From the person enrichment response, extract:
- `profile.github[]` → GitHub URLs
- `profile.twitter[]` → Twitter/X URLs
- Any URL found in `profile.summary` matching a personal-domain pattern (`*.com`, `*.io`, `*.dev`, `*.xyz`, `*.me`) that isn't a company domain

**Phase B — Web search fallback.** For each platform below NOT already found via ContactOut, run the full 4-round search pass per memory `feedback_online_presence_look_harder.md`. Time-box: 5–8 minutes across all searches. The 4 rounds are mandatory by default — do NOT stop after Round 1 unless you've validated every candidate account.

- **Round 1 (baseline)**: site-specific queries (`"{Name}" site:x.com`, etc.)
- **Round 2 (disambiguate via bio facts)**: combine name with unique identifiers (employer + school + city + specific product). Cross-check every Round 1 candidate against known bio facts — discard any account whose location/employer/year doesn't match.
- **Round 3 (handle variants)**: try non-obvious patterns (middle initial, underscore, number suffix, locale suffix).
- **Round 4 (verification)**: for any candidate, WebFetch the bio and require at least 2 fact matches before keeping.

Absence of social accounts IS signal — don't fabricate accounts, but don't stop at Round 1 misses either.

Starting source list (non-exhaustive — if a search result reveals a platform not listed here, ALSO capture it):

| Source | Search pattern |
|---|---|
| GitHub | `"{Name}" site:github.com` |
| Twitter/X | `"{Name}" site:x.com OR site:twitter.com` |
| Personal site | `"{Name}" "{Primary Company}"` (scan top results for personal-domain pattern) |
| Substack | `"{Name}" site:substack.com` |
| Medium | `"{Name}" site:medium.com` |
| Hacker News | `"{Name}" site:news.ycombinator.com/user` |
| Goodreads | `"{Name}" site:goodreads.com` |
| Product Hunt | `"{Name}" site:producthunt.com` |
| Google Scholar | `"{Name}" site:scholar.google.com` |
| Farcaster/Warpcast | `"{Name}" site:warpcast.com OR site:farcaster.xyz` |
| AngelList | `"{Name}" site:angel.co OR site:wellfound.com` |
| Podcast appearances | `"{Name}" "{Primary Company}" podcast` |

**Extensible rule**: the list above is a starting point. If a web search surfaces a platform not listed (e.g., Strava, Chess.com, Manifold Markets, Polymarket, Patreon, Discord server, Beehiiv newsletter, Instagram if Tom-relevant, etc.), capture it too. Don't filter by the starter list.

**Validation heuristic**: for each candidate URL, verify name match in the page title or bio. WebFetch the URL briefly if ambiguous. Filter out common-name collisions (multiple "John Smiths" on GitHub, etc.) by cross-referencing their `primary company` or domain.

**Write to Notion via osascript + Chrome (DEFAULT path, not fallback).** The Notion public API / MCP cannot write external URLs to Files properties. The canonical path is Notion's internal `/api/v3/saveTransactions` endpoint, driven by osascript against the active Chrome tab on Tom's Mac.

Read the shared reference at `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md` for the generalized `addLinkToFilesProperty` function, the osascript bridge pattern, and known property-key mappings (Online Presence key on -1 Scanner = `n]Rk`). Use with these parameters:

- `pageId`: the -1 Scanner page ID for the target row
- `propertyName`: `"Online Presence"`
- `url`: the discovered URL
- `displayName`: the **platform label formatted per the convention below**

**Display name convention** (for consistency across all -1 Scanner rows):
- **Social/account URLs with a handle**: `"[Platform] (@handle)"` — e.g., `"X (@gregreiner)"`, `"Instagram (@greg)"`, `"Farcaster (@greg.eth)"`, `"GitHub (@greg)"`
- **Personal sites / articles / other**: descriptive phrase — e.g., `"Personal site"`, `"Hamden Hall alumni feature"`, `"Acquired podcast guest"`
- **Writing platforms with a custom subdomain/slug**: `"[Platform]: [Publication/Handle]"` — e.g., `"Substack: greg.substack.com"`, `"Medium: @greg"`

**Call once per discovered URL** — the function is self-contained read-modify-write and correctly appends sequential calls without clobbering.

**Chrome availability**: Chrome is assumed running on Tom's Mac. If it isn't, launch it (`open -a "Google Chrome"`) before the osascript calls. Do NOT skip this step because "Chrome MCP is unavailable" — osascript is the bridge.

**True fallback (only if Chrome itself cannot run)**: write the list as a markdown `## Online Presence` section in the page body via `notion-update-page`. Flag this in the Step 6 report so Tom knows the canonical location moved. Re-run the osascript path next time Chrome is available.

**Empty**: if ContactOut + web search both return nothing, leave Online Presence empty. Don't fabricate.

### Step 5: Apply the Rubric and Write the Evaluation

Read the framework + rubric from co-located references:
- `CALIBRATION.md` — 6-founder calibration corpus (per-signal scoring evidence)
- `ONLINE_SOURCES.md` — Phase 2 online research taxonomy
- `../founder-outreach/FRAMEWORK_PRD.md` §6 — current rubric, anchors, thresholds (canonical)

Apply the framework in two phases:

**Phase 1 — derive HIGH-fidelity signals from already-ingested structured data:**
- **Non-Linearity**: count function/discipline crossings across `experience[].job_function` + title transitions.
- **Earned Reps**: cross-reference `experience[]` tenure × Companies DB Hypergrowth Windows (round-cadence-derived) AND Headcount/HC Commentary (Sales Nav scrape) — score 9/10 if hypergrowth-during-tenure with evolution; 0 if drift.
- **Range**: triple intersection of `job_function` × `industry` across employers (Technical / Commercial / Domain).

**Phase 2 — narrative research for LOW–MED fidelity signals (only if Phase 1 doesn't already disqualify):**
- **Intellectual Rigor**: WebFetch publications, blog posts, podcast transcripts surfaced via Online Presence (per `ONLINE_SOURCES.md`). Read for non-performative tone, self-correction, hedged register. If absent, flag as "unobservable from public" — do NOT score 0.
- **Anticipation**: WebSearch on `projects[]` / `headline` topics with date filters — was the bet pre-consensus at the time? Cross-check internal employer projects too.
- **Intentionality**: LLM read of `experience[].summary` for demotion / anti-accelerator / patient-tenure markers; "On leaving X" essays; podcast career-arc framing.

**Compute the verdict (per FRAMEWORK_PRD.md §6):**
- `Eval Score (Spike) = MAX(signal scores)`. Pure spike-based MAX. **No Intentionality gate** (v0.4 removed it — Intentionality is informational, not a veto).
- `Claude Rec` per §6.5 thresholds:
  - peak ≥ 7 → `Reach Out ✅`
  - peak 4–6 → `Second Look 🤔`
  - peak 0–3 → `Pass ❌`
- `Signals` (multi-select): tag every signal that scored ≥ 5.
- `Working Description` (2-3 sentence TL;DR, anchored on the peak signal).
- `Eval Breakdown` (per-signal rationale with evidence URLs from Phase 2 research).
- `Eval Summary` (rationale paragraph; bold the peak-signal sentence).

Write all six fields back to the -1 Scanner row via `notion-update-page`. Do not touch Status here.

### Step 6: (Manual mode only) Chain to Drafting if Tom-flippable to ✅

The -1 Scanner's `Status` field workflow:
- `Pending Enrichment` → initial state; picked up by pipeline-agent Task 6
- `Draft Ready` → after `founder-outreach` produces a Gmail draft (set BY founder-outreach, not here)
- `Reached Out` → Tom sent the outreach (detected by pipeline-agent Task 7)
- `Passed` → Tom decided not to send (manual flip)

After Step 5 writes the verdict, the candidate is **fully evaluated** but Status remains `Pending Enrichment` — Tom decides whether to draft.

**Manual mode** (called with a raw LI URL from a user prompt): if `Claude Rec = Reach Out ✅`, surface the recommendation but do NOT auto-invoke `founder-outreach`. Tom reviews the score in Notion first; if he agrees, he triggers `founder-outreach` separately or asks "draft for [name]". This avoids drafting emails for candidates Tom would override to Pass.

**Task 6 mode** (called by pipeline-agent with an existing page ID): same — return after Step 5 with the verdict written. Tom triages from the scheduled summary.

The split: `neg1-enricher` ends at "verdict written, row ready for review." `founder-outreach` begins at "Tom said yes, write the email."

### Step 7: Report Back

After creating (or updating) entries, provide a brief confirmation per person: name, primary company, role, **peak signal + score + Claude Rec**, and a link to the Notion page. Surface a one-line excerpt of the Eval Summary so Tom can quickly scan whether the verdict reads right.

For batches, present results as a summary table sorted by Eval Score descending.

---

## `--score-only` Mode (re-scoring without re-enriching)

Per FRAMEWORK_PRD.md §6.7 (re-scoring without re-enriching). Used when the rubric changes and Tom wants to re-score existing rows against the new rubric without burning ContactOut credits re-pulling data that hasn't changed.

**Trigger**: user passes `--score-only` flag, OR uses phrases like "rescore [name]", "re-score [name]", "score-only this row", "rescore the calibration corpus".

**Behavior**:
- **Skip Steps 1–4.5 entirely** — no ContactOut calls, no online presence re-research, no Companies DB writes.
- **Run Step 5 only**, against whatever data is already on the -1 Scanner row (Experience Summary, LI Profile Summary, Online Presence files, Companies relations + their Headcount/Momentum/HC Commentary).
- **Overwrite** Eval Score, Eval Breakdown, Signals, Working Description, Claude Rec, Eval Summary with the new verdict.
- **Preserve Status** — a row that's already `Reached Out` stays `Reached Out` regardless of the new score. We're updating the scoring artifact, not the workflow state.
- **No Step 6 chain** to founder-outreach in score-only mode (it's a re-evaluation, not a fresh outreach decision).

**Targeting**:
- Single row: pass a -1 Scanner page URL or person name.
- The 6-founder calibration corpus (per FRAMEWORK_PRD.md Future State item 14, quarterly drift check): pass `--score-only --calibration-corpus` (resolves via the names in `CALIBRATION.md`).
- Bulk: pass `--score-only --where "Eval Score is null"` or any Notion filter. Use sparingly — re-scoring 100+ rows takes time even without ContactOut calls.

**Snapshot before overwrite**: write before-state to `/tmp/score_only_snapshot_{timestamp}.jsonl` (one JSON per row: page_id, name, before_eval_score, before_eval_breakdown, before_eval_summary, before_claude_rec). Lets Tom diff old vs new verdicts and revert if a rubric change misfires.

**Reporting**: after a `--score-only` batch, surface a diff table — for each row, show `before → after` on Eval Score and Claude Rec. Highlight rows where the verdict changed (✅ → 🤔, etc.).

## Important Rules

- **Speed over confirmation** — do not ask for confirmation before creating entries. Just run the enrichment and create. Only ask if something is genuinely ambiguous.
- **Company Overview is a rollup** on the -1 Scanner table — it pulls from the Companies database automatically. Populate the overview on the Companies entry, not on the -1 Scanner entry.
- **Deduplication** — before creating a -1 Scanner entry, search the data source by name to check if the person already exists. If they do, mention it and skip rather than creating a duplicate (unless invoked in update-mode by pipeline-agent Task 6, in which case update in place).
- **Empty fields** — if the API returns empty/null for a field, leave it out of the Notion call entirely. Never fabricate or hallucinate data to fill gaps.
- **profile_only must be false** — because we need email addresses for the Email field, set `profile_only: false` when calling `contactout_enrich_linkedin_profile`. This consumes email credits but is necessary for this workflow.
- **Never re-enrich companies with `Last Enriched` populated** — credit-saving rule.

## Example

**User:** `-1 scan https://www.linkedin.com/in/janedoe/`

**Claude:**
1. Calls `contactout_enrich_linkedin_profile` with `profile_only: false` → Jane Doe, VP of Engineering at Stripe, previously at Google and Meta. Personal email: jane@gmail.com. Education: Stanford (B.S. Computer Science), MIT (M.S. AI).
2. Determines primary: VP of Engineering at Stripe.
3. Dedup-searches Companies DB by domain → Stripe found (Last Enriched → skip re-enrich), Google found (skip), Meta found but Last Enriched empty → backfills Meta with the 12 enrichment fields.
4. Creates -1 Scanner entry with the full field set (no scoring fields — those are populated by founder-outreach).
5. Returns: "Added Jane Doe (VP of Engineering, Stripe) to -1 Scanner. [link]"
