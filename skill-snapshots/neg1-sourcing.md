---
name: neg1-sourcing
description: Weekly Monday sourcing sweep — surfaces 2 reconnect (in-network) + 3 cold outreach candidates from Tier 1-3 companies matching target role criteria, logs each to -1 Scanner as Pending Enrichment, sends Slack digest.
triggers:
  - /neg1-sourcing
---

# neg1-sourcing — Weekly Pre-Founder Sourcing Sweep

Runs every Monday at 08:00 ET. Produces 2 reconnect + 3 cold outreach candidates from Tier 1–3 high-growth companies, writes them to -1 Scanner, and sends a Slack digest. neg1-enricher picks them up automatically via the `Pending Enrichment` queue.

**Unattended execution guard:** never ask questions, never halt waiting for input. If a step fails, skip it, log the error, and continue. Always reach the Slack alert even if some rows failed to write.

---

## ⚠️ One-Time Setup (run once before first scheduled execution)

The -1 Scanner database requires one new select property added manually in Notion before this skill can write to it:

| Property | Type | Options |
|---|---|---|
| **Type** | Select | Reconnect 👋 (purple), Cold ☎️ (orange) |

Add it in Notion → -1 Scanner → ... → Edit database → + New property.

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
  "source":             "Reconnect"         // Reconnect | Cold Outreach
}
```

If the script exits non-zero or returns 0 candidates total, skip to Step 4 (Slack alert) and report the failure.

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
| Name | `{name}` if available, else `(cold) {company} – {role}` | title property |
| Status | `Pending Enrichment` | select |
| LI | `linkedin_url` | url |
| Role | `role` | rich_text |
| Function | `function` | select — Engineering / Product / GTM |
| City | `city` if non-empty | single select |
| CurrentCo (CC) | relation to `company_notion_url` | omit if digest:// |
| Type | `type` | select — Reconnect 👋 / Cold ☎️ |
| Email | `email` if non-empty | email |

**Do not populate:** Growth Tier, Timing Signal, Claude Rec, Eval Summary, Online Presence, Work History, School(s), Experience Summary, LI Profile Summary, or any other field — neg1-enricher handles evaluation; growth/timing context flows in via CC Momentum.

Write each row individually. If a row fails, log the error and continue to the next.

**After writing rows — chain to enrichment:**
- **Manual / interactive mode**: immediately invoke `neg1-enricher` on each successfully written row (pass the LinkedIn URL). Do not wait for confirmation. Enrichment runs sequentially per row.
- **Scheduled mode** (unattended): skip the chain — pipeline-agent Task 6 picks up `Pending Enrichment` rows on its evening run. Do not block the scheduled run on enrichment.

---

## Step 4 — Slack digest

Invoke the `send-alert` skill with the following message. Replace placeholders with actuals.

Format:
```
📡 *neg1 sourcing — {run_date}*

*Reconnect (2)*
• {name} — {role} @ {company} [{growth_tier} · {timing_signal}]
• {name} — {role} @ {company} [{growth_tier} · {timing_signal}]

*Cold Outreach (3)*
• {linkedin_url} — {role} @ {company} [{growth_tier}]
• {linkedin_url} — {role} @ {company} [{growth_tier}]
• {linkedin_url} — {role} @ {company} [{growth_tier}]

Rows written to -1 Scanner → Pending Enrichment. neg1-enricher picks up tonight.
```

If some rows failed to write, append:
```
⚠️ {N} row(s) failed — see audit log.
```

---

## Step 5 — Audit log

Append a one-line entry to `~/.claude/scheduled-tasks/neg1-sourcing/audit-log/{YYYY-MM-DD}.log`:

```
[{timestamp}] run_date={date} reconnect={N} cold={N} written={N} failed={N}
```
