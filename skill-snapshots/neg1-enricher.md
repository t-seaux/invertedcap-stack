---
name: neg1-enricher
description: >-
  Pipeline-management primitive. Enriches LinkedIn profiles via the full ContactOut API and
  populates the -1 Scanner database in Notion with per-person rows and Companies DB relations.
  One-shot company enrichment (Company Search per distinct employer, skipped if Last Enriched
  is already populated — credit-saving). Manual trigger only. When the user feeds one or more
  LinkedIn URLs, this skill runs end-to-end: creates the -1 Scanner row AND enriches it in
  one invocation. Scheduled enrichment of unenriched rows is handled by pipeline-agent's
  Task 6, which invokes THIS skill as its primitive. Trigger phrases: "-1 scan [URL]", "-1
  scanner", "add to scanner", "add to -1 scanner", "add to sourcing", "source these", "scan
  these profiles", "enrich these LI URLs", "add these to -1", "scanner", "enrich [LI URL]",
  or any variant involving LinkedIn URLs paired with sourcing/enrichment intent. Also trigger
  when the user pastes a batch of LinkedIn URLs without further context — the default action
  for a bare list of LinkedIn URLs is to run this skill. Distinct from "add to contacts"
  (People DB), "add to crm" (Opportunities pipeline), and "founder-outreach" (scoring +
  drafting — invokes this skill when enrichment is needed first). Part of pipeline
  management.
---

# -1 Enricher

Enrichment primitive for pre-founder candidates. One job: takes a LinkedIn URL, creates a fully-enriched row in the -1 Scanner database with proper Companies relations and one-shot company enrichment. Does not score. Does not draft. Those live in `founder-outreach`.

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
- **Primary key: Domain.** Search by `Domain` field first (exact match on domain string from `experience[].domain`). Domain match is authoritative — different names with same domain are the same company.
- **Fallback: Name fuzzy match.** If no domain hit (or no domain available), fall back to name matching. Be forgiving ("Google DeepMind" matches "DeepMind", "Meta" matches "Facebook/Meta").
- If found, capture the row's `Last Enriched` value — determines whether Step 3c backfill is needed.

**3b. Create if not found** — use `notion-create-pages` with parent `data_source_id: "7d50b286-c431-49f5-b96a-6a6390691309"`.

For **Company**-category entries, call `contactout_enrich_company` to pull the full Company Search response (falling back to `contactout_enrich_domain` if it misses). Populate the fields below. For **School**-category entries, only populate the Legacy core fields (use `contactout_enrich_domain` as before) — skip all Enrichment fields. For **Fund**-category entries, populate both tiers.

| Companies Field | How to Populate |
|----------------|----------------|
| **Name** | Company name from ContactOut data |
| **Category** | "School 🎓" if the entity is a university, college, or educational institution. "Fund 💸" if the company's industry involves Venture Capital, Private Equity, Investment Management, Hedge Funds, Asset Management, or if the entity is clearly an investment fund. Otherwise "Company 🛠️". |
| **Company Overview** | **Always populate for every new entity.** For the primary company, use `company.overview`. For work history companies and schools, use `description` from `contactout_enrich_domain` (or `overview` from Company Search). If nothing returned, leave blank — never fabricate. |
| **userDefined:URL** | Company website. For primary company: `company.website`. For work history: `https://{domain}` from `experience[].domain`. Omit if unavailable. |
| **Total Funding** | Rich text in millions — legacy display. For primary: `company.funding.total_funding_usd`. For work history: from Company Search or `contactout_enrich_domain` fallback. Format: >= $1b → "$X.Xb", >= $1m → "$X.Xm", >= $1k → "$X.Xk", else "$X". Public companies with null funding → "Public". Schools → "N/A". |
| **LinkedIn URL** | Company's LinkedIn URL from Company Search `linkedin_url`, or from the `experience[].company.linkedin_url` on person enrichment. |
| **Domain** | Canonical dedup key — the domain string (e.g., `https://stripe.com`). For primary: `company.domain`. For work history: `https://{experience[].domain}`. |
| **Industry** | From `industry` field on Company Search or `experience[].company.industry`. |
| **Employee Count** | From `size` or `employees` (numeric). Integer. |
| **Founded Year** | From `founded_at` (year only, integer). |
| **Company Type** | Map ContactOut `type`: "Privately Held" → "Private"; public companies → "Public"; acquired entities → "Acquired"; nonprofits → "Nonprofit". |
| **HQ Location** | From `headquarter` or `country` — concise format ("San Francisco, CA" or "London, UK"). |
| **Revenue Range** | From `revenue` (range string as returned). |
| **Funding Status** | Map `funding.funding_status` to the select option: Pre-Seed, Seed, Series A, Series B, Series C, Series D+ (Series D and later), Public, Acquired, Bootstrapped. |
| **Funding Rounds JSON** | Serialize `funding.rounds[]` as compact JSON. Each round: `{date, type, amount_usd, lead_investors, investor_names}`. Example: `[{"date":"2018-03","type":"Series A","amount_usd":12000000,"lead_investors":["a16z"]},...]`. |
| **Hypergrowth Windows** | Computed — see Step 3d. |
| **Last Enriched** | Today's date (ISO-8601). This is the credit-saving marker — its presence means "don't re-enrich." |

