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
  `~/.claude/scheduled-tasks/office-cleaning-expense/check_lupe_paid.py`) executes the
  state machine below with zero LLM: reminder + text on confirmation; sheet
  POST + reminder check-off + ONE combined text on payment. Any change to the
  state machine or notification contract must be made THERE and mirrored here.
- **Job mode (daily reconcile)** — a daily 8:10 AM launchd job
  (`~/.claude/scheduled-tasks/office-cleaning-expense/sweep.sh`) enqueues
  `{mode:"reconcile", window_hours:168}` on the claude-job-queue → this skill
  re-scans the full 168h window and repairs anything the watcher missed. It
  exists because the watcher was silently dead 2026-09-03→07 (launchd PATH
  lacked `uv`; `imessage-read.sh` used `immutable=1`, which hides WAL-fresh
  chat.db rows) and the 2026-08-22 cleaning was lost to the old weekly sweep
  failing the same silent way. All actions idempotent; text Tom ONLY about
  things actually repaired. NEVER narrow the 168h window.
- **Manual** — Tom asks directly ("log the cleaning expense", "Lupe
  confirmed", maintenance ops). Same state machine.

## The state machine (defined by Tom, 2026-09-07)

Two events drive everything, both read from the Lupe iMessage thread:

1. **Cleaning confirmation** (Lupe: "the office is clean" etc.) → create the
   pay reminder. **Do NOT write to the sheet yet.**
2. **Payment** (Tom sends $100 Apple Cash — renders as an attachment-only
   `You:` message after a confirmation) → NOW log that cleaning to the sheet
   **and check off the reminder**.

The sheet records *paid* cleanings, not confirmed ones. A confirmation with no
payment yet = an open reminder and nothing on the sheet.

---

## Step 1 – Read the thread and pair events

Scan the Lupe Hernandez thread (contact `+19176135344`) over the scan window
via `tool_get_recent_messages`. Extract, in timestamp order:

- **Confirmations**: messages from Lupe reporting a completed clean. Cleaning
  date = date named in the message, else the message timestamp's date.
- **Payments**: messages from Tom that carry an attachment placeholder (`￼`,
  no text) — Apple Cash renders this way. A payment belongs to the most recent
  prior confirmation that has no payment yet.

Then reconcile each confirmation to one of two states:

| State | Actions |
|---|---|
| Confirmed, **not yet paid** | Ensure the pay reminder exists (Step 2). Nothing on the sheet. |
| Confirmed **and paid** | Ensure the expense is on the sheet (Step 3) AND the reminder is checked off (Step 4). |

Every action below is idempotent (reminder-by-title check, endpoint dedup), so
re-scanning the same window is always safe.

If the thread has no confirmations in the window, stop.

---

## Step 2 – Reminder on unpaid confirmation

For each confirmed-not-paid cleaning, ensure a reminder exists (see
`~/.claude/skills/add-reminder/SKILL.md` for conventions — Work list default,
`[IC]` prefix, single quotes because `$100`):

```bash
# skip if an open reminder with this exact title already exists:
~/.claude/tools/eventkit/eventkit list --list Work | grep -F '[IC] Pay Lupe $100 (office cleaning MM/DD)'
~/.claude/tools/eventkit/eventkit add --title '[IC] Pay Lupe $100 (office cleaning MM/DD)' --due today
```

`MM/DD` = the cleaning date — unique per cleaning, so re-runs dedup naturally.

---

## Step 3 – Log the Expense (only once PAID)

POST via Python. Format the date as `MM/DD/YY` (e.g. `04/12/26`) — the
**cleaning** date, not the payment date.

```python
import urllib.request
import urllib.parse
import json

url = 'https://script.google.com/macros/s/AKfycbyQLZe0i8Pe4Bxh43Psbq_UfT9vZq3cwfCGkycIkEhKsQrPmC6ZgB1DgdtAMSJAqbiO/exec'
params = urllib.parse.urlencode({
    'vendor': 'Lupe Hernandez',
    'date': 'MM/DD/YY',     # replace with actual date, e.g. '04/12/26'
    'amount': '100.00',     # endpoint reformats to '$100.00' on write
    'category': 'Office'
}).encode('utf-8')

req = urllib.request.Request(url, data=params, method='POST')
req.add_header('Content-Type', 'application/x-www-form-urlencoded')

with urllib.request.urlopen(req, timeout=30) as resp:
    body = json.loads(resp.read().decode('utf-8'))

if body.get('status') == 'success':
    print('Logged:', body)
elif body.get('status') == 'duplicate':
    print('Already logged on', body['date'], '— skipped.')
else:
    raise RuntimeError('Expense logger error: ' + str(body))
```

**Success response (new row written):**
```json
{"status":"success","vendor":"Lupe Hernandez","date":"04/12/26","amount":"$100.00","category":"Office"}
```

**Duplicate response (row already existed):**
```json
{"status":"duplicate","existingRow":11,"vendor":"Lupe Hernandez","date":"04/12/26"}
```

Treat `duplicate` as **success** — the expense is already in the sheet. Do
not retry.

---

## Step 4 – Check off the reminder (paid cleanings)

Once a cleaning's expense is on the sheet (Step 3 returned `success` OR
`duplicate`), find the matching open reminder and complete it:

```bash
~/.claude/tools/eventkit/eventkit list --list Work
# find the id whose title is '[IC] Pay Lupe $100 (office cleaning MM/DD)', then:
~/.claude/tools/eventkit/eventkit complete --id <ID>
```

No open reminder with that title (e.g. Tom paid before the sweep ever saw the
confirmation, so none was created — like 09/07/26) → nothing to do, not an
error.

---

## Step 5 – Send Alert

**Delivery: TEXT Tom via the Sendblue bot — NOT Slack.** (Changed 2026-09-07
per Tom: this workflow's source surface is his iMessage thread with Lupe, and
alerts follow the surface — spotted in text → text back. See
`feedback_alert_routing_by_source_surface` + `feedback_text_lane_alerts` in
memory. No Slack dupe.)

**The notification contract (Tom, 2026-09-07) — exactly two events text him:**

1. **Reminder created** (confirmation seen, unpaid) → one text.
2. **Payment seen** → reminder checked off + expense logged, and ONE combined
   text covering all three (payment detected / sheet logged / reminder
   checked). Never three separate texts.

Nothing changed → no text. If a single run does both (confirmation AND payment
arrived within the same window), the payment text alone suffices — fold the
reminder lifecycle into it ("reminder created + checked off").

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"
printf '%s\n' "$BODY" | SENDBLUE_API_SECRET="$(sops -d "$HOME/.claude/.sendblue-api-secret.enc")" \
  "$HOME/.claude/skills/sms-listener/send_imessage.sh" +12012567714 --stdin
```

- `+12012567714` = Tom's own iPhone (bot-to-owner). Never text Lupe.
- Body goes on **stdin** (quoted heredoc or `printf`) — never argv; the shell
  eats `$100` otherwise (`feedback_imessage_send_never_double_quote`).
- Plain text, no markdown (this is iMessage, not Slack).

Body shapes (include only the lines that apply):

```
🧹 Lupe confirmed cleaning (MM/DD) — reminder created: Pay Lupe $100. Sheet logs when you pay.
🧹 Payment detected — cleaning MM/DD logged to the expense sheet ($100, MC tab) and reminder checked off.
```

On send failure, log the error to the scheduled task's `audit-log/` — do not
fall back to Slack or any other channel.

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
