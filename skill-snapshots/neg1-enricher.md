---
name: neg1-enricher
description: >-
  Enrichment + evaluation primitive for pre-founder (-1) candidates — HEADLESS, upstream of Notion. Takes a
  LinkedIn URL, enriches via ContactOut + local company caches + online research, applies the founder-taste
  rubric (RUBRIC.md) + When gate, writes the verdict to the CANDIDATE STORE
  (~/.claude/scripts/decision-ledger/candidates.py — signals_line, eval_summary, eval_rationale, rec), and posts
  a candidate card to #neg1-sourcing when the verdict is Reach Out. NOTHING lands in Notion — that happens only
  when Tom replies draft (neg1-sourcing-listener creates the Opportunity). Job mode (claude-job-queue, args
  {mode: headless, li_url, source}) is the primary scheduled path — enqueued the moment a generator surfaces a
  name. Supports --score-only re-scoring against store rows. Trigger phrases: "-1 scan [URL]", "scan these
  profiles", "enrich these LI URLs", "score [name]", "rescore [name]", "run a founder eval on [name/this person]",
  "founder eval on [X]", or LinkedIn URL(s) with sourcing intent.
  The -1 Scanner Notion DB is DELETED (2026-07-16) — never query it; archive at
  ~/.claude/data/neg1_scanner_archive.json.
---

# -1 Enricher

Enrichment + evaluation primitive for pre-founder candidates. Takes a LinkedIn URL, produces a fully-enriched and **fully-evaluated** -1 Scanner row that Tom can act on. Per Tom's mental model: enrichment is "gather info/context, then apply my frameworks and intelligence to further enrich" — scoring IS a form of enrichment (it enriches the row with a rubric-derived verdict). Drafting the cold email lives in `founder-outreach` and only runs *after* Tom flips Claude Rec to ✅.

## v2 HEADLESS FLOW (2026-07-16 — CURRENT DEFAULT)

**The -1 Scanner DB is retired for new candidates.** Evaluation lives upstream in the candidate store (`candidates` table in `~/.claude/data/decision_ledger.db`, CLI: `python3 ~/.claude/scripts/decision-ledger/candidates.py`); Notion is touched only when Tom replies `draft` to a card — at which point an **Opportunity** is created in the Opportunities DB (see neg1-sourcing-listener). Everything below this section describes the LEGACY Notion-scanner path — still valid for existing scanner rows and only when Tom explicitly says "add to scanner".

