---
name: neg1-sourcing
description: >-
  Weekly Monday sourcing sweep — surfaces 2 warm reconnects + 8 cold candidates (2 wildcards; other 6 rotate
  archetype recipes A-D from RUBRIC.md §6), plus monthly network deep sweep + quarterly departure diff (first
  Monday after the Jan/Apr/Jul/Oct cache refresh). Upserts each candidate to the CANDIDATE STORE (state=pending)
  and immediately enqueues a per-candidate enrichment job (enqueue-neg1-enrich.sh) — cards post to
  #neg1-sourcing within minutes; NO batching delay, NO Notion writes. Dedup reads the store. Slack digest posts
  to #neg1-sourcing.
triggers:
  - /neg1-sourcing
---

# neg1-sourcing — Weekly Pre-Founder Sourcing Sweep

Runs every Monday at 08:00 ET. Produces 2 reconnect + 8 cold outreach candidates (2 wildcards) from Tier 1–3 high-growth companies, upserts them to the candidate store (`state=pending`), enqueues per-candidate enrichment jobs, and sends a Slack digest. (The old -1 Scanner / `Pending Enrichment` queue wording is retired — DB deleted 2026-07-16.)

**Unattended execution guard:** never ask questions, never halt waiting for input. If a step fails, skip it, log the error, and continue. Always reach the Slack alert even if some rows failed to write.

---

## v2 HEADLESS FLOW (2026-07-16 — CURRENT)

Candidates no longer land in the -1 Scanner. Steps 2-3 (Notion row writes) are replaced by store upserts — for each verified candidate:
```bash
python3 ~/.claude/scripts/decision-ledger/candidates.py upsert --json '{"li_url": "...", "name": "...", "city": "...", "current_role": "...", "current_company": "...", "type": "Warm ☀️|Cold 🧊", "source": "monday-sweep", "recipe": "{candidate recipe field: A-F, or the wildcard_signal for wildcard rows; omit for warm/backfill rows}", "state": "pending"}'
```
then IMMEDIATELY enqueue a per-candidate enrichment job — enrichment happens as names surface, never on a batch delay (Tom, 2026-07-16: "there shouldn't be a delay"):
```bash
~/.claude/scripts/enqueue-neg1-enrich.sh "<li_url>" "monday-sweep" "<name>"
```
Cards post to `#neg1-sourcing` within minutes as each job completes. This applies to the weekly sweep AND the Step 1.75 monthly deep sweep / departure diff. pipeline-agent Task 6 is the daily reconciliation backstop for `state=pending` rows older than ~2 hours (missed/failed jobs) — it runs nightly at 17:50 via the scheduled orchestrator's `pipeline-neg1` sub-task (wired 2026-08-12; before that the claim pointed at a runner that didn't exist). The Step 4 Slack digest is unchanged. Everything referencing -1 Scanner writes below is LEGACY.

---

## Type field reference

The `Type` select in -1 Scanner uses these existing options (do not rename or add new ones — neg1-enricher and downstream skills key off these exact strings):

| Option | Color | Meaning |
|---|---|---|
| **Warm ☀️** | orange | The person EXISTS in the network cache (`~/.claude/scripts/network_cache.db`, `profiles.linkedin_url`) |
| **Cold 🧊** | blue | Not in the network cache |

**The warm/cold call is purely mechanical — network-cache membership, nothing else** (Tom, 2026-07-27: "warm is when someone exists in the network cache, cold is if not"). Check the cache by slug for EVERY candidate before upserting, whichever pass surfaced them — a cold-recipe search can surface someone who happens to be a connection (→ Warm ☀️), and downstream (enricher cards, digest sections) trusts the store `type` without re-deriving. Always write the exact strings `Warm ☀️` / `Cold 🧊` (never bare "Cold" — two 07-27 rows landed unstyled and broke string matching).

Growth Tier and Timing Signal are intentionally NOT separate fields — that context is folded into the Eval Summary prose by neg1-enricher via CC Momentum + tenure overlap.

---

## Supply model (rewritten 2026-08-11)

**Warm / reconnect pass — employer tier-eligibility is a RANKING SIGNAL, not a gate.** Tom,
2026-08-11: *"having folks work on the companies in tier eligible companies shouldn't be a hard
rule."* The pass used to INNER JOIN the network cache against tier-eligible Companies DB rows. That
read as doctrine but measured as a data-coverage artifact: of 364 network profiles passing the role
whitelist and prefilters, only **10** cleared the gate, and **301 were invisible purely because
their employer is not in the Companies DB at all** — including people at Anthropic, Cursor,
Perplexity, Figma, Rippling, Mercury, Vanta, Harvey, Sierra and Notion. It also cut against the
thesis: requiring a legible employer is backwards for an engine that wants people *before* they are
legible, and it made a stealth or unknown employer — a strong -1 marker — disqualifying.

The pass now scans the whole cache and orders by rank band:

| Band | Meaning |
|---|---|
| 0–2 | Employer is tier-eligible, growth_tier 1 / 2 / 3 |
| 3 | Employer **not in the Companies DB** — never evaluated; stealth/early is a positive marker |
| 4 | Employer in the DB but not tier-eligible — assessed, did not qualify |

