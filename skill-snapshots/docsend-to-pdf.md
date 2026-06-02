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

The Dropbox-redesigned data room viewer is a React SPA backed by GraphQL at `https://docsend.com/presentation/graphql`. Production React is compiled without DevTools hooks — `__reactFiber` keys are NOT present on rendered nodes (the old fiber-walk approach returns `null` for every row). The doc list lives in the GraphQL response, not the DOM.

**The user's existing authorized Chrome session is the auth source.** Email-verified data rooms (a server-side flag Harsh-style founders enable) cannot be bypassed from a fresh Playwright context — the verification link goes to email. Operate inside Tom's already-loaded Chrome tab.

### Step 1: Get the folder slug (spaceLayout query)

Run this fetch from the page context (cookies attach automatically via `credentials: 'include'`). The folder slug returned (`url` field) is what every subsequent query needs.

```javascript
fetch('https://docsend.com/presentation/graphql', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  credentials: 'include',
  body: JSON.stringify({
    query: `query spaceLayout($linkUrl: String!, $timezoneOffset: Int!, $folderUrl: String, $wldn: String, $accessedFromEmailVerification: Boolean!) {
      spaceFolder(linkUrl: $linkUrl, timezoneOffset: $timezoneOffset, folderUrl: $folderUrl, wldn: $wldn, accessedFromEmailVerification: $accessedFromEmailVerification) {
        authorizedData { data { url childFolderIds databaseId href } }
      }
    }`,
    variables: {accessedFromEmailVerification: false, linkUrl: '<SPACE_SLUG>', timezoneOffset: -240, wldn: null},
    operationName: 'spaceLayout'
  })
}).then(r => r.json())
```

Extract `data.spaceFolder.authorizedData.data.url` — that's the folder slug (e.g. `nskt9ub53m7ujgaf`). Multi-folder data rooms expose `childFolderIds`; iterate them as needed.

### Step 2: Get the doc list (spaceFolder.contents.nodes)

```javascript
fetch('https://docsend.com/presentation/graphql', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  credentials: 'include',
  body: JSON.stringify({
    query: `query getDocs {
      spaceFolder(linkUrl: "<SPACE_SLUG>", folderUrl: "<FOLDER_SLUG>", timezoneOffset: -240, accessedFromEmailVerification: false, wldn: null) {
        authorizedData { data { contents { nodes { __typename ... on SpaceDocument { id name url href } } } } }
      }
    }`,
    operationName: 'getDocs'
  })
}).then(r => r.json())
```

Each node has `name`, `url` (doc slug), and `href` (e.g. `https://docsend.com/view/u7h9vv5xxxutrafi/d/c2t75ksznqiihuac`).

**URL shape caveat:** Doc hrefs use `/view/{space}/d/{doc}` — the `s/` is dropped from the data-room URL when descending into a child doc. The original docsend-to-pdf regex `/view/[^/]+/page_data` won't match because `[^/]+` can't span `/d/`.

**Field probing:** GraphQL introspection is disabled (`__type` queries return "Field doesn't exist on type 'Query'"). To discover new field names, intentionally query a wrong name — the error response includes suggestions ("Did you mean `X`?"). That's how `contents` (typed `FolderContentConnection` with `nodes`/`edges`/`totalCount`) was discovered for SpaceFolder, and `name`/`url`/`href` for SpaceDocument. `pages`, `images`, `pageCount`, `documentId`, `fileUrl`, `downloadUrl` do NOT exist on SpaceDocument — pages still live in the per-doc HTML.

### Step 3: Extract each doc's page-image URLs

For each doc `href`, fetch the doc HTML in the page context (preserves auth cookies), then extract `page_data` URLs with this updated regex:

```javascript
const r = await fetch(href, { credentials: 'include' });
const html = await r.text();
const pageUrls = Array.from(new Set(
  Array.from(html.matchAll(/https:\/\/docsend\.com\/view\/[^"'\s]+\/page_data\/\d+/g)).map(m => m[0])
)).sort((a, b) => parseInt(a.split('/').pop(), 10) - parseInt(b.split('/').pop(), 10));
```

Then fetch each `page_data` URL with `Accept: application/json` to get `{imageUrl}` (a pre-signed CloudFront URL).

### Step 4: Download images + compile to PDF (Python)

CloudFront image URLs are pre-signed and DO NOT require docsend cookies — download directly from Python `requests`. Expiry on signed URLs is ~40-60 minutes from extraction, so process each doc end-to-end (extract → download → PDF → upload) rather than batching all extractions first.

Use the same `Pillow` PDF-compile path as Step 1's single-doc flow.

### Step 5: Driving Tom's existing Chrome tab from Python

