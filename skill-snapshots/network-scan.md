---
name: network-scan
description: Query the People DB and cached LinkedIn network to surface relevant contacts by expertise, background, or relationship — to facilitate intros, find angels, identify advisors, or answer open-ended network questions.
triggers:
  - /network
  - who in my network
  - anyone in my network
  - people in my network
  - find potential angels
  - find angels for
  - should I reconnect
  - catch up with
  - network contacts
  - who should I reach out to
---

# /network — Network Intelligence Query

Query Tom's cached LinkedIn profile database to answer questions about his network. Surfaces relevant people with ranked reasoning.

## When to trigger

- Explicit: `/network <query>`
- Natural language: "who in my network has fintech experience", "find potential angels for Tuor", "who should I reconnect with given my ideal founder profile", "is there anyone I know who's worked at Stripe"

---

## Step 1 — Extract search intent and route

Parse the query to identify:
- **Target type**: angels, operators, advisors, founders, specific expertise, reconnect candidates, etc.
- **Context**: if a portfolio company is mentioned, note its sector and stage
- **Natural language intent**: rephrase as a full descriptive sentence for semantic search

**Then pick a primitive based on whether the discriminating signal is on the COMPANY or the PERSON:**

- **Company-trait queries** → `csearch` (company-side). Use when the filter is about the company someone worked at: revenue trajectory, sector, stage, funding, momentum, "worked at a 10x'er / hypergrowth / specific company", "operators from fintech infra", "anyone who's been at Stripe / Plaid / Ramp". The `csearch` primitive vector-searches the Companies DB, joins the pre-resolved `profile_companies` table, and returns each match with full tenure dates already extracted.

  **For revenue-trajectory queries** ("10x", "hypergrowth", "company that scaled ARR fast"): run `csearch` for recall, then intersect with the Deal Digest cache for factual revenue grounding — this is the default pipeline, not an optional step. See Step 2a below for the exact sequence.

- **Person-trait queries** → `vsearch` (person-side). Use when the filter lives in the person's bio/headline: expertise, role pattern, founder archetype, angel activity, "deep technical founders", "biotech investors", "operators with consumer marketing chops".

When in doubt, run both and merge; they hit different parts of the index.

---

## Step 2a — Company-side search (preferred for company-trait queries)

```bash
python3 ~/.claude/scripts/network_cache.py csearch "<natural language query>" --no-vc --per-company 8
```

Returns JSON: array of company groups, each shaped as
`{notion_url, name, category, distance, momentum, overview, total_funding, employee_count, people: [{person, url, role, employer, dates, is_current}]}`.

Each `people[].dates` is the verbatim tenure tail extracted from the LinkedIn profile (e.g. `Jan 2020 - Jan 2021 • 1 year`). No further raw-text reads needed — Step 4's reasoning works directly on this output.

Flags:
- `--no-vc` — drop VC firms from company hits (recommended for operator queries).
- `--companies N` — pre-filter pool size from company vsearch (default 40).
- `--per-company N` — cap people per company in output (0 = no cap).
- `--max-distance D` — embedding distance gate (default 1.06).

**Revenue-trajectory queries — hybrid pipeline (default for "10x", "hypergrowth", "scaled ARR fast"):**

Run these three steps in sequence; the whole pipeline completes in ~5–10 seconds:

1. `csearch` for recall — semantic search over Companies DB + profile_companies join (broad net, may include VC firms without `--no-vc` context):
```bash
python3 ~/.claude/scripts/network_cache.py csearch "<revenue/growth query>" --no-vc --per-company 8
```

2. Deal Digest intersection — ground the csearch hits against factual revenue trajectories. Load the cache and filter to companies with documented ≥5x ARR growth:
```bash
python3 -c "
import json, re
cache = json.load(open('/Users/tomseo/.claude/skills/shared-references/deal-digest-cache.json'))
# Print companies with explicit ARR multiples or hypergrowth signals
for c, entries in cache.items():
    text = ' '.join(e.get('raw','') for e in entries)
    if re.search(r'\b([5-9]\d*x|10x|[1-9]\d{1,}x|\\\$\d+M.*ARR.*\\\$\d+[BM]|\\\$\d+[BM].*ARR)', text, re.I):
        print(c)
" 2>/dev/null
```

3. Intersect: keep only csearch hits whose company name appears in the deal-digest filter output. Annotate each person with the deal-digest revenue data for that company.