Only `RECONNECT_COUNT` are taken per run, so the ranking (not a gate) is what holds quality: the
tier-eligible bands are exhausted first and the long tail behind them is real supply rather than
nothing. Admissible pool went 10 → 181. Quality now rests on the exclusion layer — funds (PF-1),
institutions (PF-13), mature enterprises (PF-4), stale unicorns (PF-3), founder seats (PF-10),
non-NA (PF-11) — so **that layer is load-bearing in a way it was not before**; see PREFILTERS.md
"Rule expansions of 2026-08-11".

**THE DEAL DIGEST IS THE ONLY SANCTIONED COMPANY LIST (Tom, 2026-08-11, closing rule):** *"apart
from the few -1 founders you're filtering explicitly from the deal digest, you should not be going
after a fixed list of companies."* A digest write-up is Tom's own curation event; a Companies DB row
is not (*"there are so not so great companies in the company db"*). Everything company-anchored —
the `COLD_SLOTS_COMPANY = 3` reserved pass and recipe C — now sources from `deal-digest-cache.json`.
Recipes A and B lost their anchors entirely and match on arc shape instead.

**Cold slot split.** `COLD_SLOTS_COMPANY = 3` of the 8. Tom: *"this is a cold outreach engine too,
so i don't mind you flagging folks I'm not connected to yet at those companies."* Composition:

| Week | Digest-anchored | Company-free (Step 1.55) |
|---|---|---|
| A, B, D, E, F | 3 of 8 | **5 of 8** |
| C | 6 of 8 | 2 of 8 |

The engine is now taste-driven by default and list-driven only where Tom explicitly curated the
list. It previously ran ONLY on recipe shortfall,
so the 2026-08-11 run gave it 1 slot of 8 while recipe D took 5 — and **3 of those 5 died on
prefilters** (the run's other two kills were a wildcard row and the generic backfill; an earlier
draft of this line said "4 of 5" and was wrong — D also produced the run's two strongest names,
Brendan Joyce and Abhishek Pawar). Current split: **2 wildcard + 3 recipe + 3 company-anchored**,
with the company-anchored pass also absorbing any recipe shortfall so an underfilled recipe week
still ships a full digest.

---

## Step 1 — Run the sourcing script

```bash
/opt/homebrew/bin/python3 /Users/tomseo/.claude/skills/neg1-sourcing/neg1_sourcing.py run
```

Captures stdout as JSON: `{run_date, reconnect: [...], cold: [...]}`.

Each candidate object:
```json
{
  "linkedin_url":       "https://linkedin.com/in/...",
  "name":               "Jane Smith",       // empty string for cold candidates
  "role":               "Staff Engineer",
  "email":              "jane@...",         // empty for cold
  "company":            "Replit",
  "company_notion_url": "https://notion.so/...",
  "city":               "San Francisco",
  "function":           "Engineering",      // Engineering | Product | GTM
  "growth_tier":        "Scaled",           // Scaled | Emerging | Promising
  "timing_signal":      "Early",            // Early | Rising | Late | Unknown
  "arr_m":              300.0,              // null if unknown
  "type":               "Warm ☀️",          // Warm ☀️ | Cold 🧊
  "wildcard":           true,               // wildcard rows only (2 of the 8 cold — script _wildcard_pass)
  "wildcard_signal":    "reps-employee-one" // which extreme-signal template surfaced them
}
```

Wildcard rows (implemented in the script since 2026-07-27, `_wildcard_pass` + `WILDCARD_QUERIES`) arrive with empty `role`/`company`/`function` — there is no target company; the Step 1.5 probe fills them. Upsert wildcard rows with `"source": "wildcard"` (NOT monday-sweep) and pass `"wildcard"` as the source arg to `enqueue-neg1-enrich.sh` — the ledger's quarterly wildcard-conversion review keys on that source value.

If the script exits non-zero or returns 0 candidates total, skip to Step 4 (Slack alert) and report the failure.

---

## Step 1.5 — Verify cold candidates via ContactOut

**Reconnect candidates are skipped** — they come from the LinkedIn network cache (real people Tom knows) and don't need verification.

For each **cold candidate**, call `contactout_enrich_linkedin_profile` with `profile_only: true` before writing anything to Notion. This is a cheap probe (no email credit consumed) that confirms the LinkedIn URL resolves to a real person who currently works at the target company.

**Verification criteria — both must pass:**
1. **Profile exists**: the response returns a non-empty `full_name`. An empty/null name or an API error means the slug is a ghost → discard.
2. **Company match**: the person's current employer (first `experience` entry with `is_current: true`, or the most recent entry) fuzzy-matches the candidate's `company` field. Use case-insensitive substring match — `"Morgan Stanley"` should not match `"Replit"`. If no current employer is present in the response, treat as mismatch → discard.

**Capture from the probe (survivors):** set the candidate's `name` to the probe's `full_name`, and overwrite `role`/`company` with the probe's current title + employer — the probe data is live and properly cased, the search-derived values are often stale lowercase keywords. These captured values flow into the store upsert AND the Step 4 digest (Tom, 2026-07-27: real names, not slugs; proper-noun capitalization).

