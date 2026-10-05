---
name: office-cleaning-expense
description: >
  Log an office cleaning expense to the "Inverted I - Outstanding Expenses" Google Sheet
  (MC tab) via a Google Apps Script web app endpoint. Uses a Python HTTP POST — no Chrome
  automation, no OAuth required. Also scans iMessages from Lupe Hernandez (or any office
  cleaner) to detect cleaning confirmations and trigger logging automatically.

  Always trigger this skill when: Lupe Hernandez confirms a cleaning via iMessage, Tom
  says "log the cleaning expense", "log office cleaning", "Lupe confirmed", "cleaning done",
  "log Lupe", or any variant indicating an office cleaning happened. Also trigger when Tom
  asks to clean up duplicates in the cleaning expense log or rename "Emma Hernandez" to
  "Lupe Hernandez" — those are operations on the same Apps Script-backed sheet.
---

# Office Cleaning Expense Logger

Logs an office cleaning expense to the **MC tab** of the
"Inverted I – Outstanding Expenses" Google Sheet by POSTing to a deployed
Google Apps Script web app. No Chrome required.

The deployed endpoint enforces idempotency and vendor-name normalization
server-side, so callers can't accidentally double-log or write the wrong name.

---

## Canonical vendor name

The cleaner is **Lupe Hernandez**. That is the only acceptable vendor name in
the sheet. The endpoint normalizes every caller-supplied name to
`Lupe Hernandez` via these aliases (case-insensitive, whitespace-tolerant):

- `Lupe Hernandez`
- `Emma Hernandez` (historical mis-transcription — Lupe was once incorrectly
  recorded as Emma; the alias exists to repair any straggler that slips
  through)
- `Lupe`
- `Emma`

If a caller passes one of the above, the row is written with
`Lupe Hernandez`. If a caller passes something completely different, the
endpoint passes it through unchanged (so unrelated vendors aren't silently
rewritten) — but in normal cleaning-log flow this should never happen.

---

## Idempotency guarantee

The endpoint **will not write a duplicate row**. Before append, it scans rows
7..lastRow on the MC tab. If a row already exists with the same
(vendor, date, category) triple, it returns `{status: 'duplicate'}` and skips
the write. It also normalizes the existing row's amount formatting if it's
stored as `100` instead of `$100.00`.

This prevents the failure mode that produced the 2026-05-25 duplicate (one
row at amount `100`, one at `$100.00`).

---

## Infrastructure

| Parameter | Value |
|---|---|
| Apps Script Project | `1Hs9f1ii8V4kpPvaNoKLPOiu074yMk_RsojfGez0Y0tZszqK2wdWsLEtc` |
| Web App URL | `https://script.google.com/macros/s/AKfycbyQLZe0i8Pe4Bxh43Psbq_UfT9vZq3cwfCGkycIkEhKsQrPmC6ZgB1DgdtAMSJAqbiO/exec` |
| Spreadsheet ID | `1E2UM9FprxlHjcvKv41EgZuAlmMMrtf-UUdk1DiqlYEs` |
| Sheet Tab | `MC` (Management Company) |
| Executes as | tom@invertedcap.com |
| Access | Anyone (no auth required on caller side) |

The web app URL is stable — it does not change when new versions are deployed.
To verify the endpoint is live: HTTP GET to the URL returns `Expense logger active`.

Canonical source code is in `Code.gs` in this skill folder. The deployed copy
must match.

---

## Entry modes

- **Live loop (primary, NOT this skill)** — the deterministic 5-minute
  `com.invertedcap.lupe-paid-watch` launchd job
  (`~/.claude/scheduled-tasks/office-cleaning-expense/lupe_watch.sh` +
  `check_lupe_paid.py` in default mode) runs the state machine below with zero
  LLM: reminder + text on confirmation; sheet POST + reminder check-off + ONE
  combined text on payment. Any change to the state machine or notification
  contract is made in that CODE (and its harness), then summarized here.
- **Job mode (weekly reconcile)** — a weekly launchd job (Monday 8:10 AM,
  `~/.claude/scheduled-tasks/office-cleaning-expense/sweep.sh`) snapshots the
  thread and enqueues `{mode:"reconcile", window_hours:168, thread_rows_file}`
  on the claude-job-queue → this skill runs the script in reconcile mode (below),
  does the judgment pass, and alerts. It exists because the watcher was silently
  dead 2026-09-03→07 (launchd PATH lacked `uv`; `imessage-read.sh` used
  `immutable=1`, which hides WAL-fresh chat.db rows) and the 2026-08-22 cleaning
  was lost to the old weekly sweep failing the same silent way. 168h equals the
  weekly cadence so coverage rolls with no gap — NEVER narrow it.
