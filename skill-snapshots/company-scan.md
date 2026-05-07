---
name: company-scan
description: >-
  Query Tom's cached Companies database to answer open-ended questions about
  companies he tracks — portfolio companies, coinvestors, competitors, diligence
  targets, funds, schools. Answers queries like "which of my tracked companies
  are doing AI for insurance", "show me Series A companies in NY with growing
  headcount", "find companies raising in the climate space", "which funds in my
  DB are based in SF". Also used via /companies slash command.
triggers:
  - /companies
  - companies in my database
  - companies I track
  - which companies
  - find companies
  - show me companies
  - portfolio companies that
  - tracked companies
---

# /companies — Company Intelligence Query

Query Tom's cached Companies DB (Notion → SQLite with Exa enrichment + embeddings)
to answer questions about the companies he tracks.

## When to trigger

- Explicit: `/companies <query>`
- Natural language: "which of my tracked companies are in fintech", "show me Series B
  companies in New York", "find funds I have in my DB that focus on climate",
  "which companies have high headcount growth", "what do I know about insurtech
  players in my database"

---

## Step 1 — Extract search intent

Parse the query to identify:
- **Company type**: Company 🛠️, Fund 💸, School 🎓, or any
- **Stage filter**: Seed, Series A, Series B, etc. (if specified)
- **HQ filter**: specific metro or region (if specified)
- **Semantic intent**: rephrase as a descriptive sentence for vector search
  (e.g. "B2B SaaS companies focused on compliance and regulatory automation for financial services")

---

## Step 2 — Search the cache

Run semantic vector search using the full natural-language intent:

```bash
python3 ~/.claude/scripts/company_cache.py vsearch "<natural language query>" --limit 40
```

Returns JSON array of `{notion_url, name, category, hq, funding_status, revenue,
total_funding, employee_count, distance, context_blob, overview_preview, momentum_preview}`.
Lower `distance` = more semantically similar (good results typically under 1.05).

If vsearch returns <5 results or the query uses exact keywords (company name, stage,
city), supplement with FTS:

```bash
python3 ~/.claude/scripts/company_cache.py query "<keywords>" --limit 20
```

---

## Step 3 — Check cache stats (first run or empty results)

```bash
python3 ~/.claude/scripts/company_cache.py stats
```

If cache is empty, tell Tom:
> "Cache is empty — run the initial sync first:
> `python3 ~/.claude/scripts/company_cache.py sync --full`
> Then enrich: `python3 ~/.claude/scripts/company_cache.py enrich --missing-only`
> Then embed: `python3 ~/.claude/scripts/company_cache.py embed --missing-only`"

---

## Step 4 — Reason over results

Pass `overview_preview`, `momentum_preview`, and `context_blob` to yourself and
reason against the query. Score each result 1–10 for relevance to the specific ask.
Keep only those ≥6.

**Sector/thesis ask** — match on Overview text and Exa-enriched content; look for
business model, customer type, technology signals.

**Stage + geography ask** — filter hard on `funding_status` and `hq`; if both
match, surface even lower-semantic-score results.

**Headcount/growth ask** — parse the `momentum_preview` and `employee_count`;
look for growth % signals in Headcount field.

**Fund/investor ask** — category = `Fund 💸`; look for sector focus in overview.

---

## Step 5 — Return ranked output

Return a clean shortlist, no more than 10 companies. Format:

```
**[Name]** — [Category] · [Funding Status] · [HQ]
[1-sentence why, specific to Tom's query]
[notion_url]

**[Name]** — ...
```

Include revenue and/or employee count when relevant to the query.

If fewer than 3 results: say so and suggest broadening or running enrichment.

End with:
> Cache covers {N} companies ({X} Exa-enriched, {Y} embedded). To refresh:
> `python3 ~/.claude/scripts/company_cache.py sync`

---

## Cache management commands

### Initial population (first time)

```bash
python3 ~/.claude/scripts/company_cache.py sync --full
python3 ~/.claude/scripts/company_cache.py enrich --missing-only
python3 ~/.claude/scripts/company_cache.py embed --missing-only
```

### Incremental refresh (runs automatically monthly on 1st at 18:14)

```bash
python3 ~/.claude/scripts/company_cache.py sync
python3 ~/.claude/scripts/company_cache.py enrich --missing-only
python3 ~/.claude/scripts/company_cache.py embed --missing-only
```

### Quarterly full refresh (runs automatically Jan/Apr/Jul/Oct 1st at 18:16)

```bash
python3 ~/.claude/scripts/company_cache.py sync --full
python3 ~/.claude/scripts/company_cache.py enrich
python3 ~/.claude/scripts/company_cache.py embed
```

### Stats

```bash
python3 ~/.claude/scripts/company_cache.py stats
```

### Inspect a company

```bash
python3 ~/.claude/scripts/company_cache.py dump <notion_url>
```
