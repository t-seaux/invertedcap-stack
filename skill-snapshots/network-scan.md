---
name: network-scan
description: Query Tom's cached LinkedIn network to answer open-ended questions — "who should I reconnect with given my founder framework", "find potential angels for [company]", "who in my network has payments experience". Also used via /network slash command.
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

## Step 1 — Extract search intent

Parse the query to identify:
- **Target type**: angels, operators, advisors, founders, specific expertise, reconnect candidates, etc.
- **Context**: if a portfolio company is mentioned, note its sector and stage
- **Natural language intent**: rephrase as a full descriptive sentence for semantic search (e.g. "people with fintech compliance and regulatory experience at banks or startups")

---

## Step 2 — Search the cache

Run a semantic vector search using the full natural-language intent:

```bash
python3 ~/.claude/scripts/network_cache.py vsearch "<natural language query>" --limit 40
```

The script returns JSON: array of `{linkedin_url, name, distance, context_blob, raw_text_preview}`. Lower `distance` = more semantically similar (good results are typically under 1.05).

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

Pass the `raw_text_full` of each candidate to yourself and reason against the query. For each person score relevance 1–10 against the specific ask. Keep only those ≥6.

Each result also carries a `portfolio_companies` array — companies the person worked at that are in Tom's tracked Companies DB. Use this for extra context: stage, category, funding, momentum, overview. A non-empty `portfolio_companies` is a strong signal of domain fit or shared context with Tom's portfolio.

**Angel investor ask** — look for: prior exits, angel activity mentioned in bio, operator at a company relevant to the portfolio company's space, strong network.

**Founder framework ask** — apply Tom's framework heuristics: hypergrowth operator experience, domain depth, founder archetype signals (missionary, domain expert, etc). Read `/Users/tomseo/.claude/skills/founder-outreach/FOUNDER_EVAL_FRAMEWORK.md` if you need the full rubric.

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