The skill assumes Tom's Chrome tab already passed the email-verification gate (he viewed the data room earlier). Run JS in his active tab via `osascript`:

```python
def run_js(js):
    def as_string(s):
        return '"' + s.replace('\\', '\\\\').replace('"', '\\"') + '"'
    script = 'tell application "Google Chrome"\n  execute active tab of window 1 javascript ' + as_string(js) + '\nend tell'
    return subprocess.run(['osascript', '-e', script], capture_output=True, text=True, timeout=60).stdout.strip()
```

Async fetches can't be awaited directly through `osascript`. Pattern: kick the promise, write the result to `window._x`, then poll `window._x` from Python until it flips from `'pending'`.

**What does NOT work:**
- **Programmatic clicks** on `[class*="Table-row--selectable"]` rows — no React onClick reachable (no fiber). `.click()`, `MouseEvent` dispatch, and even `cliclick` real-mouse clicks at the row's screen coords all no-op. The row has no anchor tag, no internal href; the navigation handler is a private React listener.
- **Old form-based email gate bypass** (the `link_auth_form[email]` POST in Step 1). Current data rooms use a meta CSRF token (`<meta name="csrf-token">`) and a GraphQL `authorize` mutation. Reproducing it from Python `requests` returns 404 against the new endpoint.
- **Reload-then-hook fetch interceptor.** Reloading wipes the hook before the page's own queries fire. Hook fetch on an already-loaded page and trigger your own queries — don't try to passively observe the page's initial load.
- **`notion.site` fallback for Notion-hosted artifacts in a data room.** If the data room links a Notion page on the founder's workspace (`notion.so/{workspace}/...`), the `.notion.site` subdomain returns 404 unless the page is published-to-web. Render it through Tom's logged-in Chrome session instead (see the legacy data-room PDF flow that extracted `.notion-page-content` outerHTML and Chrome-headless'd it).

**Worked precedent:** Kalos data room (`u7h9vv5xxxutrafi`), 2026-05-27. 8 docs → 9 PDFs uploaded to `Diligence/Kalos/`. See `/tmp/kalos_dataroom_pipeline.py` for the end-to-end orchestrator scaffold.

### Step 6: Companion Notion pages linked from the data room

Founders increasingly link a Notion FAQ / Deep Dive / Q&A page from the DocSend chip list (e.g. Kalos `https://www.notion.so/kaloscareers/FAQs-Inverted-...`). These aren't DocSend artifacts but ride alongside — handle them in the same materials run.

**Render path — order of fallbacks:**

1. **Notion MCP `notion-fetch`** with the URL. Returns 404 if Tom's integration is not granted access to the founder's workspace (the common case for guest-shared pages). Skip to (2).
2. **`<workspace>.notion.site/<slug>`** subdomain. Works ONLY if the founder has Published-to-Web enabled. Returns 404 ("This page couldn't be found") otherwise. Detect by `pdftotext`-ing the result and searching for the not-found copy.
3. **Tom's authenticated Chrome tab.** Open the URL in a new tab in Tom's existing Chrome (`tell window 1 to make new tab with properties {URL:...}`). Wait ~6s for the Notion SPA to render. Expand all `[aria-expanded="false"]` toggles (typical FAQ pages collapse content). Extract the `.notion-page-content` `outerHTML`.

**Image fidelity caveat — CORS-anonymous + canvas trick:**

Notion serves images via authenticated `/image/attachment%3A{uuid}` URLs that:
- Resolve correctly inside the page DOM (img tags load fine).
- Return **`Failed to fetch`** when called via `fetch(src, {credentials: 'include'})` (CORS-blocked redirect to S3).
- Taint the canvas if drawn from a default-loaded img (`SecurityError: Tainted canvases may not be exported`).

The fix is to **re-load each image with `crossOrigin = 'anonymous'`** — Notion's image endpoint DOES serve CORS headers when explicitly requested, and the canvas is then exportable:

```javascript
async function inlineImg(src) {
  const img = await new Promise((resolve, reject) => {
    const i = new Image();
    i.crossOrigin = 'anonymous';
    i.onload = () => resolve(i);
    i.onerror = reject;
    i.src = src;
  });
  const c = document.createElement('canvas');
  c.width = img.naturalWidth; c.height = img.naturalHeight;
  c.getContext('2d').drawImage(img, 0, 0);
  return c.toDataURL('image/jpeg', 0.92);  // JPEG keeps screenshots small; fall back to png on alpha
}
```

Then substitute each `<img src>` in the extracted HTML — match by attachment UUID (`/image/attachment%3A([0-9a-f-]+)`), since the HTML attribute is relative (`/image/...&amp;id=...`) while the runtime `img.src` is absolute. URL-as-key substitution misses every time.

**Re-render flow:** wrap the (image-inlined) outerHTML in the standard letter-format Chrome-headless template (same CSS reset as the email-body-to-PDF path in `materials-handler` Step 3D), then `chrome --headless --print-to-pdf`. Upload to the same `Diligence/{Company}/` subfolder. Same-filename re-uploads auto-trash the old Drive file (per `drive-upload.md` line 39), but the new fileId is fresh — see "Re-uploading on top of an existing chip" below.

### Re-uploading on top of an existing chip

When you re-upload a fixed PDF with the same filename to the same Drive folder, the Apps Script trashes the old file but mints a fresh `fileId`/`url` for the new one. The Notion chip still points to the trashed URL.

**`notion_files_property.py --label "<same name>" --url <new url>` will SKIP** — the helper's canonical-filename dedup (`_canonical_label`) matches the existing chip's name and returns `{skipped: true, message: "Filename collision (canonical match)"}`. This is the intended behavior to prevent duplicate chips, but it leaves the broken chip in place.

**Fix — direct PATCH to rewrite the existing chip's URL** (no chip-count change):

```python
from notion_files_property import _load_token, _notion_request
token = _load_token()
page = _notion_request("GET", f"/pages/{page_id}", token)
files = page['properties']['Diligence Materials']['files']
updated = []
for f in files:
    if f.get('name') == NEW_FILENAME and f.get('external'):
        updated.append({"name": NEW_FILENAME, "external": {"url": new_url}})
    else:
        updated.append({"name": f.get('name', ''), "external": {"url": f['external']['url']}} if f.get('external') else f)
_notion_request("PATCH", f"/pages/{page_id}", token, body={"properties": {"Diligence Materials": {"files": updated}}})
```

### Efficiency checklist — avoid prior session's dead-ends

The 2026-05-27 Kalos run burned cycles on attack vectors that never work. Skip these:

1. **No `__reactFiber` walk on data room rows.** Production React strips the fiber key. `Object.keys(row).filter(k => k.startsWith('__'))` returns `[]`. Don't bother probing.
2. **No programmatic click on `[class*="Table-row--selectable"]` rows.** `.click()`, `MouseEvent` dispatch, double-click, and even `cliclick` on screen-coords ALL no-op. Rows have no anchor/href and the onClick handler is private React state. Use GraphQL directly.
3. **No form-based email-gate bypass.** Old `link_auth_form[email]` POST returns 404. Current auth is GraphQL `authorize` mutation against `/presentation/graphql` — and email-verified rooms can't be bypassed at all from a fresh Playwright session (verification link goes to email). Operate inside Tom's already-authed Chrome.
4. **No reload-then-hook.** Page reload wipes the fetch hook before initial queries fire. Hook on an already-loaded page, then trigger your own queries.
5. **No `.notion.site` shortcut without verification.** Returns 404 if the page isn't Published-to-Web. `pdftotext` the rendered PDF and grep for "This page couldn't be found" before assuming success.
6. **No Playwright headless on Notion guest-shared pages.** No cookies, no content. Drive Tom's existing Chrome tab instead.
7. **GraphQL field discovery via error-message introspection.** `__type` is disabled, but error suggestions ("Did you mean `X`?") reveal valid fields. Probe wrong names intentionally — that's how `contents` was found.
8. **Process docs end-to-end, one at a time.** CloudFront signed URLs in DocSend page_data responses expire in ~40-60 min. Don't extract all 8 doc lists then download — extract → download → PDF → upload per doc.

### Permissions

This skill — and `materials-handler` calling it — runs entirely without permission prompts. Drive uploads, folder creates, Notion property PATCHes, Chrome MCP / osascript drives, even same-name re-uploads (which auto-trash the prior file): all execute autonomously when Tom invokes the materials flow. The single exception is explicit deletes via `drive_rename.py --trash` — those still gate on `--confirm` (see `feedback_always_confirm_before_delete.md`). Anything short of explicit trash is in-scope.

## Integration with Other Skills

Other skills (like `add-to-crm` and `pipeline-agent`) should reference this skill rather than duplicating DocSend conversion logic. When another skill needs DocSend conversion, it should say: "Follow the `docsend-to-pdf` skill instructions to convert the DocSend materials."

**When called from `add-to-crm`:** DocSend conversion should happen automatically as part of the CRM entry flow — do NOT wait for the user to request it separately. If the source email or materials contain DocSend links, convert them inline.

## Tagging Diligence Materials to an Opportunity

When the user asks to tag/attach converted PDFs to a specific investment opportunity:

1. Convert the DocSend document using the Python approach above.
2. Create a company subfolder and upload via the Apps Script endpoint (Step 3).
3. Update the Notion opportunity page body's **Diligence Materials** section with direct file links from the upload response.
