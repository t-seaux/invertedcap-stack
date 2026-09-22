---
name: writeback-review-triage
description: "Pre-processes a People-DB writeback review file (from the MONTHLY full network sweep — or a legacy quarterly file) into a ready-to-apply decision table. Two input formats, auto-detected: (A) legacy verdict format (WRONG_LI → enrichment-ladder URL fix; EX_INVESTOR → category re-judge); (B) held-drift format from network_cache cmd_writeback (`| Status | Name | Notion has | Experience shows | Headline says |`) → adjudicate each row with Tom's headline-first rules and propose apply / protect / leave-held. Writes a deterministic proposed-actions JSON next to the review file and posts ONE decision-table alert to #claude-alerts; Tom applies with 👍 (via claude-alerts-listener) or replies with per-row overrides. NEVER applies changes itself. Job mode (claude-job-queue, args {mode: headless, review_path}) is the primary path — enqueued by network-sync-notion/run.sh when a monthly sweep produces a non-empty held file. Manual trigger: \"triage the writeback review\", \"process the review queue\", \"pre-process the writeback review file\"."
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

## Step 1 — Parse the review file (detect the format first)

Two formats, auto-detected from the file's first lines:

- **Held-drift format** (the monthly path) — title line `# Writeback — inferred
  status, held for review`, rows
  `| Status | Name | Notion has | Experience shows | Headline says |`.
  → Skip Steps 2–3, use **Step 3b**.
- **Legacy verdict format** — rows
  `| Name | Verdict | Stored | Current | Suggested Cat | Confidence | Reason |`.
  Split by Verdict: `WRONG_LI` rows vs `EX_INVESTOR` rows (any other verdict:
  carry into the table with proposal `none` and a one-line note). → Steps 2–3.

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

## Step 3b — Held-drift format: adjudicate with Tom's headline-first rules

For each held row, pull the fresh profile (headline + current experience
blocks) and decide:

```bash
sqlite3 ~/.claude/scripts/network_cache.db \
  "SELECT substr(replace(raw_text, char(10), ' | '), 1, 900), linkedin_url, category FROM profiles WHERE name = '<Name>'"
```

**Tom's decision hierarchy (his own formulation, 2026-09-20 — this IS the
method; everything below is elaboration):**

> 1. Company explicitly mentioned in the headline → HIGH confidence it's
>    the main thing.
> 2. No company in the headline → look at experience (the first current
>    entry is the user-designated primary).
> 3. Self-employment is inferred from the experience blocks: every current
>    entry an advisor / board / self-employed seat → they work for
>    themselves (`Self`).

Elaboration and guards:
- **Tier 1:** headline naming the Notion value → keep (suppressed upstream);
  naming the experience company → apply; naming a NEW company outright →
  apply. Multi-company headline → the FIRST-listed one is primary.