Set the page **icon** to the entity's logo URL from ContactOut. Primary company: `company.logo_url`. Work history: `experience[].logo_url` or Company Search `logo_url`. Schools: `contactout_enrich_domain` → `logo_url`. Pass as `icon` on `notion-create-pages`.

**3c. Backfill existing rows** — if Step 3a found a row but `Last Enriched` is empty:
- This row was created before the enrichment architecture existed. Backfill once.
- Call `contactout_enrich_company` (or fallback to domain) and populate only the **new** fields: LinkedIn URL, Domain, Industry, Employee Count, Founded Year, Company Type, HQ Location, Revenue Range, Funding Status, Funding Rounds JSON, Hypergrowth Windows, Last Enriched.
- **Do NOT overwrite** existing Name, Category, Company Overview, URL, Total Funding, icon — Tom may have curated these.
- Skip backfill for School-category rows.

**3d. Compute Hypergrowth Windows** — from the Funding Rounds JSON:
- Sort rounds by `date` ascending.
- Identify consecutive round pairs where `date[i+1] - date[i] < 18 months`.
- Collapse overlapping pairs into a continuous window. If rounds in 2017, 2018, and 2019 all fall within 18-month gaps, that's the window `2017-2019`.
- Store as comma-separated year ranges — e.g., `"2017-2019, 2021-2023"`. Empty string if no windows qualify.
- A single round, or rounds with > 18 month gaps, yields no window.

**3e. Skip re-enrichment** — if Step 3a found a row with `Last Enriched` already populated, SKIP Steps 3b/3c entirely. Just use the existing page URL. This is the credit-saving rule.

**3f. Collect the page URL** returned by search or create. This URL is what gets passed into the relation fields on the -1 Scanner entry. All relation URLs must be full `https://www.notion.so/` URLs, not bare UUIDs.

For **Work History**, resolve every distinct company from the person's experience array **including the primary company as the first entry** (so its logo appears at the top of the Work History relation chips). Deduplicate by domain before searching/creating.

### Step 4: Create or Update the -1 Scanner Entry

**Create path** (URL-driven, manual invocation from user): use `notion-create-pages` with parent `data_source_id: "32c00bef-f4aa-80a5-923b-000b83921fa3"`.

**Update path** (pipeline-agent Task 6 invokes this skill with an existing page ID where `Profile Experience Summary` is empty): use `notion-update-page` with the same field population logic. Do NOT create a duplicate.

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
| **Profile Experience Summary** | Bulleted list with bold company names. **Consolidate multiple roles at the same company into a single bullet** — if someone held two positions at "Corgi" and "Corgi (YC S24)", group them: "• **Corgi** — Chief of Staff (10/2025–Present); Founding Growth (12/2024–10/2025): AI insurance". Match company names fuzzy (ignore parenthetical suffixes like "(YC S24)"). For distinct companies, format as: "• **Company Name** — Title (Start–End): summary text". Include all experience entries regardless of whether they have a summary — if an entry has no summary, just show the company, title, and dates. |
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

### Step 5: Chain to Scoring + Drafting

The -1 Scanner's `Status` field workflow is:
- `Pending Enrichment` → initial state; picked up by pipeline-agent Task 6
- `Draft Ready` → after the full pipeline completes (enrich + score + draft)
- `Reached Out` → Tom sent the outreach (detected by pipeline-agent Task 7)
- `Passed` → Tom decided not to send (manual flip)

When invoked manually by the user (not by Task 6), this skill runs the **full pipeline end-to-end** — enrichment alone is incomplete. After Step 4 completes, invoke `founder-outreach` to run Score + Draft for the newly-enriched row. `founder-outreach` sets `Status = Draft Ready` at the end.

When invoked by pipeline-agent Task 6, Task 6 handles the founder-outreach invocation after this skill returns — do NOT invoke founder-outreach from within this skill in that path.

Detection: if called with a raw LI URL from a user prompt → manual mode → chain to founder-outreach. If called by pipeline-agent Task 6 with an existing page ID → Task 6 mode → return without chaining.

### Step 6: Report Back

After creating (or updating) entries, provide a brief confirmation per person: name, primary company, role, Status, and a link to the Notion page. In manual mode, note that founder-outreach was chained and report its results as well (peak signal + Gmail draft URL).

For batches, present results as a summary table at the end.

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
