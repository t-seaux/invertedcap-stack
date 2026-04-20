---
name: log-intro
description: >
  Manually log a specific person as a qualified intro target for a company in Notion.
  Trigger when Tom says things like "intro X to Y", "add X as qualified in Y opportunity",
  "log an intro for X to Y", "queue an intro for X to Y", "set up X as a qualified intro
  for Y", or any variant where Tom explicitly names a person and a target company together
  with the intent to log an intro. This skill handles a single, direct manual log — it is
  NOT the scheduled inbox scanner (that's the intro-agent skill). Always trigger inline
  without confirmation whenever Tom names a person + a company with intro intent, even if
  the phrasing is casual or shorthand.
---

# Log Intro (Manual)

Log a named person as a qualified intro for a named company in Notion. If the company has
an existing Opportunity entry, the person is added to its `👓 Intros (Qualified)` relation.
If no Opportunity entry exists, the person is logged in the People DB only — no new
Opportunity is ever created by this skill.

## Why this exists

This is the **manual, direct** intro logging path. Tom says who to intro and to which
company; this skill resolves both in Notion and writes the relation. The `intro-agent`
skill handles scheduled inbox scanning; this skill handles explicit real-time commands.

## The Notion Data Model

- **Opportunities DB:** `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
- **People DB:** `collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`
- **Key relation field on Opportunity:** `👓 Intros (Qualified)` — a relation to the People DB

---

## Execution Workflow

### Step 1: Check for an Existing Opportunity

Search the Opportunities DB for the named company using a **two-pass approach**:

**Pass 1 — Scoped DB search:**
```
notion-search with query = "<company name>" and data_source_url = "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"
```

**Pass 2 — Workspace search (if Pass 1 returns no exact match):**
```
notion-search with query = "<company name>", query_type = "internal", content_search_mode = "workspace_search"
```
Filter results to Opportunities DB pages only.

**Outcome A — Opportunity found:** Proceed through the full workflow (Steps 2–5).

**Outcome B — No Opportunity found:** Skip to Step 3 (People DB only). Do not create an
Opportunity entry. Note in the summary that no Opportunity entry exists for this company.

**Multiple entries for the same company:** Some companies have more than one Opportunity
entry across funds. When multiple exist, always default to the original / earliest entry —
the one with the earlier Close Date and lower fund number (e.g., Dash Fund 1 before Dash
Fund 2) — unless Tom explicitly specifies a different one.

### Step 2: Fetch the Opportunity's Current Intro State (Opportunity Path Only)

Fetch the full Opportunity page and read the current state of all four intro lifecycle fields:
- `👓 Intros (Qualified)`
- `☎️ Intros (Outreach)`
- `✉️ Intros (Made)`
- `🚫 Intros (Declined / NR)`

This is needed for both duplicate detection (Step 4) and the append-safe write (Step 5).

### Step 3: Resolve the Person in the People DB

**Pass 1 — Scoped DB search:**
```
notion-search with query = "<person name>" and data_source_url = "collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9"
```

**Pass 2 — Workspace search fallback (if Pass 1 returns no exact match):**
```
notion-search with query = "<person name>", query_type = "internal", content_search_mode = "workspace_search"
```
The workspace search will surface call logs, notes, and other pages referencing the person
even if the scoped DB search misses them. From the results, identify the People DB page
(it will be under the People DB ancestor path).

**If the person exists:** Note their page ID. Verify name + company match if multiple
candidates appear.

**If the person does NOT exist:** Follow the `add-to-contacts` skill at
`/Users/tomseo/.claude/skills/add-to-contacts/SKILL.md` to create the People DB entry. That skill is
the single source of truth for People DB creation — do not duplicate its logic here.

### Step 4: Duplicate Detection (Opportunity Path Only)

Before writing, check whether the person already appears in any of the four lifecycle fields
fetched in Step 2. If so, do NOT add them to Qualified:

- `👓 Intros (Qualified)` → skip, report "already in Qualified"
- `☎️ Intros (Outreach)` → skip, report "already in Outreach"
- `✉️ Intros (Made)` → skip, report "intro already made"
- `🚫 Intros (Declined / NR)` → skip, report "previously declined/NR"

### Step 5: Write to Notion

**Opportunity path — append to `👓 Intros (Qualified)`:**

Use `notion-update-page` with `command: "update_properties"`. The relation field expects a
**JSON array string** of Notion page URLs. Always include existing entries to avoid
overwriting them:

```
"👓 Intros (Qualified)": "[\"https://www.notion.so/existing\",\"https://www.notion.so/new\"]"
```

If the field was previously empty, pass a single-element array:
```
"👓 Intros (Qualified)": "[\"https://www.notion.so/<person-page-id>\"]"
```

**No Opportunity path — no Notion write beyond People DB:**

If the person already existed in the People DB, no write is needed at all. If they were
newly created via `add-to-contacts`, that skill handles the write. Note in the summary
that there is no Opportunity entry for this company and the intro is logged in People DB
only.

### Step 6: Report Back

**Opportunity path:**
```
✅ Logged intro — [Opportunity Name] ([Fund]):
- [Person Name] ([Company], [Role]) — [existing entry linked / new entry created]
  Opportunity: [Notion URL]
  Person: [Notion URL]
```

**No Opportunity path:**
```
✅ Logged intro (no Opportunity entry for [Company]):
- [Person Name] ([Company], [Role]) — [existing entry linked / new entry created]
  Person: [Notion URL]
```

**Duplicate skip:**
```
⚠️ [Person Name] already in [field name] for [Opportunity]. No change made.
```

---

## Key Rules

- **Never create an Opportunity.** If one doesn't exist, log the person to People DB only.
- **Fund disambiguation:** When multiple Opportunity entries exist for the same company,
  use the original/earliest one (lower fund number, earlier Close Date) unless Tom says
  otherwise.
- **People DB search fallback:** Always try workspace search if the scoped DB search returns
  no match — the semantic index can miss recent or older entries.
- **Never overwrite existing Qualified entries** — always read current state and append.
- **No confirmation step** — act immediately on Tom's explicit instruction.
- **People creation delegated** — if the person doesn't exist in People DB, use the
  `add-to-contacts` skill; do not inline that logic here.
