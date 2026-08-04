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

Runs every Monday at 08:00 ET. Produces 2 reconnect + 8 cold outreach candidates (2 wildcards) from Tier 1–3 high-growth companies, writes them to -1 Scanner, and sends a Slack digest. neg1-enricher picks them up automatically via the `Pending Enrichment` queue.

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
Cards post to `#neg1-sourcing` within minutes as each job completes. This applies to the weekly sweep AND the Step 1.75 monthly deep sweep / departure diff. pipeline-agent Task 6 remains the daily reconciliation backstop for `state=pending` rows older than ~2 hours (missed/failed jobs). The Step 4 Slack digest is unchanged. Everything referencing -1 Scanner writes below is LEGACY.

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

## Step 1.6 — Prefilter screen (Tom-taught hard disqualifiers)

Read `~/.claude/skills/founder-taste/PREFILTERS.md` and screen every surviving candidate (warm AND
cold) against each rule checkable from the data in hand (role, company, cache tenure — arc-shape
rules wait for enrichment). The script already mirrors PF-1/PF-3 in code (EXCLUDE_ROLES, fund-name
heuristic, STALE_UNICORNS); this pass catches what code can't express — judgment calls like "this
cached role reads as an investor seat" or "this employer is past its window per the momentum field".

On a kill: drop the candidate, do NOT upsert or enqueue, and log `[PREFILTERED] {name/url} — {rule id}: {one clause}`
to the audit log ONLY. Prefiltered candidates get NO card and NO mention in the Slack digest or any
alert (Tom, 2026-07-20: "prefiltered folks shouldn't be revealed in alert") — the audit log is the
sole record of the kill.

---

## Step 2 — Resolve company Notion relations

For each candidate, look up the company in the Companies DB to get the Notion page URL for the relation field:
- Use `company_notion_url` from the JSON — this is already the Notion page URL.
- If it starts with `digest://`, the company is a digest-only synthetic row with no real Notion page — omit the Company relation for that row, just set the company name as a text note in Role.

---

## Step 3 — Create -1 Scanner rows

For each candidate, create a new row in the -1 Scanner database (`32c00bef-f4aa-80a5-923b-000b83921fa3`) using `notion-create-pages`.

**Before creating:** dedup check — search the database by `LI` url. If a row with that LinkedIn URL already exists, skip creation and note it in the audit log.

**Field mapping:**

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

Write each row individually. If a row fails, log the error and continue to the next.

**After writing rows — chain to enrichment:**
- **Manual / interactive mode**: immediately invoke `neg1-enricher` on each successfully written row (pass the LinkedIn URL). Do not wait for confirmation. Enrichment runs sequentially per row.
- **Scheduled mode** (unattended): skip the chain — pipeline-agent Task 6 picks up `Pending Enrichment` rows on its evening run. Do not block the scheduled run on enrichment.

---

## Step 1.5b — Cold-pass cohort recipes (archetype feeders)

6 of the 8 cold slots draw from a WEEKLY ROTATING archetype recipe (RUBRIC.md §6 sourcing strategies) instead of generic role × tier search. **Implemented in `neg1_sourcing.py` since 2026-07-27** (`_recipe_pass` + `RECIPE_ROTATION`; before this the table was prose-only and every run shipped generic role × tier rows). Rotation by ISO week number mod 6; the run JSON carries `recipe` at top level and per-candidate, and the store upsert writes it to the `recipe` column (added same day) so the ledger can back-test recipe conversion like wildcard conversion:

| Week | Recipe | Query shape (mechanism) |
|---|---|---|
| A | **FDDM** (Field-Derived Domain Mastery) | implementation / CS / solutions operators at category-defining vertical cos — curated `FDDM_ANCHORS` list in code (the cache's category field can't express "vertical winner"; tier 1 also holds mature enterprises) (keyword role × company) |
| B | **TCDM** (Technical-Commercial Dual Mastery) | FDE / solutions engineer / applied engineer at growth-stage B2B cos from the cache, tiers 1–2 — hired technical, pulled customer-facing (keyword role × company) |
| C | **Hypergrowth alumni** | early employees of best-in-class companies, now senior — curated `HYPERGROWTH_ANCHORS` ≈ Deal Digest best tier, sync when the digest re-tiers (neural per-company query) |
| D | **Composite markers** | visible demotion-to-switch-disciplines / commercial→technical reinventors (neural shape queries) |
| E | **Take-It-Slow** (added 2026-07-27, Tom-gated) | high-reps operator in a visible patient exploration gap — advising/researching post-departure, no accelerator (neural shape queries) |
| F | **Slope over y-intercept** (added 2026-07-27, Tom-gated) | elite-school young analyst who jumped from banking/PE/consulting into an unglamorous operating domain, ramping fast (neural shape queries; distinct from the young-infiltrator wildcard, which requires a public artifact) |

If the recipe pass fills fewer than its slots, the generic role × tier pass backfills the remainder (backfilled rows carry no `recipe` value).

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
```

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

Prefilter kills (Step 1.6) are NEVER mentioned in the digest — audit log only.

Do **not** append a "Type options pending" warning — Warm ☀️ / Cold 🧊 are the canonical options; if a row fails to write with one of those values, treat it as a real failure and bump the failed counter.

---

## Step 5 — Audit log

Append a one-line entry to `~/.claude/scheduled-tasks/neg1-sourcing/audit-log/{YYYY-MM-DD}.log`:

```
[{timestamp}] run_date={date} reconnect={N} cold={N} written={N} failed={N}
```
