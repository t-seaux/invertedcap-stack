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
python3 ~/.claude/scripts/decision-ledger/candidates.py upsert --json '{"li_url": "...", "name": "...", "city": "...", "current_role": "...", "current_company": "...", "type": "Warm ☀️|Cold 🧊", "source": "monday-sweep", "state": "pending"}'
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
| **Warm ☀️** | orange | In-network reconnect — Tom has worked with or knows this person |
| **Cold 🧊** | blue | Cold outreach — no prior relationship |

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
  "type":               "Warm ☀️"           // Warm ☀️ | Cold 🧊
}
```

If the script exits non-zero or returns 0 candidates total, skip to Step 4 (Slack alert) and report the failure.

---

## Step 1.5 — Verify cold candidates via ContactOut

**Reconnect candidates are skipped** — they come from the LinkedIn network cache (real people Tom knows) and don't need verification.

For each **cold candidate**, call `contactout_enrich_linkedin_profile` with `profile_only: true` before writing anything to Notion. This is a cheap probe (no email credit consumed) that confirms the LinkedIn URL resolves to a real person who currently works at the target company.

**Verification criteria — both must pass:**
1. **Profile exists**: the response returns a non-empty `full_name`. An empty/null name or an API error means the slug is a ghost → discard.
2. **Company match**: the person's current employer (first `experience` entry with `is_current: true`, or the most recent entry) fuzzy-matches the candidate's `company` field. Use case-insensitive substring match — `"Morgan Stanley"` should not match `"Replit"`. If no current employer is present in the response, treat as mismatch → discard.

**On discard**: remove the candidate from the cold list and log: `[REJECTED] {url} — {reason}` (reason: "ghost URL" or "employer mismatch: expected {company}, got {actual_employer}"). Do not replace with a substitute — if fewer than 3 cold candidates survive, proceed with however many passed.

**Process cold candidates sequentially** — don't batch the ContactOut calls in parallel to avoid rate limits.

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

6 of the 8 cold slots draw from a WEEKLY ROTATING archetype recipe (RUBRIC.md §6 sourcing strategies) instead of generic role × tier search. Rotation by ISO week number mod 4:

| Week | Recipe | Query shape |
|---|---|---|
| A | **FDDM** (Field-Derived Domain Mastery) | ex-implementation / CS / solutions operators, 2y+ tenure, at category-defining vertical cos (ServiceTitan, Procore, Toast, Veeva, Samsara, and Tier-1 vertical leaders from the Companies cache) |
| B | **TCDM** (Technical-Commercial Dual Mastery) | FDE / solutions engineer / applied engineer at growth-stage B2B cos — hired technical, pulled customer-facing |
| C | **Hypergrowth alumni** | early employees (joined pre-~150 HC) of Deal Digest best-in-class-tier companies, 2y+ tenure, now senior |
| D | **Composite markers** | visible demotion-to-switch-disciplines / commercial→technical reinventors (Exa keyword search on transition language) |

**Doctrine coupling:** this table is the sourcing expression of RUBRIC.md §6 — it is NOT independently editable. When an archetype is added, revised, or retired in the rubric (human-gated), update this rotation in the same change. Recipes never drift from doctrine.

**Wildcard slots (explore vs exploit):** every week, 2 of the 8 cold slots are reserved for candidates deliberately OUTSIDE all current archetypes but carrying ONE extreme signal the rubric respects on a shape it doesn't recognize (e.g. a 10-grade spike on Non-Linearity or Earned Reps in an arc that matches no recipe). Upsert with `source="wildcard"`. Purpose: archetype discovery — the doctrine-coupled recipes can only find shapes past taste already codified. Quarterly, review wildcard conversion in the ledger (`SELECT * FROM decisions WHERE label IN (SELECT name FROM candidates WHERE source='wildcard')`); 3+ wildcard drafts sharing a shape is a new-archetype candidate for the Casebook.

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

## Step 4 — Slack digest

**Channel routing (gate in code):** if `~/.claude/skills/neg1-sourcing/.sourcing_channel_id` exists, post via `send-alert/md_to_blocks.py` in bot-token mode: `SLACK_BOT_TOKEN_FILE=$HOME/.claude/skills/claude-alerts-listener/.bot_token SLACK_CHANNEL=$(cat ~/.claude/skills/neg1-sourcing/.sourcing_channel_id) BODY_FILE=<tmpfile> python3 ~/.claude/skills/send-alert/md_to_blocks.py` (prints the message `ts` — no webhook needed) — all sourcing surfaces live in `#neg1-sourcing` (this weekly digest of raw candidates + pipeline-agent Task 6's post-enrichment Reach Out ✅ cards). If the file does not exist, fall back to the default `send-alert` channel.

Invoke the `send-alert` skill with the following message. Bodies are GFM markdown (see `send-alert/SKILL.md`) — `**bold**` becomes bold, `[label](url)` becomes a clickable link, `*single asterisks*` would render as italic so avoid them.

**Format:**
```
📡 **neg1 sourcing — {run_date}**

**Warm (2)**
• [{name}]({linkedin_url}) — {role} @ {company} [{growth_tier} · {timing_signal}]
• [{name}]({linkedin_url}) — {role} @ {company} [{growth_tier} · {timing_signal}]

**Cold (8, incl. 2 wildcards — tag those rows `[wildcard]`)**
• [{handle}]({linkedin_url}) — {role} @ {company} [{growth_tier}]
• (8 rows)

Rows written to -1 Scanner → Pending Enrichment. neg1-enricher picks up tonight.
```

**Link-label rules:**
- Warm rows: use the candidate's `name` as the link label (always populated for in-network reconnects).
- Cold rows: use the LinkedIn handle as the link label — the path segment after `/in/` (e.g. `https://www.linkedin.com/in/jaswanthmadha` → `jaswanthmadha`). `name` is empty until neg1-enricher resolves it via ContactOut.

**Section labels mirror the Notion `Type` options exactly** — `Warm` (not "Reconnect"), `Cold` (not "Cold Outreach"). Calibrated labeling everywhere.

If some rows failed to write, append:
```
⚠️ {N} row(s) failed — see audit log.
```

Do **not** append a "Type options pending" warning — Warm ☀️ / Cold 🧊 are the canonical options; if a row fails to write with one of those values, treat it as a real failure and bump the failed counter.

---

## Step 5 — Audit log

Append a one-line entry to `~/.claude/scheduled-tasks/neg1-sourcing/audit-log/{YYYY-MM-DD}.log`:

```
[{timestamp}] run_date={date} reconnect={N} cold={N} written={N} failed={N}
```
