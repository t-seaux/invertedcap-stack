---
name: drive-save
description: >
  Upload a file directly to a specific Google Drive folder using the Drive Upload
  Apps Script endpoint. Trigger whenever Tom says "save to Drive", "upload to Drive",
  "save this PDF to Drive", "save this to [folder]", "upload this to [folder]",
  "save down to Drive", "push this to Drive", "drop this in [folder]", or any
  variant indicating he wants a file saved directly to Google Drive. Also trigger
  when Tom provides a folder ID or Drive folder URL alongside a file and wants it
  uploaded. Always trigger inline — no confirmation needed before acting.
---

# Drive Save

Upload a file to a specific Google Drive folder via the Drive Upload Apps Script.
Read `/Users/tomseo/.claude/skills/shared-references/drive-upload.md` before proceeding — it
contains the endpoint URL, API reference, Python usage, and known folder IDs.

## Workflow

### Step 1: Identify the file

The file will be one of:
- A file Tom referenced in the conversation by absolute path (he typically drops files as `@/Users/tomseo/Downloads/foo.pdf` or similar — the path is in the message)
- A file Claude just generated in the current session (at the path returned by the generating tool)
- A file Tom specifies by path on the filesystem

If the source is ambiguous, ask Tom for the path rather than guessing. In Claude Code there is no fixed "uploads" directory — the path must come from the conversation.

### Step 2: Identify the target folder

Tom will typically specify the target in one of these ways:
- A folder name (e.g., "Signal7 Investor Updates folder", "Decks folder")
- A Google Drive folder URL (`https://drive.google.com/drive/folders/{ID}`)
- A known folder name from the reference table in `drive-upload.md`

Extract the folder ID from the URL or match against the known folder IDs in the
reference. If the folder is a company subfolder under Investor Updates that doesn't
exist yet, use the `createFolder` action first, then upload to the returned `folderId`.

If Tom doesn't specify a folder, ask before proceeding.

### Step 3: Upload

Use the Python snippet from `drive-upload.md`:

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"

with open(local_path, "rb") as f:
    file_b64 = base64.b64encode(f.read()).decode("utf-8")

resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": filename,
    "fileBase64": file_b64,
    "mimeType": "application/pdf",   # adjust for non-PDF files
    "folderId": folder_id
}, allow_redirects=True, timeout=120)

result = resp.json()
print(result)  # {"success": true, "fileId": "...", "url": "...", "size": ...}
```

**MIME types for common file types:**
- PDF → `application/pdf`
- Excel → `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
- Word → `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
- PNG → `image/png`
- CSV → `text/csv`

### Step 4: Report

On success, report:
- Filename uploaded
- Target folder name (not just the ID)
- Direct file URL from the response (`result["url"]`)

On failure, report the full error from the response and suggest next steps.

## Notes

- The Apps Script endpoint trashes any existing file with the same name in the target
  folder before uploading, so re-uploads are safe and idempotent.
- Max file size is ~50MB (Apps Script limit). All typical investor updates, decks,
  and reports are well under this.
- Always use the Drive Upload Apps Script — never the local Google Drive
  filesystem path (`~/Library/CloudStorage/GoogleDrive-...`) or a FUSE mount.
  The Apps Script is the one canonical upload path.
