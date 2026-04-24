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
  URLs is to run this skill. Distinct from "add to contacts" (People DB) and "add to crm"
  (Opportunities pipeline). On MANUAL invocation this skill auto-chains into
  `founder-outreach` after scoring, regardless of Claude Rec verdict — Tom wants the draft
  sitting ready for review alongside the score. Batch invocation by pipeline-agent Task 6
  does NOT chain (that's scheduled bulk enrichment — drafting stays manual). Part of
  pipeline management.
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

### Step 3: Resolve Companies in Notion (via `add-to-companies`)

The **Company** field (primary employer, limit 1), **Work History** field (all employers), and **School(s)** field (all schools from education) are all **relations** to the Companies database.

**This skill does NOT reimplement the Companies enrichment spec.** For every distinct company and school in the person's experience + education arrays, invoke the `add-to-companies` skill as a subroutine. That skill handles dedup, enrichment, backfill, Sales Nav scrape, Deal Digest merge, icon, and Last Enriched per the canonical spec at:

**`/Users/tomseo/.claude/skills/shared-references/companies-enrichment-spec.md`**

Collect the page URL returned by each `add-to-companies` invocation — these are what populate the relation fields on the -1 Scanner row.

**For Work History**, resolve every distinct company from the person's experience array **including the primary company as the first entry** (so its logo appears at the top of the Work History relation chips). Deduplicate by domain before invoking `add-to-companies`.

**For School(s)**, pass each education entry to `add-to-companies` — the skill will assign Category = `School 🎓` and populate Legacy core fields only per the spec.

**Skip study abroad programs entirely** (see filtering rules in Step 2) — do not invoke `add-to-companies` for them.

**Rate-limit:** cap at ~1 company per 10 seconds when batching.

**All relation URLs must be full `https://www.notion.so/` URLs, not bare UUIDs.**

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
- `FOUNDER_CALIBRATION.md` — 6-founder calibration corpus (per-signal scoring evidence)
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

### Step 6: Chain to Drafting (manual invocation only)

The -1 Scanner's `Status` field workflow:
- `Pending Enrichment` → initial state; picked up by pipeline-agent Task 6
- `Draft Ready` → after `founder-outreach` produces a Gmail draft (set BY founder-outreach)
- `Reached Out` → Tom sent the outreach (detected by pipeline-agent Task 7)
- `Passed` → Tom decided not to send (manual flip)

**Manual mode** (called with a raw LI URL from a user prompt, or any direct trigger phrase): after Step 5 writes the verdict, **auto-invoke `founder-outreach`** to produce the Gmail draft. Run regardless of Claude Rec — even Pass verdicts get a draft so Tom can see what the outreach would have looked like and make the final send/discard call himself. Do NOT pause to ask for confirmation between the two skills. Single run, single report back.

**Exception**: if the user explicitly says "enrich only" / "just score, don't draft" / "no outreach" / similar, skip the chain and stop after Step 5.

**Task 6 mode** (called by pipeline-agent with an existing page ID, scheduled bulk enrichment): do NOT chain. Return after Step 5 with the verdict written. Drafting from the scheduled summary stays manual — Tom triages which rows warrant an outreach draft.

**`--score-only` mode**: does NOT chain (it's a re-evaluation, not a fresh outreach decision).

The split: `neg1-enricher` produces the scored row. On manual invocation it also triggers `founder-outreach` as a terminal step. `founder-outreach` owns the Gmail draft itself.

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
- The 6-founder calibration corpus (per FRAMEWORK_PRD.md Future State item 14, quarterly drift check): pass `--score-only --calibration-corpus` (resolves via the names in `FOUNDER_CALIBRATION.md`).
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
4. Creates -1 Scanner entry with the full field set.
5. Runs Step 5 scoring → Eval Score 8 (Earned Reps peak), Claude Rec = Reach Out ✅.
6. Auto-chains into `founder-outreach` → Gmail draft created, Status = Draft Ready.
7. Returns: "Added Jane Doe (VP of Engineering, Stripe) to -1 Scanner. Eval Score 8 / Reach Out ✅. Gmail draft ready. [link]"

## Behavior Rules

### Look harder by default — run the full online-presence search pass

When researching a candidate's online presence (Step 4.5 here, or any similar people-search task), **always run the full deeper search pass by default** — don't stop after the first round of obvious queries.

**Why:** On 2026-04-19 during Greg Reiner's enrichment, first pass returned `@gregreiner` on X and IG — both the wrong person. Tom had to explicitly ask "can you look harder?" before a second pass ran. The harder pass should be automatic, not on request.

**Default flow — all four rounds run automatically:**

**Round 1: baseline**
- `"{Name}" site:github.com / x.com / substack.com / producthunt.com / ...`
- Accepts obvious handle matches.

**Round 2: disambiguate via bio facts**
- Combine name with unique biographical disambiguators: current employer + school + city + specific product. E.g., `"Greg Reiner" Meta Penn` or `"Greg Reiner" "Facebook Live" OR "Reels"`.
- Look up referenced bio articles (alumni profiles, press quotes, podcast appearances) and mine for handle hints OR for colleagues whose socials might tag the target.
- For each Round 1 candidate, cross-check against known bio facts (location, employer, hometown, classmates). A `@gregreiner` from DC when target is NYC → not him. Discard and keep looking.

**Round 3: handle variants**
- Try non-obvious patterns: middle initial (`gregjreiner`), underscore (`greg_reiner`), number suffix (`gregreiner89` — birth year / class year), locale (`gregreinernyc`), professional suffix (`gregreinerpm`).
- Search first initial + last name, full first name + last name.

**Round 4: verification**
- For any candidate account, WebFetch the bio/description and verify at least 2 facts match (location + employer, or employer + year).
- If no second fact confirms, treat as unvalidated and exclude.

**When to stop:**
- All 4 rounds complete AND no validated account → legitimately "no public presence." Report as such — absence IS signal for Intellectual Rigor scoring.
- A validated account is found → capture with `[Platform] (@handle)` format.

**Time budget:** 5–8 minutes for the full pass. Longer than Round 1 alone, but prevents back-and-forth and ensures Online Presence is accurate on first run.

Applies across skills: this skill's Step 4.5, feedback-outreach-drafter (backchannel contact discovery), and any future people-research task.
