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

**As code (2026-10-04)** — one command; never paste the endpoint snippet:

```bash
python3 ~/.claude/scripts/drive_upload.py upload "<local_path>" --folder-id <ID or folder URL> \
    [--company "<Company>"]   # get-or-create that subfolder first (e.g. a new Investor Updates company)
    [--name "<target filename>"]   # default = the local filename; MIME comes from the extension
# → {"ok":true,"fileId":"…","url":"https://drive.google.com/file/d/…/view","folderUrl":…}
```

Exit handling: **0** → Step 4 with `url` · **1** endpoint refused → report its `error`, retry once ·
**2** bad invocation (file missing / no folder) → fix the input, nothing was uploaded · **3** transport →
retry once, then report. Spec + folder names: `shared-references/drive-upload.md`.

### Step 4: Report

On success, report:
- Filename uploaded
- Target folder name (not just the ID)
- Direct file URL from the output (`url` — always present on exit 0; never report an upload without it)

On failure, report the full error from the response and suggest next steps.

## Notes

- The Apps Script endpoint trashes any existing file with the same name in the target
  folder before uploading, so re-uploads are safe and idempotent.
- Max file size is ~50MB (Apps Script limit). All typical investor updates, decks,
  and reports are well under this.
- Always use the Drive Upload Apps Script — never the local Google Drive
  filesystem path (`~/Library/CloudStorage/GoogleDrive-...`) or a FUSE mount.
  The Apps Script is the one canonical upload path.
