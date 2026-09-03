---
name: writeback-review-triage
description: "Pre-processes a People-DB writeback review file (from the quarterly network refresh) into a ready-to-apply decision table. For each WRONG_LI entry it climbs the enrichment ladder to find the person's correct LinkedIn URL; for each EX_INVESTOR entry it re-judges the category call against the fresh cached profile text and Tom's Category conventions. Writes a deterministic proposed-actions JSON next to the review file and posts ONE decision-table alert to #claude-alerts; Tom applies with 👍 (via claude-alerts-listener) or replies with per-row overrides. NEVER applies changes itself. Job mode (claude-job-queue, args {mode: headless, review_path}) is the primary path — enqueued by network-quarterly-refresh/run.sh when a run produces a non-empty review file. Manual trigger: \"triage the writeback review\", \"process the review queue\", \"pre-process the writeback review file\"."
---

# Writeback Review Triage

Turns a raw `people-db-writeback-reviews/<ts>.md` file into one decision
table Tom can approve with a single 👍. Born from the 2026-09-02 session
where the 23-entry queue was resolved this way (see memory
`feedback_alert_triage_self_healing_loop`): Tom's time goes to judgment
calls only — never to LinkedIn lookups.

**This skill proposes; it never applies.** Application happens in
`claude-alerts-listener` (branch: Writeback review apply) after Tom's 👍.

## Args

```json
{"mode": "headless", "review_path": "/Users/tomseo/.claude/scripts/people-db-writeback-reviews/<ts>.md"}
```

Manual mode: no args — use the newest file in
`~/.claude/scripts/people-db-writeback-reviews/`.

**Idempotency guard (first check, before any work):** if a sibling
`<review>-applied.json` exists, this queue is already done — print
"already applied" and exit without posting anything.

## Step 1 — Parse the review file

Each table row: `| Name | Verdict | Stored | Current | Suggested Cat | Confidence | Reason |`.
Split by Verdict: `WRONG_LI` rows vs `EX_INVESTOR` rows (any other verdict:
carry into the table with proposal `none` and a one-line note).

Look up each person's Notion page id + email up front:

```bash
sqlite3 ~/.claude/scripts/network_cache.db \
  "SELECT name, notion_url, email FROM profiles WHERE name = '<Name>'"
```

(Escape single quotes by doubling. If no row, try `LIKE '%<Lastname>%'`.)

## Step 2 — WRONG_LI: find the correct URL (ladder)

For each WRONG_LI person, climb in order, stop when confident:

1. **ContactOut email → LinkedIn** (`mcp__contactout__contactout_email_to_linkedin`,
   load via ToolSearch) using their People-DB email.
2. **ContactOut people search** (`contactout_search_people` — has NO name
   field; search by company/title, filter results for the name yourself).
3. **Web search**: `"<Name>" <firm> LinkedIn` and variants.
4. Verify any auto-resolve candidate noted in the Reason column rather than
   trusting it (the pipeline's `found_no_match` has been wrong before).

**Guardrails — wrong-person contamination is the disease being cured:**
- A URL counts only with POSITIVE evidence tying it to this person: employer
  or role in the result matches their stored firm / plausible successor role,
  or the email domain matches a firm in the profile's history. A bare name
  match is NEVER enough.
- Matching numeric slug suffix to a previously-known URL = strong evidence.
- Ambiguous → propose `clear_url`. Better no URL than a wrong one.
- If the STORED company/role is itself contamination (scraped from the wrong
  profile), say so and propose the correction the email domain supports
  (2026-09-02 precedent: Jodi Joseph, jodi@primary.vc → Company "Primary
  Venture Partners", URL cleared).

## Step 3 — EX_INVESTOR: re-judge against fresh evidence

Pull the fresh profile text:

```bash
sqlite3 ~/.claude/scripts/network_cache.db \
  "SELECT substr(replace(raw_text, char(10), ' | '), 1, 600) FROM profiles WHERE name = '<Name>'"
```

Apply Tom's Category conventions (memory `reference_people_and_companies_db`):
- VC **associates count as Investor** — headline "Investor @ firm" beats title
  seniority.
- Staff on investing teams at PE / asset managers (e.g. Blackstone
  Innovations Investments) stay **Investor**.
- Genuine moves to **RIA / wealth management** (Evoke Advisors, IEQ Capital
  class) → **Other**.
- Deceased → propose **archive** (Tom's 2026-09-02 precedent), phrase it
  respectfully, confidence medium.

The writeback's judge over-flags non-partner titles — expect to REJECT a
majority of its EX_INVESTOR verdicts. Rejecting = propose "keep Investor"
(plus a Company update if the firm changed).

## Step 4 — Write the proposed-actions file

Path: review file with `.md` → `-proposed.json`. Array, one object per row,
payloads must be directly executable by the listener (no re-derivation):

```json
{
  "name": "Jennifer Lee",
  "page_id": "0195c38c39c94975b1fe90216ca18784",
  "action": "patch",            // "patch" | "trash" | "none"
  "properties": {"LI": {"url": "https://www.linkedin.com/in/jennifer-lee-2a5b9716"}},
  "display": "URL → /in/jennifer-lee-2a5b9716",
  "basis": "C10 Partners MP + Edison tenure matching jlee@edisonpartners.com",
  "confidence": "high"
}
```

Property shapes (must match `people_db_writeback.py`): `LI` `{"url": ...}`
(null to clear), `Company`/`Role` rich_text, `Category`
`{"select": {"name": ...}}`.

## Step 5 — Post ONE decision-table alert

Via `send-alert` skill conventions (`~/.claude/skills/send-alert/send.sh`):

```
🧹 Writeback Review Triage — <N> proposals ready

*URL fixes (<n>):*
• <Name> → `<slug>` — <basis, few words>
*Category / firm (<n>):*
• <Name>: <display> — <basis>
*Cleared / no action (<n>):*
• <Name> — <why>

👍 applies all recommendations. Or reply with overrides, e.g. "skip Jennifer Lee", "make Palmer Investor".
proposals: `<absolute path to -proposed.json>`
```

The final `proposals:` line is load-bearing — the listener reads the path
from it. Then exit. No application, no Notion writes, no second alert.