**Step 0 (preflight) — existing-Opportunity check.** BEFORE enriching, search the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`) for an existing Opp on this person — query the candidate's name AND their LinkedIn URL (founder relations + title link both carry it). If an Opp already exists (any status), the person is already in the pipeline, so the enrichment must NOT stop at a candidate-store row + Slack card. Instead: run the full enrichment/scoring, then land the framework output as a **Notes-DB note tagged to that Opp** — a `-1 {Name}: Founder Eval` page in the Notes DB with the Opportunity relation set (same shape neg1-sourcing-listener produces at draft time; the Opp *body* stays clean — Tom's rule). Still upsert the store row for ledger continuity, but the tagged note is the deliverable Tom reads. Only when NO Opp exists do you follow the default card-only path below. (Tom, 2026-07-16: "when you neg1 enrich you should check to see if there's an opportunity entry. If so you should create a new note with the enrichment and tag it to the opp.")

**Eval-note format (canonical — Tom's 2026-07-16 spec; applies wherever the eval note is written, incl. neg1-sourcing-listener's draft-time note):**
- **Icon:** the `:claude-color:` Notion custom emoji (Claude generates the artifact, so it carries Claude's logo). Not a thematic emoji.
- **Title:** `-1 ([Name](LI URL)): Founder Eval` — hyperlinked name (same `-1 ([Name](LI))` shape as the Opp) plus the `: Founder Eval` suffix (Tom's confirmed format, 2026-07-16 screenshot).
- **Body = the "Tier A" mini-memo** (Tom-approved 2026-07-16; prototyped on Stephanie Wan). Notion-native only — no rendered images/PDFs. Blocks in this exact order:
  1. **Verdict callout** — `<callout icon="{verdict emoji}" color="{verdict color}">` where ✅→`green_bg`, 🤔→`yellow_bg`, ❌→`red_bg`. Inside: `**Claude Rec: {verdict}**` — verdict as plain text, NO emoji after it (the callout icon already carries the verdict emoji) and no parenthetical — + a 2–3 sentence summary + a re-surface/next-check line. The callout IS the summary — no separate `**Summary.**` paragraph.
  2. **Fact row + arc** — one line `**Seat:** … · **Home:** … · **Contact:** … · **Source:** …`, then `**Arc:** Stop → Stop → Stop` career one-liner.
  3. **`## Signals` table** (header-row) — columns `Signal | Score | In one line`. Score = plain `{n}/10` (NO dot scales — Tom found them weird). Peak signal bolded with a trailing `(Spike)` label (not a ★). Signal names spelled out in full (never NL/Ant/Rng). Numeric scores live ONLY in this table — the prose below must NOT repeat `({n}/10)` score-led paragraphs.
  4. **`## Founder Eval`** — six per-signal paragraphs, each led by the bolded signal name (`**Intellectual Rigor (Spike).**`, `**Intentionality.**`, …), ordered by score descending so the spike leads. NO What/How/Why lens rollup headings (Tom tried it and rejected it, 2026-07-16) and NO `({n}/10)` in the bold leads — scores stay in the table. Each paragraph: evidence + why-this-level in prose; the spike gets the most ink.
  5. **`**The detail that sticks.**`** — one memorable concrete fact (the line Tom would repeat to someone else).
  6. **`**Closest archetype.**`** — one line mapping to the §6 four-archetype taxonomy (or "too early to assign").
  7. **`## Gaps / what would change the read`** — 2–4 falsifiable bullets: evidence that would move the scores (a 0→1 seat, an opportunity cost walked past, fuller enrichment). This is evidence-framing, NOT a reach-out recommendation.
  8. **Footer provenance quote** — `> Pre-founder (-1) evaluation via the founder-taste rubric. Enriched from {sources}. {date}.`
- **No timing section.** Do NOT write a "When gate" / "W1/W2/W3" block AND do NOT write an "Is the timing right to reach out?" section — Tom makes the timing call himself (2026-07-16). The When gate still COMPUTES during scoring (§8.9) and can still downgrade the rec; it just isn't surfaced in the note at all.
- Keep the store's `eval_summary`/`eval_rationale` fields for machine use — the note renders the Tier A memo, not those fields verbatim.

Headless mode maps the legacy steps as follows:
- **Step 1 (ContactOut enrich + cache)** — unchanged, including the raw-payload cache write.
- **Step 2 (primary role)** — unchanged.
- **Step 3 (companies)** — do NOT create Notion Companies rows or relations. Read company context from the local caches instead: `~/.claude/scripts/company_cache.py` (headcount/momentum) and the Deal Digest cache (`shared-references/deal-digest-cache.md`) for revenue traction. Only candidates Tom drafts get Companies-DB treatment later (add-to-crm path).
- **Step 4 (write the row)** — replace the Notion write with a store upsert:
  ```bash
  python3 ~/.claude/scripts/decision-ledger/candidates.py upsert --json '{"li_url": "...", "name": "...", "email": "...", "city": "...", "spike_era_role": "...", "spike_era_company": "...", "current_role": "...", "current_company": "...", "education": "School (year)", "arc": ["School (yr)", "Stop (yrs)", ...], "presence": [{"label": "Blog", "url": "..."}], "path": "2nd degree via ...|cold", "type": "Cold 🧊|Warm ☀️", "source": "monday-sweep|manual|network-scan", "contactout_cache": "<cache file path>", "state": "surfaced"}'
  ```