**Wildcard rows verify existence-only:** they have no target company (the query described an arc, not an employer), so criterion 2 does not apply — a non-empty `full_name` passes, and the probe's current title/employer fill the empty `role`/`company`. Everything else (dedup, prefilters, store upsert, enqueue) is identical, except `source="wildcard"`.

**On discard**: remove the candidate from the cold list and log: `[REJECTED] {url} — {reason}` (reason: "ghost URL" or "employer mismatch: expected {company}, got {actual_employer}"). Do not replace with a substitute — if fewer than 3 cold candidates survive, proceed with however many passed.

**Process cold candidates sequentially** — don't batch the ContactOut calls in parallel to avoid rate limits.

---

## Step 1.55 — Archetype adjudication (COMPANY-FREE rows only; added 2026-08-11)

**Applies to every candidate NOT sourced from a company anchor** — recipe D/E/F rows and wildcard
rows. Company-anchored rows (generic cold pass, recipes A/B/C) skip this step: their employer is
already a vetted quality signal, which is the entire point of anchoring.

**Why this exists.** Tom, 2026-08-11: *"some candidates should absolutely come from the high flying
companies in our cache, but the rest could be folks who meet my founder taste / archetypes without
necessarily coming from a static list of companies in my cache."* The company-free half is the only
half that can outgrow the cache — but it had NO quality gate. A neural Exa hit went straight from
search to ContactOut verification to enqueue, with nothing ever asking *does this arc actually match
an archetype?* That is why recipe D shipped a Bain consultant, a JPM treasury-sales analyst and a
25-year finance-ops manager on 2026-08-11: the queries matched narrative vocabulary, and no step
downstream checked the arc.

**The mechanism — cheap recall net, then judgment.** This is the pattern that worked on the
2026-08-11 demotion-to-switch mining: a loose detector for RECALL, then an agent reading the actual
arc for PRECISION. Four rule-based iterations there went 4/4 → 1/4 → 2/4 → 0/4 true positives, while
LLM adjudication over the same recall net cleanly separated real crossings from acquihire landings
and fraternity presidencies. Rules cannot express these archetypes; reading the arc can.

**Procedure.** After the Step 1.5 ContactOut probe (which already returns the full `experience[]`),
for each company-free candidate read `~/.claude/skills/founder-taste/RUBRIC.md` §5 (six signals) and
§6 (four archetypes + the Tuor composite) and judge the ARC — not the headline, not the search
snippet:
1. Which archetype, if any, does this arc read on? Name it, or answer "none".
2. What is the concrete evidence in `experience[]`? Cite the roles and dates.
3. Known false-positive shapes to reject explicitly — all observed on 2026-08-11:
   - **Founder-artifact** — a "Co-Founder / CEO / CTO" line followed by a lower title is usually an
     acquihire landing or a return to the person's own craft, not a signal.
   - **Direction-blindness** — an arc that touches both commercial and technical domains but travels
     the WRONG way (engineering → consulting reads as "crossed into engineering" to a text matcher).
   - **Student-title inflation** — fraternity/club presidencies and student consulting clubs read as
     senior commercial seats.
   - **Narrative-only match** — the profile *talks* about a pivot but the arc does not show one.
4. **PF-14 — venture-backed startup pace.** Does the arc show, anywhere, that this person has
   operated at venture-backed startup speed? Tom, 2026-08-12: *"one thing i care about is that folks
   understand the pace of working at a venture-backed startup (the speed is FAST)."* Domain
   expertise and technical range do not substitute — kill an arc spent entirely in traditional
   companies however deep the expertise. **Judge operating tempo, NOT name recognition:** a
   seed-stage company neither of you has heard of passes; a legible 70-year-old freight forwarder,
   bank, agency or family business fails. Getting this backwards re-introduces the legibility bias
   the whole engine was rebuilt to remove — see PREFILTERS.md PF-14, which spells out the tension
   with the "keep it open" rule.
5. **Kill anything that reads on no archetype.** Log `[NO-ARCHETYPE] {name} — {one clause}` to the
   audit log. These are NOT named in the digest's Filtered-out section — that section is for
   prefilter (PF) kills, where a *rule* may be wrong. A no-archetype kill means the search was
   loose, which is expected for a wide recall net and not something Tom needs to adjudicate.

Survivors carry their archetype read into the store upsert, so the enricher and the quarterly
back-test can compare *sourced-as* against *scored-as*.

## Step 1.6 — Prefilter screen (Tom-taught hard disqualifiers)

Read `~/.claude/skills/founder-taste/PREFILTERS.md` and screen every surviving candidate (warm AND
cold) against each rule checkable from the data in hand (role, company, cache tenure — arc-shape
rules wait for enrichment). The script already mirrors PF-1/PF-3 in code (EXCLUDE_ROLES, fund-name
heuristic, STALE_UNICORNS); this pass catches what code can't express — judgment calls like "this
cached role reads as an investor seat" or "this employer is past its window per the momentum field".

On a kill: drop the candidate, do NOT upsert or enqueue, and log `[PREFILTERED] {name/url} — {rule id}: {one clause}`
to the audit log ONLY. Prefiltered candidates get NO individual card, but ARE named as one-liners in the
digest's "Filtered out" section (Step 4 — that section explicitly superseded the 2026-07-20
never-reveal ruling); the audit log carries the full rule-id detail.