- **Tier 2 ("vague"):** genuinely names no employer. A headline with an at/@
  employer the parser failed to read is NOT vague — parse-failure + a third
  company in experience is the wrong-LinkedIn signature (Mary Bui/Sephora);
  flag, never book. Investors are excluded from the tier-2 default (their
  experience is board seats — the firm stays primary; Tom's rule).
- **Tier 3:** see the Self/Investor convention below (Toan Huynh is the
  contrast case: advisor-TITLED but at a real employer listed first —
  tier 2 applies, not tier 3).
- **Memberships are never employers** — YPO, Council on Foreign Relations
  term member, Milken circles, EO, angel-group membership. A Notion value
  like that is stale by definition; find the real primary.
- **Board-observer seats at a fund's portfolio companies = still at the
  fund** (Amanda Angelini precedent). "Investor / Board Member" headline +
  observer rows → keep the fund, propose `protect`.
- **Stealth / placeholder experience + build intent → NewCo**; fund word or
  Investor-category disambiguation → **New Fund**. If the HEADLINE names the
  stealth entity, use the real name instead (Sameer Kenkare → Graphon AI).
- **Full name over abbreviation** is fine (JSQ → Juniper Square); never the
  reverse.
- Category rides along ONLY when the profile gives it away (exp says
  "…Startup" → Startup; fund / "raising a fund" → Investor) or the slot is
  uncategorized. An ambiguous case keeps the existing category.
- **Independent angel investors → Company `Self`, Category `Investor`**
  (Tom's convention; Patrick Heim + Tyler Dean precedents 2026-09-20):
  someone whose current state is solo investing gets Company `Self`, never a
  stale employer, a portfolio company, or an advisory seat. Detection
  pattern (Tyler Dean): invest-intent headline naming no CURRENT firm
  ("VC, Growth Equity and LP investor | Ex. Morgan Stanley, 8VC…" — the
  Ex-list is past) + EVERY current experience entry is an advisor / board /
  self-employed seat → `Self` + `Investor`. Pair with `protect` so
  LinkedIn's advisor rows don't re-flag monthly.
- **Multi-company headline → the FIRST-listed company is the primary**
  (Aiden Lee, 2026-09-20: "Fika Ventures / Atlantic Music Group" → Fika
  Ventures, even though experience corroborated the second one).
- **A person Tom says he doesn't know → propose `trash`** (Notion archive
  is recoverable). Precedents: Benjamin Peeters, Christian Magel, Aparna
  Rae (2026-09-20). Only on Tom's explicit say-so in the thread — never
  propose trash from profile evidence alone.
- **Status `Protected — LI moved again`** — Tom manually protected this
  person against one known LinkedIn disagreement, and LinkedIn has since
  moved to a THIRD company that the engine could NOT confirm or dismiss
  (confirmed moves auto-book upstream and release the protect; board-seat/
  membership churn stays protected silently — only the unconfirmable middle
  reaches this table). His protect may be outdated. Adjudicate the new
  evidence fresh (headline-first, as above): looks like a real move →
  propose `patch` to the new company (the listener clears the old override);
  still noise (another board seat, another side thing) → propose `protect`
  again (re-setting it refreshes the snapshot, quieting it until LI moves
  next). These rows lead the alert's held section — they're the ones where
  Tom's own knowledge is at stake.

Disposition per row:
- **Confident move** → `action: "patch"` with the new Company (+Category per
  the rule above).
- **Confident keep** (board seat, side gig, portfolio-observer) →
  `action: "protect"` — keeps Notion AND stops the row re-flagging monthly.
- **Genuine coin-flip** → `action: "none"` with a one-line why; it stays in
  next month's held pile, which is correct.

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

**Held-mode extra action** — `"protect"` (keep Notion, stop monthly re-flags
via the manual-override mechanism). Shape:

```json
{
  "name": "Daisy Cai",
  "linkedin_url": "https://www.linkedin.com/in/daisycai",
  "action": "protect",
  "keep": "B Capital",
  "li_shows": "Reflection AI",
  "display": "keep B Capital (exp rows are portfolio boards)",
  "basis": "headline 'General Partner B Capital'",
  "confidence": "high"
}
```

`li_shows` = what LinkedIn currently proposes (the "Experience shows" column).
The listener records it as the protect's snapshot, so the row stays quiet
until LinkedIn moves to something ELSE — at which point it resurfaces as
`Protected — LI moved again`.

A `"patch"` entry in held mode must ALSO carry `linkedin_url` so the listener
can update the cache mirror after the Notion write (otherwise next month's
sweep re-detects the same drift).

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

Held-mode body sections (same header + closer, sections swap):

```
*Moves to apply (<n>):*
• <Name>: <old> → <new> — <basis, few words>
*Keeps to protect (<n>):*
• <Name>: keep <company> — <basis>
*Left held — coin-flips (<n>):*
• <Name> — <why>
```

The final `proposals:` line is load-bearing — the listener reads the path
from it. Then exit. No application, no Notion writes, no second alert.