- **Step 5 (rubric + When gate)** — unchanged logic (RUBRIC.md §5–§8 + §8.9), but outputs land in the store: `scores` (JSON dict), `signals_line` (compact format), `rec` (post-gate), `pre_gate_rec`, `when_gate` (fired rule + evidence or null), `eval_summary`, `eval_rationale` (full per-signal prose — this is the store's version of the old Eval Breakdown), `working_desc`, `gap` (sharpest Gaps clause).
- **Step 6 (chain to drafting)** — REMOVED in v2. Every headless run ENDS by posting the candidate card to `#neg1-sourcing` (pipeline-agent Task 6 step 2b format — read that block for the exact card anatomy) when the verdict is `Reach Out ✅`; save the posted ts via `set-state --card-ts`. `Second Look 🤔` and `Pass ❌` post no card (they're visible via `candidates.py list`). Tom decides from the card.

**Job mode (per-candidate, event-driven — the PRIMARY scheduled path):** invoked via claude-job-queue with args `{mode: "headless", li_url, source, name?}` (enqueued by `~/.claude/scripts/enqueue-neg1-enrich.sh` the moment a generator surfaces a name — there is NO batching delay). Run the full headless flow above for the single candidate, card it if ✅, exit. Idempotency: if the store row for `li_url` is already `state=surfaced` with a `card_ts` (or any later state), exit silently — the job is a retry.

---

**RETIRED 2026-07-16:** the -1 Scanner DB was deleted after full decommission — data archived at `~/.claude/data/neg1_scanner_archive.json` + migrated into the candidate store. Everything below referencing the Scanner is historical documentation only; do NOT query the DB.

## Notion Targets (RETIRED — historical)

- **-1 Scanner Database** — Data Source ID: `32c00bef-f4aa-80a5-923b-000b83921fa3`
- **Companies Database** — Data Source ID: `7d50b286-c431-49f5-b96a-6a6390691309`

## Workflow

### Step 1: Enrich Each Profile

For each LinkedIn URL, call `contactout_enrich_linkedin_profile` with `profile_only: false`. This returns the full profile payload — name, headline, location, seniority, company details, experience history, education, skills, languages, publications — along with email addresses (personal and work).

**Cache the raw payload.** Immediately after the call returns, BEFORE any field extraction, write the full response to `~/.claude/plugins/data/contactout-enrichment-inline/people/{vanity}.linkedin_profile.json` using the Write tool. The vanity slug is the LinkedIn URL's last path segment, lowercased, with any trailing slash and query string stripped. Overwrite if the file exists. This is the network-intel index's data source — never skip.

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

The **CurrentCo (CC)** field (primary employer, limit 1), **Work History** field (all employers), and **School(s)** field (all schools from education) are all **relations** to the Companies database.

**This skill does NOT reimplement the Companies enrichment spec.** For every distinct company and school in the person's experience + education arrays, invoke the `add-to-companies` skill as a subroutine. That skill handles dedup, enrichment, backfill, Sales Nav scrape, Deal Digest merge, icon, and Last Enriched per the canonical spec at:

**`/Users/tomseo/.claude/skills/shared-references/companies-enrichment-spec.md`**

Collect the page URL returned by each `add-to-companies` invocation — these are what populate the relation fields on the -1 Scanner row.

**For Work History**, resolve every distinct company from the person's experience array **including the primary company as the first entry** (so its logo appears at the top of the Work History relation chips). Deduplicate by domain before invoking `add-to-companies`.

**For School(s)**, pass each education entry to `add-to-companies` — the skill will assign Category = `School 🎓` and populate Legacy core fields only per the spec.

**Skip study abroad programs entirely** (see filtering rules in Step 2) — do not invoke `add-to-companies` for them.

**Rate-limit:** cap at ~1 company per 10 seconds when batching.

**All relation URLs must be full `https://www.notion.so/` URLs, not bare UUIDs.**

**Cache write-back:** after `add-to-companies` completes for the primary employer, sync that company into the local SQLite cache so future network-scan queries pick up the latest enriched data:

```bash
python3 /Users/tomseo/.claude/scripts/company_cache.py sync-one --domain {primary_domain}
```

### Step 4: Create or Update the -1 Scanner Entry

**Create path** (URL-driven, manual invocation from user): use `notion-create-pages` with parent `data_source_id: "32c00bef-f4aa-80a5-923b-000b83921fa3"`.

**Update path** (pipeline-agent Task 6 invokes this skill with an existing page ID where `Experience Summary` is empty): use `notion-update-page` with the same field population logic. Do NOT create a duplicate.

Set the page **icon** to `profile.profile_picture_url` from the ContactOut response if available. If the API returns no profile picture URL, leave the icon unset. Note: ContactOut profile image URLs may not always render correctly in Notion — this is a known CDN limitation, but attempt it anyway.

| -1 Scanner Field | How to Populate |
|-------------------|----------------|
| **Name** | `profile.full_name` from the API |
| **LI** | The LinkedIn URL provided as input |
| **CurrentCo (CC)** | The primary company's Notion page URL as a JSON string (single relation) |
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
| **Growth Tier / Timing Signal** | **Do not write these fields — they don't exist.** Growth and timing context goes in `Eval Summary` prose via CC Momentum + tenure overlap (e.g. "Joined Stripe in 2019 when ARR was ~$50M, Series E-era rapid scaling"). |
| **Type** | **Manual mode** (Tom-initiated via "add X to -1 enrich/engine" or raw LinkedIn URL): default to `Cold 🧊` unless Tom's phrasing indicates a prior relationship ("reconnect with X", "follow up with X from [prior company]") — then use `Warm ☀️`. **Task 6 mode** (pipeline-agent scheduled bulk enrichment with existing page IDs): do not write — neg1-sourcing set this at row creation. |
| **Last Enriched** | Today's date (ISO-8601). Set on the -1 Scanner row when enrichment completes — this is the person-level marker distinct from the Companies DB `Last Enriched` on each employer. Always populate on create AND on any re-enrichment pass. |

### Step 4.5: Research Online Presence

Populate the `Online Presence` Files property with every discoverable URL for this person. Two phases:

**Phase A — ContactOut harvest.** From the person enrichment response, extract:
- `profile.github[]` → GitHub URLs
- `profile.twitter[]` → Twitter/X URLs
- Any URL found in `profile.summary` matching a personal-domain pattern (`*.com`, `*.io`, `*.dev`, `*.xyz`, `*.me`) that isn't a company domain

**Phase B — Web search fallback.** For each platform below NOT already found via ContactOut, run the full 4-round search pass per memory `feedback_online_presence_look_harder.md`. Time-box: 5–8 minutes across all searches. All 4 rounds are mandatory by default — execute the checklist below IN ORDER, completing each round before moving to the next. Do NOT stop after Round 1 unless you've validated every candidate account.

1. **Round 1 (baseline)**: run the site-specific queries from the source table (`"{Name}" site:x.com`, `"{Name}" site:github.com`, etc.). Collect obvious handle-match candidates.
2. **Round 2 (disambiguate via bio facts)**: combine name with unique identifiers (employer + school + city + specific product). Cross-check EVERY Round 1 candidate against known bio facts — discard any account whose location/employer/year doesn't match.
3. **Round 3 (name/handle variants)**: try non-obvious patterns (middle initial, underscore, number suffix, locale suffix, first-initial + last-name).
4. **Round 4 (verification of candidate links)**: for every surviving candidate, WebFetch the bio/description and require at least 2 fact matches (e.g. location + employer, or employer + year) before keeping. Unverified candidates are excluded.

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

**Write to Notion via the public API.** As of 2026-05-13 Notion's public API supports external-URL writes to Files properties directly — the prior osascript + Chrome path is obsolete. Shell out to the helper:

```bash
python3 ~/.claude/scripts/notion_files_property.py \
    --page-id <neg1_scanner_page_id> \
    --prop "Online Presence" \
    --url "<discovered_url>" \
    --label "<display_label>"
```

See `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md` for the canonical interface. Exit 0 = success (idempotent on URL); exit 1 = hard failure.

**Display name convention** (for consistency across all -1 Scanner rows):
- **Social/account URLs with a handle**: `"[Platform] (@handle)"` — e.g., `"X (@gregreiner)"`, `"Instagram (@greg)"`, `"Farcaster (@greg.eth)"`, `"GitHub (@greg)"`
- **Personal sites / articles / other**: descriptive phrase — e.g., `"Personal site"`, `"Hamden Hall alumni feature"`, `"Acquired podcast guest"`
- **Writing platforms with a custom subdomain/slug**: `"[Platform]: [Publication/Handle]"` — e.g., `"Substack: greg.substack.com"`, `"Medium: @greg"`

**Call once per discovered URL** — the helper is self-contained read-modify-write; sequential calls correctly append without clobbering.

**Fallback (only on helper exit 1)**: write the list as a markdown `## Online Presence` section in the page body via `notion-update-page`. Flag in the Step 6 report so Tom knows the canonical location moved. Retry the helper on the next run.

**Empty**: if ContactOut + web search both return nothing, leave Online Presence empty. Don't fabricate.

### Step 5: Apply the Rubric and Write the Evaluation

Read the framework + rubric from co-located references:
- `founder-taste/CASEBOOK.md` — 6-founder calibration corpus (per-signal scoring evidence)
- `ONLINE_SOURCES.md` — Phase 2 online research taxonomy
- `../founder-taste/RUBRIC.md` §5–§8 — current rubric (signals + archetypes), anchors, thresholds (canonical)

Apply the framework in two phases:

**Phase 1 — derive HIGH-fidelity signals from already-ingested structured data:**
- **Non-Linearity**: count function/discipline crossings across `experience[].job_function` + title transitions.
- **Earned Reps**: cross-reference `experience[]` tenure against the highest-fidelity hypergrowth signal available, in priority order (per founder-taste/RUBRIC.md §Signal 2): (1) **CC Momentum** rollup on the -1 Scanner row — pulls Deal Digest revenue traction directly from the primary employer's Companies DB row, no extra hop required; (2) Companies DB Headcount + HC Commentary (Sales Nav scrape); (3) Companies DB Hypergrowth Windows (funding-round cadence). When tenure overlaps best-in-class peer-tier ramp (e.g., 6-10x+ YoY ARR for early-stage), rate High. Apply sector-difficulty multiplier (health systems, public sector, defense, regulated finance) when the traction was earned in a hard buyer environment.
- **Range**: triple intersection of `job_function` × `industry` across employers (Technical / Commercial / Domain).

**Phase 2 — narrative research for LOW–MED fidelity signals (only if Phase 1 doesn't already disqualify):**
- **Intellectual Rigor**: WebFetch publications, blog posts, podcast transcripts surfaced via Online Presence (per `ONLINE_SOURCES.md`). Read for non-performative tone, self-correction, hedged register. If absent, flag as "unobservable from public" — do NOT score 0.
- **Anticipation**: WebSearch on `projects[]` / `headline` topics with date filters — was the bet pre-consensus at the time? Cross-check internal employer projects too.
- **Intentionality**: LLM read of `experience[].summary` for demotion / anti-accelerator / patient-tenure markers; "On leaving X" essays; podcast career-arc framing.

**Compute the verdict (per founder-taste/RUBRIC.md §8):**
- Rate each signal **High**, **Medium**, or **Low** using the rubric anchors in founder-taste/RUBRIC.md. Pure spike-based MAX — one singular High is enough. **No Intentionality gate** (Intentionality is informational, not a veto).
- `Claude Rec` based on peak signal rating:
  - Any signal High → `Reach Out ✅`
  - Peak Medium → `Second Look 🤔`
  - All signals Low → `Pass ❌`
- **Apply the When gate (RUBRIC.md §8.9) AFTER the verdict.** From `experience[]` career-state evidence, check:
  - **W1 stale peak**: peak-signal evidence ended 3+ years ago AND roles since are comfort-mode (ascending employee titles at mature companies, advisory/angel float, no new 0→1 exposure).
  - **W2 career-stage ceiling**: 5+ years single-lane senior role at an established company with no discipline crossings since the peak.
  - **W3 fresh seat elsewhere** (OBSERVE-ONLY — annotate, never downgrade): took a new employee seat at someone else's company within the last ~12 months, or is mid-tenure inside a current rocketship with visible upward trajectory.
  If W1 or W2 fires AND Claude Rec is `Reach Out ✅`: keep the pre-gate rec in the Signals `rec:` element, downgrade `Claude Rec` to `Second Look 🤔`, and prepend one line to Eval Summary — `When gate: W# – <one-line evidence>.` If only W3 matches: keep the rec unchanged and append `When note: W3 pattern (observe) – <evidence>. Watchlist: re-check in 12-18 months.` The gate never auto-passes, never changes signal scores, and never fires on someone already in a founder-shaped seat (founding now, running their own firm, or visibly between stints exploring). The `rec:` element of the Signals line is the PRE-gate auto-rec — that is what lets the ledger back-test the gate itself.
- `Working Description` (2-3 sentence TL;DR, anchored on the peak signal).
- `Signals` (compact property line `NL:{n} · Reps:{n} · Rigor:{n|U} · Ant:{n} · Int:{n} · Rng:{n} · rec:{✅|🤔|❌}` (0-10 numbers; U = unobservable; legacy rows may carry H/M/L letters; `rec` = the PRE-gate auto-rec)) — the machine-readable scores every downstream parser reads.
- **Eval Rationale (page BODY, not a property)**: append a `## Eval Rationale (YYYY-MM-DD)` heading + per-signal prose paragraphs with evidence URLs from Phase 2 research. This replaced the deleted `Signals` property (2026-07-16 — too wordy for the table); the depth lives one click in, the row stays scannable.
- `Eval Summary` — use the canonical structure from memory `feedback_eval_summary_format.md`: **Primary Signal: [Name].** paragraph → **Other Qualities** bullets (• **Signal Name (Rating).** sentence.) → **Gaps.** paragraph. When Intentionality is unobservable from public data or rated Low, say so explicitly in one clause (e.g. "Intentionality unobservable from public sources" in Gaps) — informational only, never a veto; the no-Intentionality-gate scoring rule is unchanged.

Write all four fields back to the -1 Scanner row via `notion-update-page`. Do not touch Status here (Task 6 mode sets `Enriched` in Step 6; every other status transition belongs to founder-outreach or Tom).

### Step 6: Chain to Drafting (manual invocation only)

The -1 Scanner's `Status` field workflow:
- `Pending Enrichment` → initial state; picked up by pipeline-agent Task 6
- `Enriched` → enrichment + scoring done, verdict written; awaiting Tom's triage (set by Task 6 mode after Step 5; manual runs skip past it because they chain straight to drafting)
- `Draft Requested` → Tom pressed the **Request Draft** button on the row (or flipped the dropdown); the notion-webhook Worker fires `founder-outreach` in webhook mode within ~1–2 min, with pipeline-agent Task 6's sweep as backstop
- `Draft Ready` → after `founder-outreach` produces a Gmail draft (set BY founder-outreach)
- `Reached Out` → Tom sent the outreach (detected by pipeline-agent Task 7)
- `Passed` → Tom decided not to send (manual flip)

**Manual mode** (called with a raw LI URL from a user prompt, or any direct trigger phrase): after Step 5 writes the verdict, **auto-invoke `founder-outreach`** to produce the Gmail draft. Run regardless of Claude Rec — even Pass verdicts get a draft so Tom can see what the outreach would have looked like and make the final send/discard call himself. Do NOT pause to ask for confirmation between the two skills. Single run, single report back.

**Exception**: if the user explicitly says "enrich only" / "just score, don't draft" / "no outreach" / similar, skip the chain and stop after Step 5.

**Task 6 mode** (called by pipeline-agent with an existing page ID, scheduled bulk enrichment): do NOT chain. After Step 5 writes the verdict, set `Status = Enriched` (the one exception to Step 5's "do not touch Status" rule — Task 6 mode owns this transition), then return. Drafting stays Tom-initiated: he triages the `Enriched` rows and presses **Request Draft** on the ones that warrant outreach.

**`--score-only` mode**: does NOT chain (it's a re-evaluation, not a fresh outreach decision).

The split: `neg1-enricher` produces the scored row. On manual invocation it also triggers `founder-outreach` as a terminal step. `founder-outreach` owns the Gmail draft itself.

### Step 6.5: Post-Write Field Validation

After all writes complete (Steps 4–6), re-fetch the -1 Scanner row via `notion-fetch` and audit every field below. For each gap, attempt an inline fix before reporting. This is a mandatory quality gate — not optional, not skippable.

**Required fields — fail if empty or placeholder:**

| Field | Pass condition | Auto-fix |
|---|---|---|
| Page icon | Non-null icon set | Re-attempt with `profile.profile_picture_url`; if still missing, flag |
| Name | Non-empty string, not in `"{Company} – {role}"` placeholder format | Re-fetch from ContactOut and re-write |
| LI | Non-empty URL | Flag — cannot auto-fix |
| Type | Exactly `Warm ☀️` or `Cold 🧊` | Re-write with correct value |
| Role | Non-empty string | Re-extract from ContactOut payload and re-write |
| CurrentCo (CC) | ≥1 relation | Re-run Step 3 for primary employer and re-write |
| Work History | ≥1 relation | Re-run Step 3 for experience array and re-write |
| Claude Rec | Non-null select | Re-run Step 5 verdict derivation and re-write |
| Eval Summary | Non-empty, ≥100 chars, contains "Primary Signal:" and "Other Qualities" and "Gaps." | Re-run Step 5 and re-write |
| Signals | Non-empty, ≥100 chars | Re-run Step 5 and re-write |
| Working Description | Non-empty | Re-run Step 5 and re-write |
| Experience Summary | Non-empty | Re-extract from ContactOut payload and re-write |
| Last Enriched | Set to today's date | Write today's date |

**Conditional fields — fail only if upstream data was available:**

| Field | Pass condition |
|---|---|
| Email | Populated if `profile.personal_email` or `profile.work_email` was non-empty in the ContactOut response |
| City | Populated if `profile.location` was non-empty |
| Function | Populated if the role title maps to any of the defined categories |
| School(s) | ≥1 relation if `profile.education` had non-study-abroad entries |
| Field(s) of Study | Non-empty if School(s) is populated |
| LI Profile Summary | Populated if `profile.summary` was non-empty |
| Online Presence | ≥1 entry if Phase A or Phase B searches returned validated URLs |

**Company and school icon check (run for every page in CurrentCo (CC), Work History, and School(s) relations):**

For each company or school page in these three relations, call the Notion REST API to check whether the page has an icon set. For any page missing an icon:
1. Derive the domain from its `URL` property on the Companies DB page.
2. Verify `https://icons.duckduckgo.com/ip3/{domain}.ico` returns HTTP 200.
3. If yes, PATCH the page icon via `PATCH /v1/pages/{page_id}` with `{"icon": {"type": "external", "external": {"url": "https://icons.duckduckgo.com/ip3/{domain}.ico"}}}`.
4. If no (404/redirect), flag the page in the Step 7 report as "icon missing — DuckDuckGo returned no result."

**Formatting checks:**

- `Eval Summary` must follow the canonical structure: `**Primary Signal: [Name].** [paragraph]` → blank line → `**Other Qualities**` → bullet list → blank line → `**Gaps.** [paragraph]`. If malformed, re-write.
- `Field(s) of Study` must use the normalized degree format: `"Field (Bachelor's)"`, not `"B.S."` / `"BS"` etc. If raw abbreviations detected, normalize and re-write.
- `Experience Summary` entries must be separated by a blank line and use bold company-name prefix. If malformed, re-write.

**Online Presence URL verification (post-write):**

After the row's writes complete, verify every URL in the `Online Presence` Files property is live: issue an HTTP HEAD (fall back to GET if HEAD is rejected) for each. Treatment:

- **2xx/3xx** → OK.
- **`linkedin.com` URLs returning 999 or 403** → treat as OK (LinkedIn auth-walls automated requests; this is not a dead link).
- **404 or connection-dead** → remove that entry from the `Online Presence` property (re-write the Files property without the dead entry via the public API) and report the removed URL in the Step 7 run summary as `Online Presence: removed dead link <url> (404)`.

**Accumulate all auto-fixes and gaps.** Pass the list to Step 7 for inclusion in the report.

---

### Step 7: Report Back

After creating (or updating) entries, provide a brief confirmation per person: name, primary company, role, **peak signal + rating (Strong/Moderate/Weak) + Claude Rec**, and a link to the Notion page. Surface a one-line excerpt of the Eval Summary so Tom can quickly scan whether the verdict reads right.

For batches, present results as a summary table sorted by peak signal rating descending (High → Medium → Low).

---

## `--score-only` Mode (re-scoring without re-enriching)

Per founder-taste/RUBRIC.md §8.7 (re-scoring without re-enriching). Used when the rubric changes and Tom wants to re-score existing rows against the new rubric without burning ContactOut credits re-pulling data that hasn't changed.

**Trigger**: user passes `--score-only` flag, OR uses phrases like "rescore [name]", "re-score [name]", "score-only this row", "rescore the calibration corpus".

**Behavior**:
- **Skip Steps 1–4.5 entirely** — no ContactOut calls, no online presence re-research, no Companies DB writes.
- **Run Step 5 only**, against whatever data is already on the -1 Scanner row (Experience Summary, LI Profile Summary, Online Presence files, Companies relations + their Headcount/CC Momentum/HC Commentary).
- **Overwrite** Signals, Working Description, Claude Rec, Eval Summary with the new verdict.
- **Preserve Status** — a row that's already `Reached Out` stays `Reached Out` regardless of the new rating. We're updating the evaluation artifact, not the workflow state.
- **No Step 6 chain** to founder-outreach in score-only mode (it's a re-evaluation, not a fresh outreach decision).

**Targeting**:
- Single row: pass a -1 Scanner page URL or person name.
- The 6-founder calibration corpus (per founder-taste/RUBRIC.md Future State item 14, quarterly drift check): pass `--score-only --calibration-corpus` (resolves via the names in `founder-taste/CASEBOOK.md`).
- Bulk: pass `--score-only --where "Claude Rec is null"` or any Notion filter. Use sparingly — re-scoring 100+ rows takes time even without ContactOut calls.

**Snapshot before overwrite**: write before-state to `/tmp/score_only_snapshot_{timestamp}.jsonl` (one JSON per row: page_id, name, before_eval_breakdown, before_eval_summary, before_claude_rec). Lets Tom diff old vs new verdicts and revert if a rubric change misfires.

**Reporting**: after a `--score-only` batch, surface a diff table — for each row, show `before → after` on Claude Rec. Highlight rows where the verdict changed (Reach Out → Second Look, etc.).

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
5. Runs Step 5 evaluation → Earned Reps: High (peak signal), Claude Rec = Reach Out ✅.
6. Auto-chains into `founder-outreach` → Gmail draft created, Status = Draft Ready.
7. Returns: "Added Jane Doe (VP of Engineering, Stripe) to -1 Scanner. Earned Reps: High / Claude Rec: Reach Out ✅. Gmail draft ready. [link]"

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