Each match's `momentum` field in csearch output also includes recent digest mentions inline — use that as a quick signal before loading the full cache.

## Step 2b — Person-side search (preferred for person-trait queries)

```bash
python3 ~/.claude/scripts/network_cache.py vsearch "<natural language query>" --limit 40
```

Returns JSON: array of `{linkedin_url, name, distance, context_blob, raw_text_preview, companies}`. Lower `distance` = more semantically similar (good results are typically under 1.05).

If vsearch returns <5 results or you need to find someone by exact company/name, supplement with FTS:

```bash
python3 ~/.claude/scripts/network_cache.py query "<keywords>" --limit 20
```

---

## Step 3 — Check cache stats (first run only)

If Step 2 returns 0 results, run:

```bash
python3 ~/.claude/scripts/network_cache.py stats
```

If the cache is empty, tell Tom:
> "Cache is empty — run the enrichment job first: `/network enrich`. Then export LinkedIn connections at linkedin.com/settings/data-privacy/export-data and run: `python3 ~/.claude/scripts/network_cache.py enrich-file ~/path/to/Connections.csv`"

---

## Step 4 — Reason over results

For `vsearch` results: pass the `raw_text_preview` (or fetch full `raw_text` from the cache via `dump <url>` for borderline cases) and reason against the query. Score relevance 1–10; keep ≥6.

For `csearch` results: each company group already carries everything you need — distance, momentum, overview, and per-person tenure tails. Filter people by tenure (was the person there during the relevant window?) and role fit (operator vs. board observer vs. investor).

The person-side `vsearch` results also carry a `companies` array (resolved via `profile_companies`) with stage, category, funding, momentum, overview. A non-empty `companies` is a strong signal of domain fit or shared context with Tom's portfolio.

**Angel investor ask** — look for: prior exits, angel activity mentioned in bio, operator at a company relevant to the portfolio company's space, strong network.

**Founder framework ask** — apply Tom's framework heuristics: hypergrowth operator experience, domain depth, founder archetype signals (missionary, domain expert, etc). Read `/Users/tomseo/.claude/skills/founder-taste/RUBRIC.md` if you need the full rubric.

**Reconnect / catch up ask** — surface people with strong profiles Tom may not have engaged with recently. Lean toward operators and founders, not investors, unless Tom specifies.

**Expertise ask** — direct match on role history and company type.

---

## Step 5 — Return ranked output

Return a clean shortlist, no more than 10 people. Format:

```
**[Name]** — [Current role / company]
[1-sentence why, specific to Tom's query]
[linkedin_url]

**[Name]** — ...
```

If fewer than 3 results: say so and suggest broadening the search or running enrichment on more profiles.

End with:
> Cache covers {N} profiles. To expand: export LinkedIn connections CSV and run `python3 ~/.claude/scripts/network_cache.py enrich-file ~/path/to/Connections.csv`.

---

## Enrichment commands (when Tom runs /network enrich or asks to populate the cache)

### From People DB (Notion)

Query People DB (`1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`) for all rows where the `LI` property is populated. Extract each LinkedIn URL and write to a temp file, then enrich:

```bash
python3 ~/.claude/scripts/network_cache.py enrich-file /tmp/people_db_urls.txt
```

### From LinkedIn connections CSV

Tom exports from: linkedin.com → Me → Settings & Privacy → Data Privacy → Get a copy of your data → Connections. The export CSV has a `Profile URL` column.

```bash
python3 ~/.claude/scripts/network_cache.py enrich-file ~/path/to/Connections.csv
```

### From ContactOut cache (free, no Exa credits)

```bash
python3 ~/.claude/scripts/network_cache.py enrich-contactout
```

Run this first — it's instant and imports anyone already enriched by neg1-enricher.

### Link profiles to company cache

After enriching, run this to build/refresh the person→company join table:

```bash
python3 ~/.claude/scripts/network_cache.py resolve-companies --missing-only
```

This matches each profile's employers (via LinkedIn company URL slugs) against Tom's Companies DB in SQLite. Results appear as `portfolio_companies` in every `query` / `vsearch` result.

### Check coverage

```bash
python3 ~/.claude/scripts/network_cache.py stats
```

---

## EXA_API_KEY setup

The script reads `EXA_API_KEY` from environment or `~/.claude/scripts/network.env`. If the key isn't set, write it:

```bash
echo "EXA_API_KEY=your_key_here" >> ~/.claude/scripts/network.env
```
