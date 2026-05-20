---
name: office-cleaning-expense
description: >
  Log an office cleaning expense to the "Inverted I - Outstanding Expenses" Google Sheet
  (MC tab) via a Google Apps Script web app endpoint. Uses a Python HTTP POST — no Chrome
  automation, no OAuth required. Also scans iMessages from Lupe Hernandez (or any office
  cleaner) to detect cleaning confirmations and trigger logging automatically.

  Always trigger this skill when: Lupe Hernandez confirms a cleaning via iMessage, Tom
  says "log the cleaning expense", "log office cleaning", "Lupe confirmed", "cleaning done",
  "log the $100 expense", "office cleaning logged", or any variant indicating a cleaning
  visit has occurred and needs to be tracked. Also trigger as part of scheduled iMessage
  scans for cleaning confirmations. When in doubt, trigger — this skill is fast and cheap
  to run.
---

# Office Cleaning Expense Logger

Logs a $100 office cleaning expense to the **MC tab** of the
"Inverted I – Outstanding Expenses" Google Sheet by POSTing to a deployed
Google Apps Script web app. No Chrome required.

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
To verify the endpoint is live: HTTP GET to the URL returns `"Expense logger active"`.

---

## Step 1 – Detect Confirmation

Use `read_imessages` or `get_unread_imessages` to look for a recent message from
**Lupe Hernandez** confirming the cleaning. Extract the date of cleaning from the
message content or, if absent, use the message timestamp.

If no confirmation is found, stop — do not log anything.

---

## Step 2 – Log the Expense

Post the expense via Python in Bash. Format the date as `MM/DD/YY` (e.g. `04/12/26`).

```python
import urllib.request
import urllib.parse

url = 'https://script.google.com/macros/s/AKfycbyQLZe0i8Pe4Bxh43Psbq_UfT9vZq3cwfCGkycIkEhKsQrPmC6ZgB1DgdtAMSJAqbiO/exec'
params = urllib.parse.urlencode({
    'vendor': 'Lupe Hernandez',
    'date': 'MM/DD/YY',     # replace with actual date, e.g. '04/12/26'
    'amount': '$100.00',
    'category': 'Office'
}).encode('utf-8')

req = urllib.request.Request(url, data=params, method='POST')
req.add_header('Content-Type', 'application/x-www-form-urlencoded')

with urllib.request.urlopen(req, timeout=30) as resp:
    import json
    body = json.loads(resp.read().decode('utf-8'))
    if body.get('status') == 'success':
        print('Logged:', body)
    else:
        raise RuntimeError('Expense logger error: ' + str(body))
```

**Success response:**
```json
{"status":"success","vendor":"Lupe Hernandez","date":"04/12/26","amount":"$100.00"}
```

If `status` is `"error"`, raise an exception and stop — do not retry silently.

---

## Step 3 – Send Alert

Read the `send-alert` skill for delivery config. Send one Signal Note to Self alert:

```
🧹 Office Cleaning Logged

Lupe Hernandez – {date} – $100.00 – Office
Logged to Inverted I – Outstanding Expenses (MC tab)
```

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

If the sheet structure ever changes (e.g. tab renamed), update Code.gs in the
Apps Script project, save (Cmd+S), then:

> Deploy → Manage deployments → Edit → New version → Deploy

The web app URL stays the same across redeployments.

**Current deployed code (Version 2, Apr 13 2026):**

```javascript
function doPost(e) {
  try {
    var params = e.parameter;
    var spreadsheetId = '1E2UM9FprxlHjcvKv41EgZuAlmMMrtf-UUdk1DiqlYEs';
    var sheet = SpreadsheetApp.openById(spreadsheetId).getSheetByName('MC');
    var date = params.date || '';
    var vendor = params.vendor || '';
    var amount = params.amount || '';
    var category = params.category || 'Office';
    sheet.appendRow(['', vendor, date, amount, category, '']);
    return ContentService
      .createTextOutput(JSON.stringify({status: 'success', vendor: vendor, date: date, amount: amount}))
      .setMimeType(ContentService.MimeType.JSON);
  } catch(err) {
    return ContentService
      .createTextOutput(JSON.stringify({status: 'error', message: err.toString()}))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
function doGet(e) {
  return ContentService.createTextOutput('Expense logger active').setMimeType(ContentService.MimeType.TEXT);
}
```
