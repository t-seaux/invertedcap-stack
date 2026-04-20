---
name: docsend-to-pdf
description: Convert DocSend documents and data rooms to PDF files. Handles both single DocSend document URLs and multi-document data room/space URLs. Trigger whenever the user shares a DocSend link and wants it saved as PDF, or when another skill (like add-to-crm) needs to convert DocSend materials. Trigger phrases include "docsend", "convert docsend", "save docsend", "download docsend", "docsend to pdf", "save this deck", "download the deck", or any URL matching docsend.com/view/*. Also trigger when the user asks to "save down materials" or "download the data room" from a DocSend link. Always trigger inline as part of add-to-crm — do NOT wait for the user to ask separately.
---

# DocSend to PDF

Convert DocSend documents and data rooms to locally saved PDF files, upload to a company-specific subfolder in the Google Drive Diligence folder, and link in Notion.

## Known Pitfalls

- The `docsend2pdf` pip package **fails on CSRF** — do NOT use it.
- The `convert_docsend` MCP tool is **not reliably available** across sessions — do not depend on it.
- The Google Drive MCP (`google_drive_search` / `google_drive_fetch`) is **read-only** — it cannot upload files.
- `googleapis.com` does not resolve in Claude's execution environment — do NOT attempt direct Drive or Gmail API calls.
- **NEVER use Zapier webhooks** — they are unreliable and deprecated. Always use the Google Apps Script endpoint (see Step 3).

## How It Works

DocSend URLs come in two flavors:

- **Single document**: `https://docsend.com/view/{slug}` — one document, one PDF.
- **Data room / space**: `https://docsend.com/view/s/{slug}` — a container holding multiple documents, each needing its own conversion.

The key signal is `/view/s/` in the URL. If present, it's a data room. If absent, it's a single doc.

## Step 1: Convert DocSend to PDF (Python Approach)

Use this proven Python approach (pip dependency: `Pillow`):

```python
import requests, re, time
from PIL import Image
from io import BytesIO

# pip install Pillow --break-system-packages -q

session = requests.Session()
session.headers.update({
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
})

docsend_url = "<DOCSEND_URL>"

# 1. Visit the page to establish session cookies
resp = session.get(docsend_url)

# 2. Check for page_data URLs — if absent, there's an email gate
page_urls = re.findall(r'(https://[^"]+/view/[^/]+/page_data/\d+)', resp.text)

if not page_urls:
    # 2a. Bypass email gate by submitting the auth form
    csrf = re.search(r'name="authenticity_token"[^>]*value="([^"]+)"', resp.text)
    if not csrf:
        csrf = re.search(r'value="([^"]+)"[^>]*name="authenticity_token"', resp.text)
    csrf_token = csrf.group(1) if csrf else None

    method = re.search(r'name="_method"[^>]*value="([^"]+)"', resp.text)
    if not method:
        method = re.search(r'value="([^"]+)"[^>]*name="_method"', resp.text)
    method_val = method.group(1) if method else 'patch'

    now_ms = str(int(time.time() * 1000))
    resp = session.post(docsend_url, data={
        'utf8': '✓',
        '_method': method_val,
        'authenticity_token': csrf_token,
        'link_auth_form[email]': 'tom@invertedcap.com',
        'link_auth_form[email_sniffing][email_polling_id]': '',
        'link_auth_form[email_sniffing][email_polling_start]': now_ms,
        'link_auth_form[email_sniffing][email_polling_complete]': now_ms,
        'link_auth_form[email_sniffing][email_submitted_at]': now_ms,
        'link_auth_form[timezone_offset]': '240',
    }, headers={
        'Referer': docsend_url,
        'Content-Type': 'application/x-www-form-urlencoded',
        'Origin': 'https://docsend.com',
    }, allow_redirects=True)

    page_urls = re.findall(r'(https://[^"]+/view/[^/]+/page_data/\d+)', resp.text)

# 3. Extract the document title from <meta> tags
title_match = re.search(r"itemprop='name'>\s*<meta\s+content='([^']+)'", resp.text)
if not title_match:
    title_match = re.search(r"content='([^']+)'\s+name='twitter:title'", resp.text)
doc_title = title_match.group(1) if title_match else None

# If title is generic DocSend description, discard it
if doc_title and 'docsend helps' in doc_title.lower():
    doc_title = None

# 4. Sort and dedupe page_data URLs
page_urls = sorted(set(page_urls), key=lambda x: int(x.split('/')[-1]))

# 5. Fetch each page_data endpoint (returns JSON with "imageUrl" field)
images = []
for page_url in page_urls:
    data = session.get(page_url, headers={'Accept': 'application/json'}).json()
    img_resp = session.get(data['imageUrl'])
    img = Image.open(BytesIO(img_resp.content))
    if img.mode != 'RGB':
        img = img.convert('RGB')
    images.append(img)

# 6. Compile into PDF
output_path = '/Users/tomseo/Downloads/<FILENAME>.pdf'
images[0].save(output_path, 'PDF', save_all=True, append_images=images[1:], resolution=150)
```

## Step 2: Name the PDF File

Use the **document title from the DocSend `<meta>` tag** (extracted in Step 1) to name the file:

- Format: `[Company Name] - [Document Title].pdf`
- Example: If `doc_title` = "Clerical Seed Investment Memo" and the company is Clerical → `Clerical - Seed Investment Memo.pdf`
- Example: If `doc_title` = "Series A Pitch Deck" and the company is Acme → `Acme - Series A Pitch Deck.pdf`
- **Strip the company name from `doc_title` if it's redundant** — e.g. if `doc_title` = "Clerical Seed Investment Memo", the filename becomes `Clerical - Seed Investment Memo.pdf` (not `Clerical - Clerical Seed Investment Memo.pdf`).
- **Fallback**: If `doc_title` is `None` or generic (e.g. "DocSend"), fall back to a descriptive name based on context: `[Company Name] - Pre-Seed Pitch Deck.pdf`, `[Company Name] - Memo.pdf`, etc. If you have multiple documents from the same company, give each a distinct name — never let two files share the same filename.

## Step 3: Upload to Google Drive via Apps Script

**NEVER use Zapier.** Use the Google Apps Script endpoint for all Drive operations.

Read the full reference at `/Users/tomseo/.claude/skills/shared-references/drive-upload.md` for the endpoint URL and payload format.

The workflow is:

1. **Create a company subfolder** under the Diligence root folder (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`). This is idempotent — if the folder already exists, it returns the existing one.
2. **Upload each PDF** into the company subfolder using the returned `folderId`.
3. **Use the direct file URLs** (from the upload response `url` field) in the Notion page body.

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"
DILIGENCE_ROOT = "1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d"

# 1. Create company subfolder
folder_resp = requests.post(DRIVE_URL, json={
    "action": "createFolder",
    "folderName": "<COMPANY_NAME>",
    "parentFolderId": DILIGENCE_ROOT
}, allow_redirects=True, timeout=60)
folder_result = folder_resp.json()
subfolder_id = folder_result["folderId"]
subfolder_url = folder_result["url"]

# 2. Upload PDF into subfolder
with open(output_path, 'rb') as f:
    pdf_b64 = base64.b64encode(f.read()).decode('utf-8')

upload_resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": filename,
    "fileBase64": pdf_b64,
    "mimeType": "application/pdf",
    "folderId": subfolder_id
}, allow_redirects=True, timeout=120)
upload_result = upload_resp.json()
file_url = upload_result["url"]  # Direct link to the file
```

**Important:** Python `requests.post(..., allow_redirects=True)` handles Apps Script 302 redirects correctly. `curl -L` does NOT work for POST.

## Step 4: Save locally and present to user

1. Save to `/Users/tomseo/Downloads/[filename].pdf`
2. Present to user via `present_files`

## Step 5: Update Notion (if linked to an opportunity)

In the Notion opportunity page body, update the **Diligence Materials** section with direct Drive file links:

```
**Diligence Materials**
- [Filename.pdf](https://drive.google.com/file/d/<fileId>/view) — [Document description] (N pages)
- [Company Diligence Folder](https://drive.google.com/drive/folders/<subfolderId>)
```

Always use the direct file URL from the upload response, not the generic Diligence root folder link.

**CRITICAL: No DocSend URLs in the final Notion page.** Once DocSend materials have been converted to PDF and uploaded to Drive, the Notion page body must contain ONLY the Drive PDF links — never the original `docsend.com/view/...` URLs. This applies to the Diligence Materials section AND any other section in the page body (e.g. Round, Source Context). If other sections reference DocSend as the source of materials, rewrite them to reference the Diligence Materials section instead. The DocSend links are transient input — the Drive PDFs are the permanent artifact.

## Data Room Handling

For data room URLs (`/view/s/{slug}`):

**Step 1: Extract document URLs using Chrome**

1. Navigate to the data room URL in Chrome (use `tabs_context_mcp` → `navigate`).
2. Wait for the page to load, then use `read_page` to confirm the document list is visible.
3. Extract document slugs from React's internal state using `javascript_tool`:

```javascript
const rows = Array.from(document.querySelectorAll('[class*="Table-row--selectable"]'));
const fiberKey = Object.keys(rows[0]).find(k => k.startsWith('__reactFiber'));

function extractItem(el) {
  let fiber = el[fiberKey];
  let depth = 0;
  while (fiber && depth < 30) {
    const props = fiber.memoizedProps || {};
    if (props.item && props.item.url && props.item.name) {
      return { name: props.item.name, url: props.item.url };
    }
    fiber = fiber.return;
    depth++;
  }
  return null;
}

JSON.stringify(rows.map(r => extractItem(r)), null, 2);
```

4. Construct full document URLs: `https://docsend.com/view/{space_slug}/d/{doc_slug}`

**Step 2: Convert each document** using the Python approach from Step 1, processing each URL sequentially. Upload all to the same company subfolder.

## Integration with Other Skills

Other skills (like `add-to-crm` and `pipeline-agent`) should reference this skill rather than duplicating DocSend conversion logic. When another skill needs DocSend conversion, it should say: "Follow the `docsend-to-pdf` skill instructions to convert the DocSend materials."

**When called from `add-to-crm`:** DocSend conversion should happen automatically as part of the CRM entry flow — do NOT wait for the user to request it separately. If the source email or materials contain DocSend links, convert them inline.

## Tagging Diligence Materials to an Opportunity

When the user asks to tag/attach converted PDFs to a specific investment opportunity:

1. Convert the DocSend document using the Python approach above.
2. Create a company subfolder and upload via the Apps Script endpoint (Step 3).
3. Update the Notion opportunity page body's **Diligence Materials** section with direct file links from the upload response.
