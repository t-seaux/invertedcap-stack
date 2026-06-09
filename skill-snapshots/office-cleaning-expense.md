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

## Step 1 – Detect Confirmation

Use `tool_get_recent_messages` or `tool_fuzzy_search_messages` to look for a recent
message from **Lupe Hernandez** confirming the cleaning. Extract the date of
cleaning from the message content or, if absent, use the message timestamp.

If no confirmation is found, stop — do not log anything.

---

## Step 2 – Log the Expense

POST via Python. Format the date as `MM/DD/YY` (e.g. `04/12/26`).

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

## Step 3 – Send Alert (only on `success`)

Read the `send-alert` skill for delivery config. Skip the alert when the
response is `duplicate` (the sheet was already up to date; no need to ping).

```
🧹 Office Cleaning Logged

Lupe Hernandez – {date} – $100.00 – Office
Logged to Inverted I – Outstanding Expenses (MC tab)
```

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
