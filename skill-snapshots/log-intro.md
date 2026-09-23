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
If no Opportunity entry exists, nothing is written — the report confirms the person's existing
People row and notes there is no Opp. No new Opportunity or People row is ever created by this
skill without Tom's explicit per-person approval.

## People DB Guardrails (MANDATORY)

Canonical rules and incident: `shared-references/people-db-guardrails.md` – read it before any People DB lookup or write. It overrides anything else in this skill. The People DB syncs both ways with Tom's iPhone Contacts, so a wrong write here lands on his phone.

1. **Never create a People entry — text Tom and wait for his 👍.** If a person isn't found after BOTH the scoped People DB search and the workspace search, run `python3 ~/.claude/skills/shared-references/people_db_ask.py --name "<Name>" --source-skill <this skill> [--email] [--li] [--company] [--opp-id --opp-name --relation] --context "<why>"` — it texts Tom "🧍 People DB: <Name> … ⚠ Not in the People DB yet … 👍 to add to People DB" and stages the payload (idempotent: re-runs never double-text). Tom's 👍 makes sms-listener §4b create the row via add-to-contacts and finish the skipped relation write. Until then, skip every Notion write for that person, in every mode (manual, scheduled, webhook). In reports, list them as "🧍 texted for 👍: <Name>".
2. **Never modify contact fields on an existing People page** (Email, Name, Company, Role, LI, phone) unless Tom explicitly asks. This skill writes only Opportunity-side relation fields. If a recipient's email doesn't match the People page they resolved to, flag the mismatch – never copy the email over.
3. **Match on identity, not proximity.** Resolve a person by exact email, exact name + company, or exact LinkedIn URL. Never infer a person from a shared Opportunity relation (e.g. the Opp's Qualified roster), first name alone, or the closest fuzzy search hit.
4. **Ambiguous → flag, don't guess.** Multiple candidates or conflicting keys → flag with the candidates and skip all writes for that person.


## Why this exists

This is the **manual, direct** intro logging path. Tom says who to intro and to which
company; this skill resolves both in Notion and writes the relation. The `intro-agent`
skill handles scheduled inbox scanning; this skill handles explicit real-time commands.

## The Notion Data Model

- **Opportunities DB:** `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
- **People DB:** `collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`
- **Key relation field on Opportunity:** `👓 Intros (Qualified)` — a relation to the People DB

---

**Canonical lifecycle rules:** `shared-references/intro-lifecycle-contract.md` — on any conflict, the contract wins. The inline gates/rules in this file remain in force as defense-in-depth.

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

Fetch the full Opportunity page and read:
- `Status`
- `👓 Intros (Qualified)`
- `☎️ Intros (Outreach)`
- `✉️ Intros (Made)`
- `🚫 Intros (Declined / NR)`

This is needed for the terminal-status guard (Step 2.5), duplicate detection (Step 4), and the append-safe write (Step 5).

### Step 2.5: Terminal-Status Guard (MANDATORY)

If the Opp's `Status` ∈ `{Pass (DNM), Pass (Met), Pass Note Pending, Lost, NR / Missed, Exited}`, SKIP the write and report back to Tom: `⚠️ [Opp Name] is in terminal status [status] — not a live deal. Confirm you still want to log [Person Name] here, and I'll proceed.` Tom can override by re-running with explicit confirmation; otherwise stop without writing.

Rationale: Tom occasionally names a closed Opp by accident (homonym, stale recall, or genuine intent to add to a revived deal). Stopping to confirm is cheaper than silently polluting closed Opps.

(Directionality is N/A here — log-intro is manual and Tom is always the introducer by definition. Common-word disambiguation is also N/A — Tom names the Opp explicitly.)

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

**If the person exists:** Note their page ID — but only if the match is on identity (People DB
Guardrails rule 3): exact full name + a Company that matches the company Tom named / the person's
known employer, or an exact email / LinkedIn URL Tom supplied. A name-only hit whose Company doesn't
line up, or two candidates, is ambiguous → flag with the candidates (Name · Company · Email · link)
and stop for that person. Do not touch any field on the People page.

**If the person does NOT exist (after both passes):** Do NOT create the entry. Text Tom and stop
for that person:

```
python3 ~/.claude/skills/shared-references/people_db_ask.py --name "<Name>" --source-skill log-intro \
  [--email ..] [--li ..] [--company ..] --opp-id <opp id> --opp-name "<Opp>" \
  --relation "👓 Intros (Qualified)" --context "intro to <Opp>"
```

Tom's 👍 on that text makes sms-listener §4b create the row via `add-to-contacts` and append it to
the Opp's `👓 Intros (Qualified)` — this skill writes nothing more for that person. Report
"🧍 texted for 👍: <Name>". (If Tom instead says "add them" in this same session, that is explicit
approval: run `add-to-contacts`, then continue from Step 4.)

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

No Notion write at all — this skill never edits the person's People page. Note in the summary
that there is no Opportunity entry for this company, so nothing was logged beyond confirming the
person's People row (or, if they aren't in the People DB, the 👍 text from Step 3).

### Step 6: Report Back

**Opportunity path:**
```
✅ Logged intro — [Opportunity Name] ([Fund]):
- [Person Name] ([Company], [Role]) — [existing entry linked / 🧍 texted for 👍]
  Opportunity: [Notion URL]
  Person: [Notion URL]
```

**No Opportunity path:**
```
✅ Logged intro (no Opportunity entry for [Company]):
- [Person Name] ([Company], [Role]) — existing People entry (no Opp to link)
  Person: [Notion URL]
```

**Duplicate skip:**
```
⚠️ [Person Name] already in [field name] for [Opportunity]. No change made.
```

---

## Key Rules

- **Never create an Opportunity.** If one doesn't exist, report it and write nothing.
- **Fund disambiguation:** When multiple Opportunity entries exist for the same company,
  use the original/earliest one (lower fund number, earlier Close Date) unless Tom says
  otherwise.
- **People DB search fallback:** Always try workspace search if the scoped DB search returns
  no match — the semantic index can miss recent or older entries.
- **Never overwrite existing Qualified entries** — always read current state and append.
- **No confirmation step for the relation write** — act immediately on Tom's explicit
  instruction once the person resolves cleanly. The one stop is a person missing from the
  People DB (text Tom for a 👍) or an ambiguous match (flag), per People DB Guardrails.
- **Never create People rows** — a missing person gets the `people_db_ask.py` text; creation
  happens only on Tom's 👍 via sms-listener §4b → `add-to-contacts`.
- **Never edit People page fields** (Email, Name, Company, Role, LI, phone) — the only write
  this skill makes is the Opportunity's `👓 Intros (Qualified)` relation.