---

## Step 2 — Resolve company Notion relations

For each candidate, look up the company in the Companies DB to get the Notion page URL for the relation field:
- Use `company_notion_url` from the JSON — this is already the Notion page URL.
- If it starts with `digest://`, the company is a digest-only synthetic row with no real Notion page — omit the Company relation for that row, just set the company name as a text note in Role.

---

## Step 3 — Upsert candidates to the store

> ⛔ **Execute the v2 block at the top of this file, NOT the legacy table below.** The `-1 Scanner`
> Notion DB was **deleted 2026-07-16** — `notion-create-pages` against
> `32c00bef-f4aa-80a5-923b-000b83921fa3` (or `32c00bef-f4aa-8041-90e0-ff3f0e0dbff5`) returns **404**,
> so this step wrote nothing on every weekly run until it was corrected on 2026-08-11. The identical
> failure was found the same day in `pipeline-agent` Tasks 6/7 and `neg1-enrichment-sweep`; see
> [[feedback_skill_authoring]] "Retiring a data source".

**Do this:**

```bash
python3 ~/.claude/scripts/decision-ledger/candidates.py upsert --json '{"li_url": "...", "name": "...", "city": "...", "current_role": "...", "current_company": "...", "type": "Warm ☀️|Cold 🧊", "source": "monday-sweep", "recipe": "...", "state": "pending"}'
~/.claude/scripts/enqueue-neg1-enrich.sh "<li_url>" "monday-sweep" "<name>"
```

**Dedup:** read the store, not Notion — `candidates.py get --li <url>`. If a row exists, skip and
note it in the audit log.

**Field mapping — LEGACY REFERENCE ONLY.** The table below describes the deleted Notion DB's
properties. It survives because the *semantics* still map onto store columns (`name`, `li_url`,
`current_role`, `city`, `current_company`, `type`, `email`) and because `Type` values are keyed on
by neg1-enricher. Do not execute it as a write.

| -1 Scanner field | Value | Notes |
|---|---|---|
| Name | `{name}` if available, else `{company} – {role}` (cold rows have `name=""` until enrichment resolves it; **never** prefix with `(cold)` — Tom reads the title at a glance and the Type field already encodes warm/cold) | title property |
| Status | `Pending Enrichment` | select |
| LI | `linkedin_url` | url |
| Role | `role` | rich_text |
| Function | `function` | select — Engineering / Product / GTM |
| City | `city` if non-empty | single select |
| CurrentCo (CC) | relation to `company_notion_url` | omit if digest:// |
| Type | `type` | select — Warm ☀️ / Cold 🧊 |
| Email | `email` if non-empty | email |

**Do not populate:** Growth Tier, Timing Signal, Claude Rec, Eval Summary, Online Presence, Work History, School(s), Experience Summary, LI Profile Summary, or any other field — neg1-enricher handles evaluation; growth/timing context flows in via CC Momentum.

Upsert each candidate individually. If one fails, log the error and continue to the next.

**After upserting — chain to enrichment:**
- **Both modes**: immediately run `enqueue-neg1-enrich.sh` per candidate (see the v2 block). Enrichment happens as names surface, never on a batch delay — cards post to `#neg1-sourcing` within minutes.
- `pipeline-agent` Task 6 is the daily **reconciliation backstop** for `state=pending` rows older than ~2 hours (missed or failed jobs) — not the primary path, and no longer keyed on a `Pending Enrichment` Notion status.

---

## Step 1.5b — Cold-pass cohort recipes (archetype feeders)

6 of the 8 cold slots draw from a WEEKLY ROTATING archetype recipe (RUBRIC.md §6 sourcing strategies) instead of generic role × tier search. **Implemented in `neg1_sourcing.py` since 2026-07-27** (`_recipe_pass` + `RECIPE_ROTATION`; before this the table was prose-only and every run shipped generic role × tier rows). Rotation by ISO week number mod 6; the run JSON carries `recipe` at top level and per-candidate, and the store upsert writes it to the `recipe` column (added same day) so the ledger can back-test recipe conversion like wildcard conversion:

| Week | Recipe | Query shape (mechanism) |
|---|---|---|
| A | **FDDM** (Field-Derived Domain Mastery) | **NO company anchor** — neural queries for the person: implementation / CS / solutions operators who onboarded hundreds of customers in one industry. Employer may be unknown to us; the domain reps are the signal. Gated by Step 1.55 |
| B | **TCDM** (Technical-Commercial Dual Mastery) | **NO company anchor** — neural queries for the person: hired to write code, pulled customer-facing because customers asked for them by name. Gated by Step 1.55 |
| C | **Legible-company alumni** | early employees of companies Tom has WRITTEN UP, now senior — anchors from the Deal Digest cache (~575, weighted by growth evidence + recency). **The only company-anchored recipe** (neural per-company query) |
| D | **Composite markers** | range-as-reluctant-stretch only — hired technical, pulled customer-facing (1 neural query). **Behavior 1 (demotion-to-switch-disciplines) is PARKED as a structured pass, `D_STRUCTURED_TODO`** — see below |
| E | **Take-It-Slow** (added 2026-07-27, Tom-gated) | high-reps operator in a visible patient exploration gap — advising/researching post-departure, no accelerator (neural shape queries) |
| F | **Slope over y-intercept** (added 2026-07-27, Tom-gated) | elite-school young analyst who jumped from banking/PE/consulting into an unglamorous operating domain, ramping fast (neural shape queries; distinct from the young-infiltrator wildcard, which requires a public artifact) |

