---
name: coop-finances
description: >-
  Maintain 25 Garden Place Corp's P&L workbook from raw bank txn data. Canonical artifact: ~/Library/Mobile
  Documents/com~apple~CloudDocs/Desktop/Garden/25_Garden_Place_PL.xlsx (iCloud-synced, edited in place via
  openpyxl). Trigger paths: (A) scheduled 1st-of-month Slack prompt for the latest Citi CHK-7926 CSV + Reserve
  screenshot; (B) webhook processing when Tom replies with attachments; (C) manual — Tom drops a CSV with
  "ingest these coop txns" or "update the coop financials". Ingests the feed, classifies txns per
  references/VENDOR_CLASSIFICATIONS.md + CHECK_LEDGER.md, applies maintenance clustering, flags unknown vendors
  / off-pattern CHECKs, saves in place, posts a Slack summary. New vendor classes confirmed in-thread get
  appended. Drive snapshot opt-in only ("push a snapshot to Drive").
---

# coop-finances

Tom is treasurer of 25 Garden Place Corp (4-unit Brooklyn co-op as of 2026-05-13). This skill maintains the canonical annual P&L workbook.

## Canonical artifact

- **Canonical (read/write)**: `~/Library/Mobile Documents/com~apple~CloudDocs/Desktop/Garden/25_Garden_Place_PL.xlsx` — iCloud-synced, edited in place via openpyxl. Tom can also open/edit in Excel.app or Numbers on any of his Apple devices; iCloud syncs both ways.
- **Drive snapshot (optional)**: file ID `1q4M2bn5GHZTiBuj4RoW8k7n474fOws9i`. **STALE as of 2026-05-20** — keep around as the original upload, or delete. New Drive snapshots only on request ("push a snapshot to Drive" or for sharing with a CPA/coowner).
- **Structure**: one sheet per year (2024, 2025, 2026…); income lines (Balance Forward, Maintenance, Assessments, Repair Settlement, Interest Income), expense lines, and a Reserve Fund section. See `reference_coop_pl_workbook.md` memory for column/row layout.

iCloud xlsx is the source of truth. Edit in place — no upload step needed. Drive is opt-in mirror for sharing.