- **Manual** — Tom asks directly ("log the cleaning expense", "Lupe
  confirmed", maintenance ops). Same script, live thread read.

## The state machine (defined by Tom, 2026-09-07)

Two events drive everything, both read from the Lupe iMessage thread
(`+19176135344` — confirmed against Contacts, not a chat.db handle guess):

1. **Cleaning confirmation** (Lupe: "the office is clean", "it's ready", "done
   cleaning" — `CLEAN_RE`) → create the pay reminder `Pay Lupe - office cleaning`
   (Work list, due today, notes `cleaning:MM/DD/YY`). **Do NOT write to the
   sheet yet.**
2. **Payment** — Tom's ritual: an Apple Cash bubble (an attachment-only `me`
   row, `￼[attachment]`) plus a thanks/"great"/"wonderful" text, OR a strong
   word alone ("paid", "sent", "Zelle", "Venmo") → NOW log that cleaning to the
   sheet **and check off the reminder**. Thanks *without* the Cash bubble is not
   a payment (2026-09-07: the sheet write rides on this match).

The sheet records *paid* cleanings, not confirmed ones, under the **cleaning
date** (the confirmation message's local date — not the payment date; e.g.
07/04/26 was paid 07/05 and logged 07/04). A confirmation with no payment yet =
an open reminder and nothing on the sheet.

---

## As code (2026-10-04)

Steps 1–4 (read thread, pair confirmations ↔ payments, ensure reminder, POST the
expense, check off the reminder) are **code**, not prose — do not re-implement
them by hand:

```bash
S="$HOME/.claude/scheduled-tasks/office-cleaning-expense/check_lupe_paid.py"
# Reconcile/job mode — read the snapshot file sweep.sh made (headless = no Full Disk Access):
python3 "$S" --window-hours "${window_hours:-168}" --thread-file "$thread_rows_file"
# Manual mode — live read (your interactive session has FDA); add --dry-run to preview:
python3 "$S" --window-hours 168
```

Output: one JSON line per cleaning whose confirmation is in the window —
`{"action":"reconcile","cleaning_date","state":"paid|unpaid","sheet","reminder","repaired","failed","needs_tom",...}`
— then a final `{"action":"summary",...}` line.

- `sheet`: `success` (row was missing → written = a repair) · `duplicate`
  (already there — success, not a repair) · `error:…` · `n/a` (unpaid).
- `reminder`: `completed` / `created` (repairs) · `none-open` (paid before any
  reminder existed, e.g. 09/07/26 — not an error) · `exists` ·
  `not-created-other-open` / `not-created-older-unpaid` (`needs_tom`: the
  script never opens a 2nd Lupe reminder, because the live watcher assumes at
  most one and a single payment would otherwise complete both) ·
  `create-failed` / `complete-failed`.

| Exit | Meaning | Do |
|---|---|---|
| 0 | Ran clean | Alert only if any line has `repaired` or `needs_tom`; else silent. |
| exit 2 | A write failed (sheet POST error, eventkit add/complete/list) | ⚠ alert with the failing line(s). |
| exit 3 | No thread rows at all (snapshot empty AND live read empty) | ⚠ alert: the watcher may be blind — check `sweep.log` / chat.db access. |
| 4 | Bad arguments | Fix the invocation; never hand-roll the steps instead. |

Default mode (`python3 "$S"`, no flags) is the live watcher's contract — one
open reminder, no window, no sheet write (lupe_watch.sh POSTs). Harness:
`~/.claude/scheduled-tasks/office-cleaning-expense/tests/test_check_lupe_paid.py`
(real-thread fixture; run it after ANY change to the script).

Why the reads go through bash, never the imessages MCP: `imessage-read.sh` reads
`chat.db` under a process holding Full Disk Access (the `/bin/bash` grant covers
launchd bash jobs like `sweep.sh` and `lupe-paid-watch`); the MCP runs under a
node/uv parent whose grant is absent in unattended runs (TCC blames the
responsible process). A headless job reading chat.db at all fails "Full Disk
Access denied" (2026-09-08, job D71A7346) — hence the snapshot file.

---

## Judgment pass (after the script — reconcile and manual)

The regexes are deliberately tight (a false payment writes $100 to the books).
Read the window of the thread yourself for what they can miss, and **report —
don't auto-write** — anything you find:

- **Loose confirmation phrasing** the script didn't pair. `CLEAN_RE` covers
  every completion phrasing in the real thread as of the 2026-10-04 census
  (incl. the former misses 2026-05-10 "I cleaned office today" and 2026-06-22
  "yesterday I went to clean the office"); anything new is a regex gap — if
  Lupe clearly reported a clean and Tom clearly paid (Cash bubble), check the
  sheet (`?cmd=list`, below), say so in the alert, and add the phrasing to the
  harness census.
- **Text-less Cash** (2026-06-22: Cash bubble, no thanks text) → not matched.
- **Relative cleaning day** — the script resolves it (`cleaning_date_for`):
  "yesterday" → day before the message, a weekday name ("on Sunday") → the
  most recent such day, "today"/nothing → message day. Flag only if the
  wording is ambiguous beyond those forms.

Negatives must stay negatives: "I couldn't clean the office this weekend",
"I won't be able to clean this week".

---

## Alerts

**Delivery depends on mode.** The live 5-min watcher (`lupe_watch.sh`) owns the
real-time office-is-clean / paid TEXTS to Tom (1:1 `+12012567714`, never the
family group) — that is hardcoded there, NOT done by this skill.

- **Reconcile/job mode → Slack `#claude-alerts`, NOT a text** (Tom, 2026-09-09:
  the weekly reconcile is an infra/self-heal backstop). Alert ONLY when a line
  has `repaired: true` or `needs_tom: true`, the exit code is 2 or 3, or the
  judgment pass found something. Otherwise **silent**.

  ```bash
  printf '%s\n' "$BODY" | "$HOME/.claude/skills/send-alert/send.sh"
  ```

  Body shapes (GitHub-flavored markdown):

  ```
  💸 <u>**Office Cleaning: Reconcile**</u>
  ✓ Repaired: cleaning MM/DD logged to the expense sheet ($100, MC tab) + reminder checked off.
  💸 <u>**Office Cleaning: Reconcile**</u>
  ⚠ Snapshot was empty and the live read found nothing in the 168h window. Check `sweep.log`; the watcher may be silently dead.
  ```

- **Manual mode → report the outcome inline** in the session. No text, no Slack.

On send failure, log the error to the scheduled task's `audit-log/`.

**One-off log for a date the thread doesn't show** (Tom: "log the cleaning for
10/02"): `python3 -c 'import sys; sys.path.insert(0, "'"$HOME"'/.claude/scheduled-tasks/office-cleaning-expense"); import check_lupe_paid as m; print(m.post_expense("10/02/26"))'`
— `success` and `duplicate` both mean the row is on the sheet; anything else is
a failure (do not retry blindly).

---

## Maintenance Operations

The deployed endpoint supports three GET commands for one-off cleanup:

- `GET ?cmd=list` — returns all data rows as JSON. Useful for diagnosis without
  UI access.
- `GET ?cmd=normalize_vendors` — rewrites any vendor alias (e.g.
  `Emma Hernandez`) to `Lupe Hernandez` in place. Idempotent.
- `GET ?cmd=delete_dupes` — collapses any (vendor, date, category) groups to one
  row. Keeps the row with `$N.NN` amount formatting and deletes the rest.
  Idempotent — safe to re-run. Also normalizes vendor names and amount
  formatting as it goes.

Example:

```bash
curl 'https://script.google.com/macros/s/AKfycbyQLZe0i8Pe4Bxh43Psbq_UfT9vZq3cwfCGkycIkEhKsQrPmC6ZgB1DgdtAMSJAqbiO/exec?cmd=delete_dupes'
```

Trigger `delete_dupes` when Tom reports a visible duplicate in the sheet, and
`normalize_vendors` when he reports a name mismatch (Emma → Lupe).

---

## Sheet Structure Reference

- Row 1: blank
- Row 2: "MANAGEMENT COMPANY EXPENSES" title
- Row 4: Total row (formula-driven, auto-updates when rows are appended)
- Row 6: Headers — VENDOR / DATE / AMOUNT / CATEGORY / FILE NAME
- Rows 7+: Expense entries (appended via `appendRow`)

The Apps Script appends rows as `['', vendor, date, amount, category, '']` —
the leading blank holds column A; the trailing blank holds FILE NAME.

---

## Updating the Apps Script

The deployed code must match `Code.gs` in this skill folder. To deploy:

1. Edit `Code.gs` in this folder.
2. Open the Apps Script project:
   `https://script.google.com/d/1Hs9f1ii8V4kpPvaNoKLPOiu074yMk_RsojfGez0Y0tZszqK2wdWsLEtc/edit`
3. Paste the file contents into the editor, save (Cmd+S).
4. **Deploy → Manage deployments → Edit → New version → Deploy.**

The web app URL stays the same across redeployments.

If `clasp` auth is current, you can also push directly:

```bash
cd /Users/tomseo/.claude/skills/office-cleaning-expense
clasp push       # uploads Code.gs to the project
clasp deploy --description "v3"
```

When `clasp` auth is stale (`invalid_rapt`), redeploy via the browser UI above.