If the recipe pass fills fewer than its slots, the generic role × tier pass backfills the remainder (backfilled rows carry no `recipe` value).

**Recipe A dropped its company anchor entirely, 2026-08-11 — there is no list to maintain.** The
progression of Tom's calls that day: *"this shouldn't be rigidly based on 10 hardcoded names... that
list should be open and fluid"* → *"meh i dont think you need to maintain a list at all"* → the
decisive one:

> *"keep it open! if there is evidence that they're at a company that we haven't heard of yet, but
> that from that experience they've built serious domain expertise, that's in scope"*

**FDDM is a property of the PERSON, not the employer.** Requiring a recognizable company measured
legibility rather than domain reps — the same bug as the reconnect pass's tier-eligible JOIN and
recipe C's `growth_tier` gate, all three found the same day. An unknown vertical-software company
that put an operator in front of hundreds of customers produces the walking-encyclopedia effect just
as well as a famous one, and intercepting those people *before* the employer becomes legible is the
whole premise of the engine.

A now runs company-free neural queries (`A_QUERIES`) through Step 1.55 adjudication, like D/E/F.
**The adjudicator must judge domain-expertise evidence in the arc — implementation/CS tenure,
customer volume, consistency within one industry — and must NOT require a recognizable employer.**
The old `FDDM_ANCHORS` list, `FDDM_ROLES`, and a briefly-lived `fddm_anchors.json` are all deleted.

**Recipe C's anchor pool is DERIVED from the Deal Digest, 2026-08-11 — 12 hardcoded names → ~575.**
Tom: *"wait just 12? i feel like the pool needs to be bigger"*, then *"nope — not companies db..
companies featured in deal digest are legible companies. there are so not so great companies in the
company db."* **Legibility = Tom wrote the company up in a Deal Digest.** That is a curation event;
a Companies DB row is not (1,210 rows covering everything ever tracked — passed deals, portfolio,
incidental names). `_hypergrowth_anchors()` now reads `deal-digest-cache.json` and returns a
weighted shuffle: growth evidence parsed from the digest bullets ("6x YoY", "$0 to $8M in 6 months")
plus mention recency and repeat mentions RANK the pool, they do not gate it — a digest company with
no stated multiple is still legible and still worth mining for alumni. Fund/accelerator/institution
exclusions are load-bearing here since there is no tier gate.

Two measured facts behind the change: 11 of the 12 old hardcoded anchors were not even in the
tier-eligible pool (several — Figma, Flexport, Airtable, Scale AI — are past their window), and
gating on `growth_tier` filtered for *"has been processed"* rather than quality: 377 companies carry
momentum commentary but were never tiered, locking out Databento, Blacksmith, Dust.tt, Aikido,
Retell and Kumo. Same class of bug as the reconnect pass's tier-eligible JOIN, found the same day.
The old list survives as `HYPERGROWTH_SEED`, used only if the digest cache is unreadable.

**Recipe D was cut back 2026-08-11 — a mechanism change, not an archetype change.** The D archetype
(RUBRIC.md §6, Composite case: Tuor) decomposes into two observable behaviors. Behavior 2
(range-as-reluctant-stretch) stays a neural query and works: the hybrid end-state is narrated on
profiles, because people literally title themselves "Forward Deployed Engineer". Behavior 1
(demotion-to-switch-disciplines — a seniority DROP and a discipline CHANGE at the same transition)
does not work as a text query and has been parked as `D_STRUCTURED_TODO` in the script, alongside
`WILDCARD_STRUCTURED_TODO`'s liquidity-decliner and scarred-alumnus, for exactly the same reason:
**it is a derived timeline fact that nobody narrates on a profile.** You compute it by diffing
seniority across consecutive `experience[]` entries.

Five formulations were tested live on 2026-08-11 with every top result probed via ContactOut. Naming
the technical end-state returns pure technical ICs (Exa weights the common phrase, drops the rare
clause). Naming the title drop alone returns engineering-management→IC, where the discipline never
changes — not the archetype. Leading with the rare commercial phrase returns bootcamp
career-changers with no Earned Reps. The queries being replaced were worse still: they re-surfaced
candidates Tom had already passed on (Steven Morrisroe, 2026-08-10). D's freed slots flow to the
company-anchored pass, which is vetted. **The archetype is unchanged and still doctrine — only the
retrieval mechanism was wrong.**

**Doctrine coupling:** this table is the sourcing expression of RUBRIC.md §6 — it is NOT independently editable, and the code lists (`FDDM_ROLES/ANCHORS`, `TCDM_ROLES`, `HYPERGROWTH_ANCHORS`, `C_QUERY`, `D/E/F_QUERIES`) are part of it. When an archetype is added, revised, or retired in the rubric (human-gated), update this rotation AND the code in the same change. Recipes never drift from doctrine.