**Two-way edit safety**: Because Tom can edit the iCloud file himself, every skill run must read the current state first and apply diffs additively. Don't overwrite cells you didn't compute. Cluster-rebuilt Maintenance rows are safe (the cluster IS the month's total). Other line items: add to existing cell value, don't replace.

## Modes

### Mode A — Scheduled (1st of month, 9 AM ET)

Wired via `~/Library/LaunchAgents/com.tomseo.scheduled.coop-finances-prompt.plist` (to be created).

Posts to `#claude-alerts` via `send-alert`:

> 📒 **Coop finances — monthly drop**
>
> Drop the latest as thread replies:
> 1. Citi CHK-7926 CSV (operating account — past month or more)
> 2. Reserve account — screenshot OR xls export
>
> I'll dedupe against the existing workbook, classify, post a flagged summary.

No processing happens here — just the prompt.

### Mode B — Webhook (Tom replies in thread with attachment)

Routed through the existing `claude-alerts-listener` infrastructure (slack-retro-webhook → claude-job-queue → processor.py). The listener detects parent-message origin (this skill's monthly prompt) and dispatches to `coop-finances` with the attachment path.

Required listener patch (one-time): pass attachment file IDs through to the job payload. Currently `claude-alerts-listener` reads reply text only.

### Mode C — Manual

Tom drops a CSV or screenshot directly in chat with phrasing like:
- "ingest these coop txns"
- "update the coop financials"
- "log these to the coop sheet"
- raw CSV upload with no preamble (auto-detect: header matches `Status,Date,Description,Debit,Credit`)

Skip the Slack-summary step; respond inline instead.

## Steps (all modes)

### 1. Read canonical artifact

Open the iCloud xlsx directly with openpyxl (path above). No download step.

### 2. Parse input

- CSV: standard Citi export shape (Status, Date, Description, Debit, Credit). Filter to `Cleared` rows.
- Reserve screenshot (PNG/JPG): OCR'd inline via vision — extract (date, type, amount, running balance) tuples. Validate against the running balance.
- Reserve xls: openpyxl read.

### 3. Dedupe against existing rows

For each input txn, check if a matching `(month, line item, amount)` already exists in the workbook. Match tolerance: amount ±$0.50, same calendar month. Skip duplicates silently (count them in summary).

### 4. Classify each new txn

Cascade:

1. **Maintenance clustering** (deposits only): see CLUSTERING below.
2. **Vendor lookup**: `references/VENDOR_CLASSIFICATIONS.md` — match Description substring → (line item, month rule).
3. **CHECK ledger lookup**: if Description matches `CHECK NNNN`, look up by check number in `references/CHECK_LEDGER.md`. If not found, classify by amount cohort per the heuristics table — only auto-post if confidence = High. Otherwise flag.
4. **Unknown** → flag for Tom.

For every successful auto-classification of an unrecognized digital-payment vendor (Tom is migrating off CHECKs — expect new strings), append a draft entry to VENDOR_CLASSIFICATIONS.md marked `[pending-confirm]` and surface in the Slack summary.

### 5. Maintenance clustering algorithm

Sort `TELLER DEPOSIT` rows chronologically. Walk through, accumulating into open cluster:

- Each $1,100 / $2,200 / $3,300 / $4,400 deposit adds to the open cluster.
- When the open cluster hits **$4,400 ± $50**, close it. Assign the closed cluster to the next unfilled Maintenance month (starting from the earliest cluster's earliest deposit's calendar month, then incrementing).
- Standalone non-$1,100-multiple round amounts ≥ $4,000 (e.g. $4,000, $6,000, $8,000, $12,000, $20,000, $45,536) → Assessment OR Repair Settlement → flag for Tom's confirmation (default Assessment).
- If a cluster doesn't close cleanly within a calendar month (e.g. an underpayment), surface in the Slack summary: "Cluster at month X totaled $Y instead of $4,400 — confirm short pay?"
- A cluster can span calendar months (e.g. late-March deposits cover April). That's expected — post by SERVICE MONTH, not deposit calendar month.

### 6. Write updates

In-place edits to the xlsx:
- Add txn amount to the existing cell (don't overwrite — accumulate). For deposits posted via clustering, the cluster total ($4,400) replaces any prior value for that month's Maintenance row, since the cluster IS that month's full income.
- TOTAL column and Subtotal rows are formulas — they recompute automatically.
- For new years, create a new sheet via the same row scaffold (copy 2026 layout, blank cells).

Save xlsx in place. iCloud handles sync. No Drive upload unless Tom explicitly asks for a snapshot.

### 6.5 Post-run verification gates (MANDATORY — never silently accept)

`update_pl.py` already computes the verification primitives; the calling agent must treat them as **hard gates**, not informational output:

- **CSV ↔ _Raw reconciliation**: the output JSON includes `csv_sum_delta_in` / `csv_sum_delta_out` (from `reconcile_csv_sums` — parse-vs-written sums). Both must be `0.00`. Any non-zero delta means a row was dropped or duplicated between parse and write — lead the Slack reply/inline response with `⚠️ RECONCILIATION FAILED: in delta $X, out delta $Y — review before trusting this run` and do not present the run as clean.
- **Cluster integrity**: any `cluster_issues` entries surface verbatim in the flagged section.
- **Reserve screenshot OCR validation**: when a Reserve screenshot (not CSV/xls) is the input, the OCR'd transcription must be validated before ingest — recompute the running balance from the extracted (date, type, amount) tuples and compare to the screenshot's final displayed balance. **Mismatch > $1 → reject the OCR pass entirely**; do not ingest. Reply: "Screenshot OCR didn't reconcile (computed $X vs displayed $Y) — please export the Reserve account as CSV/xls instead." Additionally pass the screenshot's final balance through to `check_reserve_balance` so the workbook-level check runs; surface any reported mismatch as a flag.

### 7. Slack summary (Mode A/B) or inline (Mode C)

Post to the same thread (Mode B) or `#claude-alerts` (Mode A first run after data drop):

```
📒 Coop finances updated
• N txns ingested, M deduped
• Maintenance: Mar/Apr/May fully posted ($4,400 ea)
• Reserve interest: $X.XX (Jan-May 2026)

Flagged for review (please reply):
• CHECK 3679 ($175 on 2025-10-01) — confirm category
• Unknown vendor "ZELLE PAYMENT FROM X" ($Y on date) — confirm vendor + category
• Off-pattern deposit $4,000 on 2026-03-04 — Assessment or other?

Workbook: [Drive link]
```

Tom replies with classifications (e.g. "CHECK 3679 = Cleaning"). Skill catches the reply (via `claude-alerts-listener` thread-reply path), updates the workbook + VENDOR_CLASSIFICATIONS.md / CHECK_LEDGER.md, and confirms.

## Reserve fund

Same drop pattern. Two acceptable input shapes:
- **Screenshot**: extract `INTEREST` rows + running balance via vision OCR.
- **xls**: openpyxl read.

For each new INTEREST row: post to Reserve Fund → Interest Income row, txn-date month. Verify running balance matches what the workbook reconstructs (Reserve Balance Forward + Interest Income running sum). If mismatch, flag.

## Conventions

- **Maintenance income → SERVICE MONTH posting** (via clustering).
- **Everything else → TXN-DATE month**.
- **Never auto-post a CHECK or vendor unless confidence = High**. Flag rather than guess.
- **New vendor → append to VENDOR_CLASSIFICATIONS.md after Tom confirms**.
- **Reserve interest is small ($0.40–$0.55/mo) — never silently drop it**.

## Drive snapshot (on-demand only)

If Tom asks to share with a CPA/coowner or wants a Drive copy: upload the current iCloud xlsx via `mcp__claude_ai_Google_Drive__create_file`. This creates a new Drive file (no in-place update). Surface the new file ID to Tom. The 2026-05-20 original (`1q4M2bn5GHZTiBuj4RoW8k7n474fOws9i`) is stale — Tom can delete manually.

## Files

- `references/VENDOR_CLASSIFICATIONS.md` — vendor → (line item, month rule) table
- `references/CHECK_LEDGER.md` — confirmed CHECK # classifications (append-only)
- `update_pl.py` — main script (BUILT 2026-05-20). Citi CSV parser + classifier + maintenance clustering + xlsx mutator. Invoke: `python3 ~/.claude/skills/coop-finances/update_pl.py --csv path/to/citi.csv [--reserve-csv path] [--since YYYY-MM-DD] [--dry-run]`. Default `--since=2026-01-01` (treasurer transition boundary; cells/txns before this date are never touched). Output is JSON to stdout for the calling agent to format into a Slack/inline summary.

## Wiring status

1. ✅ **launchd plist** — `~/Library/LaunchAgents/com.tomseo.scheduled.coop-finances-prompt.plist` loaded 2026-05-20. Fires 1st of month 9 AM ET. Runs `~/.claude/scheduled-tasks/coop-finances-prompt/run.sh` which posts the monthly prompt via `send-alert`.
2. ✅ **Worker patch** — `slack-retro-webhook` index.ts now passes `files[]` through into job args. **Requires deploy**: `cd ~/.claude/cloudflare-workers/slack-retro-webhook && wrangler deploy`.
3. ✅ **Listener branch** — `claude-alerts-listener/SKILL.md` has a "Special branch: Coop finances ingest / classify" section that handles both attachment ingests (Sub-flow A) and text classifications (Sub-flow B: `CHECK <N> = <Category>`, `VENDOR <substring> = <Category>`).
4. ✅ **Slack `xoxb` bot token** with `files:read` scope provisioned on the `claude` Slack app, SOPS-encrypted at `~/.claude/.slack-bot-token.enc`. `processor.py` was patched (2026-05-20) with `_sops_decrypt()` + `_skill_env()` to inject as `SLACK_USER_TOKEN` env var on every `claude --print` invocation. Decryption uses `~/.config/sops/age/keys.txt`; result cached in-process for the processor's lifetime.
5. 🔜 **Reserve screenshot OCR** — when a PNG/JPG is dropped, the listener reads it via vision and transcribes to a temp reserve CSV before invoking update_pl.py. Logic is documented in the listener branch; verify on first real screenshot drop.