**Wildcard slots (explore vs exploit):** every week, 2 of the 8 cold slots are reserved for candidates deliberately OUTSIDE all current archetypes but carrying ONE extreme signal the rubric respects on a shape it doesn't recognize (e.g. a 10-grade spike on Non-Linearity or Earned Reps in an arc that matches no recipe). Upsert with `source="wildcard"`. Purpose: archetype discovery — the doctrine-coupled recipes can only find shapes past taste already codified. **Implemented in the script since 2026-07-27** (`_wildcard_pass` — Exa neural templates in `WILDCARD_QUERIES`, 2 sampled per run, one candidate each, emitted first in the `cold` array with `wildcard: true` + `wildcard_signal`; before this the wildcard slots existed only in prose and every run shipped 8 recipe/generic cold rows). The template set was locked with Tom 2026-07-27 from a corpus study of his LP letters + investment memos + July 2026 LPAC deck — each template carries its grounding quote as a code comment. **7 active** (family-vertical-insider, moonlighter, young-infiltrator, wedge-strategy-writer, ant-pre-consensus, range-solo-builder, + nl-hard-crossing as the one deliberately OFF-corpus explore slot preserving true archetype discovery) and **2 parked in `WILDCARD_STRUCTURED_TODO`** (liquidity-decliner, scarred-alumnus — the same-day Exa eval returned 0 results for both: they are DERIVED timeline facts nobody narrates on a profile; build them as structured passes over the company cache × ContactOut departures, not as neural queries).

**Wildcard search mechanics (2026-07-27 eval findings):** `_exa_search_wildcard` harvests BOTH profile URLs and `linkedin.com/posts/` URLs — a post narrating the shape in first person is the strongest match, and the author slug is embedded in the post URL. A coarse follower cap (`WILDCARD_MAX_FOLLOWERS`) drops obviously-famous profiles. **The Step 1.6 prefilter screen has two EXTRA kills for wildcard rows:** (1) already-legible people — famous OSS creators, founders of at-scale funded companies; the flip already happened and the -1 engine hunts pre-legibility (Q4 2025 letter: intercept "before they're 'found out'"); (2) performative build-in-public self-promoters whose narration lacks substance — post-derived candidates skew this way, and it is Tom's named anti-signal ("shameless chest-pounding"). Judge the arc, not the volume of narration. When adding an archetype-adjacent template or retiring one whose shape got codified into RUBRIC.md §6, edit `WILDCARD_QUERIES` — same human-gated doctrine coupling as the recipe table. Quarterly, review wildcard conversion in the ledger (`SELECT * FROM decisions WHERE label IN (SELECT name FROM candidates WHERE source='wildcard')`); 3+ wildcard drafts sharing a shape is a new-archetype candidate for the Casebook.

All recipes still pass the ContactOut verification gate (Step 1.5) and the full rubric downstream — the recipe only shapes WHO enters the funnel.

## Step 1.75 — Monthly network deep sweep + departure diff (FIRST Monday of the month only)

The weekly reconnect pass samples the network; this step mines it. Source: `~/.claude/scripts/network_cache.db` (`profiles` table — ~5.8k connections, ~3.5k with live company data).

**A. WHAT-lens deep sweep (top 5):**
1. Pull profiles with non-empty `company`, excluding anyone already in the candidate store (`candidates.py get --li`), already actioned, or in the Opportunities DB.
2. Coarse-score from cache + `company_cache.py` data: employer momentum/hypergrowth (Deal Digest tier, headcount growth) × function fit (engineering / product / technical-GTM from `role`) × tenure signal from `parsed_json`.
3. Top 5 → `candidates.py upsert` with `state=pending`, `type="Warm ☀️"`, `source="network-deep-sweep"`.

**B. Departure diff (cap 3) — runs ONLY on the first Monday after a quarterly cache refresh (Jan/Apr/Jul/Oct; the `network-quarterly-refresh` launchd job re-enriches the full cache via Exa on the 1st at 18:12):**
1. Read `~/.claude/scripts/decision-ledger/network_snapshot.json` (`{li_url: company}` from the last run; if absent, write it and skip the diff this month).
2. Diff current cache vs snapshot: profiles whose `company` changed or emptied = movers — the When window may just have OPENED.
3. Movers passing a coarse founder-shape filter (was at a hypergrowth-cohort employer, technical/product function) → `upsert` with `state=pending`, `source="departure-trigger"` and a note in `path` ("role change detected {old} → {new}").
4. Rewrite the snapshot with current values.
Cadence rationale: the cache refreshes quarterly (not monthly), so a monthly diff would compare static data 2 months out of 3. Quarterly-aligned, the diff catches a full quarter's role changes in one pass at zero marginal cost. If the snapshot predates the last refresh and the cache HAS moved, run; otherwise log "no cache movement since last diff" and skip.

**C. Structured liquidity-decliner + scarred-alumnus pass (monthly, added 2026-07-27):**
```bash
/opt/homebrew/bin/python3 ~/.claude/skills/neg1-sourcing/neg1_sourcing.py structured
```
Emits up to 4 UNVERIFIED candidates (2 per shape) from `"ex-{Company}"` headline searches over curated watchlists (`LIQUIDITY_WATCHLIST` — recent IPO/acquisition; `SETBACK_WATCHLIST` — public stumble/wind-down; both rotate least-recently-swept, state at `decision-ledger/structured_sweep_state.json`). These shapes are DERIVED timeline facts an Exa query can't verify — the probe step must do the math. **Verification (replaces the standard Step 1.5 criteria for these rows):** ContactOut `profile_only` probe, then keep ONLY if:
- **liquidity-decliner:** experience array shows a real stint at the watchlist company ending AFTER its liquidity event (they stayed through it — leaving before doesn't count), AND the current seat is small/unknown/absent (roughly <100 HC, or between roles). The comfortable path was staying; they left.
- **scarred-alumnus:** a 1y+ stint at the setback company overlapping its down chapter (they didn't bail at the first wobble), AND a current arc that reads as processing/rebuilding, not resume-laundering.
Discard anything that fails — the "ex-{Co}" headline population is mostly ordinary alumni; expect to discard most. Survivors: upsert + enqueue with `source` and `recipe` = the shape name (`liquidity-decliner` / `scarred-alumnus`). Watchlists are hand-curated — refresh from Deal Digest when liquidity/setback events land.

## Step 1.9 — QUARTERLY feedback review (first Monday after Jan/Apr/Jul/Oct — same trigger week as the departure diff)

The loop-closer (added 2026-07-27 — until then the back-tests were specced in three places but had NO scheduled trigger; they ran never). On the first Monday after a quarter boundary, after the sweep completes, run the back-tests and post ONE review card to `#neg1-sourcing` (bot-token mode) for Tom to gate:

1. **Wildcard conversion:** `SELECT c.name, c.recipe, c.state, d.decision, d.why FROM candidates c LEFT JOIN decisions d ON lower(c.name)=d.label WHERE c.source='wildcard'` — which templates produced drafts vs passes? 3+ drafts sharing a shape → propose a new Casebook archetype; 0 drafts from a template across 2 quarters → propose retiring it.
2. **Recipe conversion:** same query keyed on `c.recipe IN ('A'..'F')` — which archetypes actually convert to `draft`? Propose anchor-list/query refinements.
3. **Prefilter false-kill audit** (PREFILTERS.md Audit section): would any drafted/reached-out candidate have been killed by a current PF rule? Flag to Tom, never silently relax.
4. **Calibration drift check:** neg1-enricher `--score-only --calibration-corpus` (CASEBOOK.md names) — flag any exemplar whose score moved ≥2 points from its canonical value.
5. **Retro-corpus review:** read `founder-taste/DECISION_RETROS.md` entries from the quarter; propose which recurring nuggets deserve promotion into RUBRIC.md / PREFILTERS.md / recipe or wildcard queries (human-gated — proposals only, in the review card).

The card ends with proposed changes as a checklist; Tom approves/vetoes in-thread and the listener applies approved items (doctrine-coupled edits: rubric + recipes + code mirrors in the same change). NOTHING auto-applies.

## Step 4 — Slack digest

**Channel routing (gate in code):** if `~/.claude/skills/neg1-sourcing/.sourcing_channel_id` exists, post via `send-alert/md_to_blocks.py` in bot-token mode: `SLACK_BOT_TOKEN_FILE=$HOME/.claude/skills/claude-alerts-listener/.bot_token SLACK_CHANNEL=$(cat ~/.claude/skills/neg1-sourcing/.sourcing_channel_id) BODY_FILE=<tmpfile> python3 ~/.claude/skills/send-alert/md_to_blocks.py` (prints the message `ts` — no webhook needed) — all sourcing surfaces live in `#neg1-sourcing` (this weekly digest of raw candidates + pipeline-agent Task 6's post-enrichment Reach Out ✅ cards). If the file does not exist, fall back to the default `send-alert` channel.

Invoke the `send-alert` skill with the following message. Bodies are GFM markdown (see `send-alert/SKILL.md`) — `**bold**` becomes bold, `[label](url)` becomes a clickable link, `*single asterisks*` would render as italic so avoid them.

**Format (Tom's locked shape, 2026-07-27; Deep Sweep section added 2026-08-03):**
```
📡 **-1 Sourcing Summary – Week of {run_date}**

**Warm (2)**
• [{Name}]({linkedin_url}) — {Role} @ {Company} [{growth_tier} · {timing_signal}]
• [{Name}]({linkedin_url}) — {Role} @ {Company} [{growth_tier}]

**Cold (8, incl. 2 wildcards — tag those rows `[wildcard]`)**
• [{Full Name}]({linkedin_url}) — {Role} @ {Company} [{growth_tier}]
• (8 rows)

**Deep Sweep ({N})**
• [{Name}]({linkedin_url}) — {Role} @ {Company} [{growth_tier} · {timing_signal}]
• (one row per monthly deep-sweep / departure-diff candidate)

**Filtered out ({N})**
• [{Name}]({linkedin_url}) — {plain-English reason}
• (one row per prefilter kill this run; omit the section entirely when N = 0)
```

**The Filtered-out section (Tom, 2026-08-10 — SUPERSEDES the 2026-07-20 "never reveal" ruling).**
Prefilter kills still get **no individual card** — that part is unchanged — but they are now named
here with the rule that killed them. Tom's reasoning: "the more you surface the more reactions you
get from me," and a prefilter kill is exactly where a *rule* may be wrong rather than the candidate.
A silently starved funnel is invisible to him; a named list is one line he can react to. A reply
naming someone in this section is a valid signal to relax or narrow the cited rule — route it like
any card reply. Keep to one line per person and never expand into card anatomy.

**EVERY name on this digest carries an embedded LinkedIn link** — both the carded names and the
filtered-out ones (Tom, 2026-08-10: "embed LI profiles to each name"). Always `[Name](full-url)`,
never a bare URL and never an unlinked name; this is the same standing rule that governs every
candidate/contact list on a Tom-facing surface. Prefer the canonical `www.linkedin.com/in/{slug}`
form even when the row was surfaced under a country subdomain. (An earlier draft of this section
said not to link filtered-out profiles on the theory that an audit surface differs from a candidate
surface — that was wrong and Tom reversed it the same day: he wants to click straight through to
judge whether the rule misfired.)

**Blank line between every section — the digest is NOT a card.** Cards ban blank lines (they emit
`\n\n` spacers and break the tight bullet block); the digest requires them between sections, exactly
as the format above shows. Do not carry the card rule over here — that mistake shipped a wall-of-text
summary on 2026-08-10 and Tom had to ask for the breaks back.

**NEVER print a recipe LETTER on this surface — spell the archetype out** (Tom, 2026-08-12: *"without
context and just saying recipe b in the alert... it's hard to know... so don't use recipe... just
spell it out"*). Same principle as the PF-id and W-code bans below: `A`–`F` are internal rotation
machinery and carry no meaning at a glance. Use the archetype name from RUBRIC.md §6, which is Tom's
own vocabulary:

| Internal | Print this |
|---|---|
| A | Field-Derived Domain Mastery |
| B | Technical–Commercial Dual Mastery |
| C | Deal Digest company alumni |
| D | Composite (Tuor shape) |
| E | Take-It-Slow-Before-Going-Fast |
| F | Slope Over Y-Intercept |
| company-anchored pass | Deal Digest company |
| wildcard | the wildcard signal in plain words, e.g. "moonlighter", "young infiltrator" |

The letter still goes in the store `recipe` column and the audit log — the quarterly back-test reads
it there. It just never reaches a Tom-facing surface.

**NEVER print the PF-id on this surface** (Tom, 2026-08-10: "Don't need PF number. Just tell me the
reason in plain English"). Same rule as the W1/W2/W3 ban on card Timing bullets: rule ids are
internal machinery and mean nothing to him at a glance. Write the disqualifying fact itself —
"already a founder; runs his own seed-stage company, $7.5M raised", not "PF-10". The rule id stays
in the local audit log and the store row, where the back-test reads it.

- **Header is exactly** `-1 Sourcing Summary – Week of {run_date}` (en dash) — not "neg1 sourcing".
- **Deep Sweep section (first Mondays only):** on the first Monday of the month, Step 1.75 also runs and produces `source='network-deep-sweep'` (WHAT-lens deep sweep) and `source='departure-trigger'` (departure diff) rows. List **every such row from THIS run** under **Deep Sweep ({N})**, N = the count, using the Warm row format (they upsert as `type="Warm ☀️"` — include `timing_signal` when present). On every other week no monthly rows exist — **omit the entire section** (header and all), never render an empty `Deep Sweep (0)`. Rationale: the weekly digest must be the single complete index of everyone sourced this week — deep-sweep candidates get individual `#neg1-sourcing` cards too, but without this section they never appear in the roundup and are easy to miss (they were silently absent from the 2026-08-03 summary — 5 rows).
- **Never render "Unknown"**: when `timing_signal` (or any bracket segment) is Unknown, omit that segment — `[Scaled · Unknown]` → `[Scaled]`.
- **No footer line.** The "Rows upserted → Pending Enrichment / picks up tonight" closer is dropped (it was legacy wording anyway — v2 enrichment cards within minutes). The digest ends after the last candidate row (failure warning below is the only exception).
- **Real names everywhere, never LinkedIn slugs**: warm rows use the cache `name`; cold rows use the `full_name` captured from the Step 1.5 ContactOut probe (every surviving cold candidate has one — a nameless probe is a discard).
- **Proper-noun capitalization on every row**: title-case roles and companies ("Founding Engineer @ Extend", not "founding engineer @ Extend"). No lowercase search-keyword strings.

**Section labels mirror the Notion `Type` options exactly** — `Warm` (not "Reconnect"), `Cold` (not "Cold Outreach"). Calibrated labeling everywhere.

If some rows failed to write, append:
```
⚠️ {N} row(s) failed — see audit log.
```

Prefilter kills (Step 1.6) get no individual card; they appear as one-line entries in the digest's "Filtered out" section (audit log carries full detail).

Do **not** append a "Type options pending" warning — Warm ☀️ / Cold 🧊 are the canonical options; if a row fails to write with one of those values, treat it as a real failure and bump the failed counter.

---

## Step 5 — Audit log

Append a one-line entry to `~/.claude/scheduled-tasks/neg1-sourcing/audit-log/{YYYY-MM-DD}.log`:

```
[{timestamp}] run_date={date} reconnect={N} cold={N} written={N} failed={N}
```
