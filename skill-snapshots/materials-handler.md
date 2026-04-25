---
name: materials-handler
description: >
  Download diligence materials (decks, memos, term sheets, investor updates) for a pipeline company, save to Google Drive, and link in Notion (page body + Diligence Materials property field). Handles Gmail attachments via Apps Script, DocSend links via Python conversion, direct file URLs, and body-only emails (investor updates, memos written inline) rendered to PDF via Chrome headless. Trigger on "save materials for [company]", "add [X] as diligence material for [company]", "download materials", "grab the deck", "save the deck", "save this investor update as a material", "materials for [company]", or any variant wanting diligence materials saved and linked to a Notion opportunity. Also triggers when pipeline-agent or add-to-crm delegates materials handling. Trigger if context involves saving email attachments, email bodies, or deck files to Drive and linking them to a deal, even without the word "materials". Always trigger inline — no confirmation needed.
---

# Materials Handler

Download diligence materials for a pipeline company, save them, and link them in the corresponding Notion opportunity (page body + Diligence Materials property field).

This skill exists as a standalone, manually-triggerable flow and is also called as a subroutine by the pipeline agent (Task 5) and add-to-crm (Step 6). The logic lives here — those skills reference it rather than duplicating it.

## Critical Operating Principles

1. **Act autonomously** — never ask Tom clarifying questions. Make reasonable decisions and proceed. If something is ambiguous, pick the most likely interpretation and note the assumption in the summary.
2. **Notion links go in TWO places, always** — the page body `📎 Diligence Materials` section AND the Diligence Materials Files property field. The property field is populated via Notion's internal `/api/v3/saveTransactions` endpoint, driven by osascript + Chrome (the default path in Claude Code) or Chrome MCP's `javascript_tool` if available. The public Notion API cannot set external URLs in Files properties. Do not skip the property field because "Chrome MCP is unavailable" — osascript is the bridge.
3. **osascript + Chrome is the default bridge, not a fallback** — on Tom's Mac, any step that would use Chrome MCP's `javascript_tool` runs through `osascript` driving the active Chrome tab. Navigate the tab to the target Notion page, fire an async JS IIFE that stashes its result in `window._notionResult`, then poll. See `/Users/tomseo/.claude/skills/shared-references/add-link-to-diligence-materials.md` and `/Users/tomseo/.claude/projects/-Users-tomseo/memory/feedback_notion_internal_api.md` for the exact pattern. Only skip the property-field step if Chrome itself is not running — and even then, try launching it.
4. **Four delivery paths for materials**:
   - **Gmail attachment** → Gmail Attachment Saver Apps Script → saves directly to the target Drive folder, returns `fileId` and `url` → link in Notion
   - **DocSend link** → Python convert (`requests` + `Pillow`) → save to `/Users/tomseo/Downloads/` → upload to Drive via Drive Upload Apps Script
   - **Direct file URL** (Dropbox, raw PDF) → `web_fetch` / `curl` download → save to `/Users/tomseo/Downloads/` → upload to Drive via Drive Upload Apps Script
   - **Email body (no attachment)** — e.g. investor updates, memos written inline — render the body to PDF via Chrome headless, save to `/Users/tomseo/Downloads/`, upload to Drive via Drive Upload Apps Script. Use this path whenever Tom says things like "add this investor update as a diligence material" or "save this email as a material" and the email itself is the artifact.
5. **Skip the Decks folder** — the primary target is the Diligence folder (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`). Only use the Decks folder if explicitly asked.
6. **Do not attempt direct googleapis.com API calls** — `googleapis.com` does not resolve in the Apps Script path either way, so use the deployed endpoints documented in `shared-references/`.
7. **Upload autonomy — Drive Upload Apps Script, never ask** — use the Drive Upload Apps Script (see `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`) for every non-Gmail file: call `createFolder` to get or create the company's Diligence subfolder, then `upload` with the returned `folderId` and the base64-encoded file content. On failure, retry once, then note the failure in the summary. Do not ask Tom to upload files manually.
8. **Per-company subfolders in Diligence** — all materials for a given opportunity go into a dedicated subfolder: `Diligence/[Company Name]/`. Use the Apps Script's `createFolder` action to get-or-create the subfolder idempotently under the Diligence root (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`). Use the company name exactly as it appears in Notion (the opportunity title). When linking in Notion, link to the specific file URL whenever possible, and the company subfolder URL as a fallback.

## Inputs

This skill accepts any of these input modes (auto-detect based on what Tom provides):

- **Company name** (primary mode): e.g. "save materials for Acme". The skill looks up the company in Notion, extracts founder contact emails, and searches Gmail for recent emails with attachments or deck links.
- **Specific Gmail message ID, deep link, or subject**: Processes that exact message — attachments via the Gmail Attachment Saver, or body via the email-body-to-PDF path if no attachment.
- **Explicit artifact + company** (e.g. "add the Chief Rebel investor update as a diligence material for Chief Rebel"): Find the referenced artifact in Gmail (most recent match on the named topic), then treat it as a single-item materials add against the named opportunity. Pick the most recent matching email if there's ambiguity; don't stop to ask.
- **Delegation from another skill**: The calling skill passes a company name, Notion page ID, and optionally a list of Gmail message IDs or material URLs to process.

## Notion Context

```
Opportunities data_source_id: fab5ada3-5ea1-44b0-8eb7-3f1120aadda6
Agent View URL: https://www.notion.so/5fa871c765d74251b8f96b63f248ef25?v=31400beff4aa80fdb2e0000c1b6ae673
Google Drive Diligence folder ID: 1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d
Google Drive Diligence folder URL: https://drive.google.com/drive/folders/1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d
Google Drive Decks folder ID: 1YUxmNe8LI9ctlMKQ22WoVSbyjkyKJXr0
```

## Environment Detection

Chrome is assumed available on Tom's Mac. The property-field write uses `osascript` to drive the active Chrome tab by default. If Chrome MCP (`Claude_in_Chrome`) is exposed in the current harness, prefer it — otherwise fall through to `osascript`. Only skip the property-field step if Chrome itself isn't running and can't be launched.

All file uploads go through the Drive Upload Apps Script and the Gmail Attachment Saver Apps Script. Body-only email rendering uses Chrome headless (`/Applications/Google Chrome.app/Contents/MacOS/Google Chrome --headless --disable-gpu --no-pdf-header-footer --print-to-pdf=...`).

## Step 1: Resolve the Company in Notion

Search for the opportunity by company name using `notion-search` with `data_source_url: "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"`.

Extract from the opportunity page:
- **Page ID** (for later update)
- **Name** (title)
- **Contact** (founder email addresses)
- **🏁 Founder(s)** (founder names from the relation, for Gmail search)
- **Existing page content** (to check for an existing Diligence Materials section)

If the company is not found in Notion, inform the user and stop.

## Step 2: Search Gmail for Materials

Run targeted Gmail searches combining the company name, founder names, and contact emails with attachment and link signals. Use these queries (adjust based on available founder info):

1. `from:(<founder_email>) has:attachment newer_than:7d`
2. `"<company_name>" has:attachment newer_than:7d`
3. `from:(<founder_email>) ("docsend" OR "dropbox" OR "drive.google" OR "figma.com" OR "miro.com" OR "loom.com" OR "pitch.com" OR "notion.site" OR "canva.com" OR "deck" OR "memo" OR "pitch") newer_than:7d`

Use `maxResults: 10` per query. Deduplicate by message ID across queries.

For each hit, call `gmail_read_message` to confirm relevance:
- **Include**: Pitch decks, financial models, memos, one-pagers, term sheets, side letters, data room links, blurbs.
- **Exclude**: SaaS marketing, newsletters, calendar invites, receipts.

Classify each relevant email's materials into categories:
- **Gmail attachment** (binary file attached to the email)
- **DocSend link** (URL matching `docsend.com/view/`)
- **Direct file URL** (Google Drive share link, Dropbox link, raw PDF URL)
- **Data room link** (DocSend `/view/s/` or similar multi-doc container)
- **Link-only / non-convertible** (Figma, Miro, Loom, Pitch.com, Canva, Notion.site, etc.) — interactive/hosted materials that cannot be cleanly downloaded or converted to PDF. These get linked as-is; the external URL is the canonical artifact.

## Step 3: Process Materials

### 3A: Gmail Attachments (Apps Script)

Use the Gmail Attachment Saver Apps Script to save attachments directly to the company's Diligence subfolder on Google Drive. No Chrome required.

Read the reference at `/Users/tomseo/.claude/skills/shared-references/gmail-attachment-saver.md` for the deployment URL and full API details.

For each email containing relevant attachments:

1. **Determine the target Drive folder ID**: Call the Drive Upload Apps Script's `createFolder` action with `name = [Company Name]` and `parentId = 1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d` to get-or-create the company's Diligence subfolder idempotently. Use the returned `folderId` as the target. If the Apps Script is unavailable, fall back to the Diligence root folder ID (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`) so the file still lands in Diligence.

2. **Call the Apps Script endpoint** via Python:
   ```python
   import requests

   APPS_SCRIPT_URL = "https://script.google.com/macros/s/AKfycbwf_3QMmAYW8YB3WjfCW94p2pB3M-0sXCkqOqDg1BZ5_tD9eLauAJ2BEXpuusYORDBJ/exec"

   resp = requests.post(APPS_SCRIPT_URL, json={
       "messageId": message_id,        # hex ID from Gmail MCP
       "driveFolderId": folder_id       # company subfolder ID or Diligence root
   }, allow_redirects=True, timeout=60)

   result = resp.json()
   ```

3. **Extract file metadata from the response**: Each file in `result["files"]` contains `fileName`, `fileId`, `url` (direct Drive link), `mimeType`, and `size`. Use `url` directly for Notion linking — no separate Drive MCP search needed.

4. **Error handling**: If `result["success"]` is `false`, log the error and fall back to generating a Gmail deep link (`https://mail.google.com/mail/u/0/#all/<messageId>`) for manual download. Do not retry more than once.

### 3B: DocSend Links (No Chrome Needed)

Follow the `docsend-to-pdf` skill at `/Users/tomseo/.claude/skills/docsend-to-pdf/SKILL.md` for the exact Python conversion approach:

1. Use the `requests` + `Pillow` method to convert the DocSend document to PDF.
2. Name the file using the DocSend `<meta>` title: `[Company Name] - [Document Title].pdf`. Strip redundant company name if present in the title. Fallback: `[Company Name] - Deck.pdf`.
3. Save to `/Users/tomseo/Downloads/[filename].pdf`.
4. Present to user via `present_files`.
5. **Upload to Google Drive Diligence folder via Drive Upload Apps Script**: See `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`. First call `createFolder` to get-or-create the company's Diligence subfolder under `1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`, then call `upload` with the returned `folderId` and the base64-encoded file content. On success, use the returned `fileId` and `url` directly — no separate Drive MCP search needed. On failure, retry once, then note the failure in the summary but do not ask Tom to upload manually.
6. **Use the URL returned by the Apps Script**: The `upload` response contains `fileId` and `url` (`https://drive.google.com/file/d/<fileId>/view`). Use this link directly in the Notion page body — no separate Drive MCP lookup needed.

For **data room URLs** (`/view/s/`), follow the docsend-to-pdf skill's data room handling to extract individual document URLs first, then convert each.

### 3C: Direct File URLs

- **Google Drive share links**: Extract the file ID directly from the URL. No download needed — just construct the view link and document in Notion.
- **Dropbox or raw PDF URLs**: Use `web_fetch` or `curl` to download the file, save to `/Users/tomseo/Downloads/`. Then upload to the company's Diligence subfolder using the Drive Upload Apps Script — same `createFolder` → `upload` pattern as 3B. The Apps Script returns `fileId` and `url` directly.

### 3D: Email Body → PDF (Chrome Headless)

Use this path when the email *body itself* is the material — no attachment, no DocSend link. Typical triggers: investor updates, inline memos, "save this email as a diligence material for [company]".

1. Fetch the full message content via `gmail_get_thread` (`messageFormat: FULL_CONTENT`) to get `plaintextBody`, subject, sender, and date.
2. Render an HTML file at `/Users/tomseo/Downloads/<slug>.html` that wraps the body in a clean layout: title (subject), a meta line (company · sender · date · subject), and the body content. Preserve sections and bullets from the source — don't invent structure. Use the standard print-friendly CSS (`@page { size: Letter; margin: 0.75in; }`, `-apple-system` font stack, 12pt body, bordered header).
3. Shell out to Chrome headless to convert to PDF:
   ```bash
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
     --headless --disable-gpu --no-pdf-header-footer \
     --print-to-pdf="/Users/tomseo/Downloads/<Filename>.pdf" \
     "file:///Users/tomseo/Downloads/<slug>.html"
   ```
4. Name the PDF `[Company Name] - [Descriptive Label].pdf`. For investor updates, derive the label from the subject (e.g. "Week 20 Investor Update"). Don't include emojis or special punctuation in the filename.
5. Upload to the company's Diligence subfolder via the Drive Upload Apps Script (same `createFolder` → `upload` pattern). Use the returned `fileId` / `url` for Notion linking.

### 3E: Link-only / Non-convertible Materials (Figma, Miro, Loom, Pitch.com, Canva, Notion.site, etc.)

Use this path for interactive/hosted materials that can't be cleanly downloaded or PDF-rendered. Skip Drive entirely — the external URL itself is the canonical artifact and goes directly into both Notion locations (page body bullet AND Diligence Materials property field).

1. **Do NOT attempt to download, headless-render, or DocSend-convert these URLs.** Figma decks and Miro boards don't print usably via Chrome headless, and Loom/Pitch.com require auth or JS interactivity that breaks conversion. Trying wastes time and produces a broken artifact.
2. **Derive a display label** from the email subject or URL slug: `[Company] - Deck (Figma)`, `[Company] - Brainstorm (Miro)`, `[Company] - Walkthrough (Loom)`, etc. Keep the parenthetical platform tag — it tells a future reader why the link is external instead of Drive-hosted.
3. **Page body bullet** — see format in Step 4.
4. **Property field** — the external URL goes into the Diligence Materials Files property directly (Step 4's "always link the specific file URL" rule is relaxed for this path; see Step 4 for details).

## Step 4: Update the Notion Opportunity Page

The canonical home for material links is the **Diligence Materials property field** (Files property chips at the top of the page). The page body stays clean and contains only the Note section — do NOT add a `📎 Diligence Materials` body section by default.

### Page Body — DO NOT add a Diligence Materials section by default

Skip the page body entirely for materials. The property-field chips are visible at the top of every opportunity page and are the canonical surface for material links. Adding a duplicate body section just restates what's already one scroll-up, and clutters the Note section below it (see `add-to-crm/references/schema.md` — body is Note-only by default).

**Exception — only add a body section when:**
- The Note section doesn't reference the materials at all (founder attached a binary deck and didn't link it inline), AND
- There's per-material context that won't fit on the chip display label (conversion provenance like "converted from DocSend (22 pages)", or a ⚠️ status flag).

If you're writing an exception-case body section, use the bullet format below. Otherwise skip.

```
- [**[Filename]**](<URL>) — [per-material context]
```

### Notion Diligence Materials Property Field (osascript + Chrome, default)

**This step always runs.** Append each saved Drive link to the **Diligence Materials** Files property on the opportunity page. The bridge is `osascript` driving the active Chrome tab — that's the default on Tom's Mac. If the Chrome MCP `javascript_tool` is exposed in the current harness, prefer it; otherwise osascript is first-class, not a fallback.

Read the shared reference at `/Users/tomseo/.claude/skills/shared-references/add-link-to-diligence-materials.md` for the `addLinkToDiligenceMaterials` function, and `/Users/tomseo/.claude/projects/-Users-tomseo/memory/feedback_notion_internal_api.md` for the osascript bridge pattern (navigate to the page, wait ~4s, fire the JS as an async IIFE that stashes the result in `window._notionResult`, poll `JSON.stringify(window._notionResult)` via a second osascript call). Required headers for internal API calls: `x-notion-active-user-header: c6058e42-a6df-4155-9e7e-9a2a5b6af322`, `x-notion-space-id: ebe97bdb-2635-4ecc-abee-968e61632450`, `notion-audit-log-platform: web`. The Diligence Materials property key is hardcoded as `=sSX`.

**Critical: Always link the specific file URL** (`https://drive.google.com/file/d/<fileId>/view`), never the folder URL. The file ID comes from the Drive Upload Apps Script `upload` response — use it directly, don't re-search.

**Link-only materials (Step 3E) are the explicit exception** — for Figma, Miro, Loom, Pitch.com, Canva, Notion.site etc., pass the external URL itself (e.g. `https://figma.com/deck/...`) to `addLinkToDiligenceMaterials`. These materials have no Drive counterpart; the external URL is the canonical artifact and MUST still be written to the property field — "no Drive URL" is not a reason to skip.

For each file saved to Drive in the preceding steps, call the function with the opportunity page ID, the Drive file URL, and a descriptive display name that matches the PDF filename (e.g., `Chief Rebel - Week 20 Investor Update.pdf`). For link-only materials, use the external URL and a label like `Bloom - Deck (Figma)`. For multiple files, call once per URL — each call is a self-contained read-modify-write cycle.

Skip this step only if Chrome is not running and cannot be launched. In that case, note it in the summary and continue — the page body link is the interim record.

## Step 4.5: Extract Contact Signals from Materials

After materials are saved/linked, mine them for better founder contact info than what's already in the Notion Contact property. Decks and investor updates almost always contain the founder's canonical work email on a contact slide — ContactOut's best guess is often a stale personal address (`@gmail`, `@live`, `@outlook`, etc.).

### What to extract

- **Emails** — match `[\w.+-]+@[\w-]+\.[\w.-]+` across the material text.
- **LinkedIn URLs** — match `linkedin\.com/in/[\w-]+`.
- **Phone numbers** — optional; only capture if formatted (`+1 555-xxx-xxxx` etc.), not loose digit strings.

### How to read each material type

- **PDFs saved locally** (Step 3B DocSend, Step 3C Dropbox/raw, Step 3D email-body-to-PDF): run `pdftotext -layout /Users/tomseo/Downloads/<file>.pdf -` and scan stdout.
- **Gmail Attachment Saver PDFs** (Step 3A — written straight to Drive, no local copy): download the bytes via `curl -sL "https://drive.google.com/uc?export=download&id=<fileId>" -o /tmp/<fileId>.pdf` then `pdftotext -layout`. If Drive returns the "confirm download" interstitial for large files, skip this material — don't fight the virus-scan gate.
- **Link-only materials** (Step 3E — Figma, Miro, Loom, Pitch, Canva, Notion.site): drive the active Chrome tab via osascript, navigate to the URL, wait ~6s for content to render, pull `document.body.innerText`, stash in `window._figmaText`, poll it. Reuse the exact osascript bridge pattern from `feedback_notion_internal_api.md`. Figma decks render their contact slide's email as plain text in the DOM; Miro/Loom behave similarly.

### How to decide whether to update Notion Contact

Read the current Contact property from the opportunity page, then apply this rule:

1. **Contact is empty** → write any extracted email whose local-part contains the founder's first or last name (from the `🏁 Founder(s)` relation). If multiple match, pick the one on a custom domain over a free provider.
2. **Contact is set, currently on a free provider** (`@gmail.com`, `@live.com`, `@outlook.com`, `@yahoo.com`, `@hotmail.com`, `@icloud.com`, `@me.com`, `@aol.com`) **AND** an extracted email is on a custom domain AND matches the founder name → **upgrade**: replace the Contact property with the extracted email. Preserve the existing email as a comment in the Source Context section (`Earlier contact on file: <old>`) so the prior record isn't lost.
3. **Contact is set and already on a custom domain** → do nothing, even if a different custom-domain email appears in the materials. Don't guess across two plausible work emails.
4. **Extracted email doesn't match any founder name** (e.g. `hello@bloom.site`, `investors@bloom.site`) → do not write it to Contact. Generic inbound addresses belong in the page body, not the Contact property.

### How to write the update

Use `notion-update-page` with `command: "update_property"` on the `Contact` property. Single value — comma-separated if the property accepts multiples (it does) and you're adding a second.

Log every decision (extracted / kept / upgraded / ignored + reason) in the Step 5 report summary so it's traceable.

### When to skip this step entirely

- Zero materials saved (nothing to read).
- All materials are Gmail Attachment Saver PDFs AND Drive download fails for all of them — note the skip in the summary.
- `pdftotext` is not on PATH (install via `brew install poppler` if needed, but don't fail the whole skill — note and skip).

## Step 5: Report Summary

Return a concise summary:

```
📎 MATERIALS HANDLER — [Company Name]

Drive folder: Diligence/[Company Name]/
Found: [N] materials across [M] emails
  - [filename1] → Diligence/[Company Name]/ ✅ (Gmail Attachment Saver)
  - [filename2] → Diligence/[Company Name]/ ✅ (Drive Upload Apps Script)
  - [filename3] → Diligence/[Company Name]/ ✅ (email body → Chrome headless PDF)
  - [filename4] → Pending ⚠️ (upload failed, retry or manual)
  - [Figma link] → linked as-is ✅ (link-only, not downloadable)
DocSend: [N] converted and uploaded
Contact extraction: upgraded tom@old.com → tom@company.com ✅ / kept existing (custom domain) / nothing found
Notion: Page body updated ✅ | Property field updated ✅ (osascript + Chrome) / failed ⚠️ / skipped (Chrome not running)
```

## Constraints and Known Limitations

- **Gmail MCP cannot download binary attachments** — there is no attachment download endpoint. The Gmail Attachment Saver Apps Script (`/Users/tomseo/.claude/skills/shared-references/gmail-attachment-saver.md`) is the primary path for saving Gmail attachments to Drive.
- **Google Drive MCP is read-only** — `google_drive_search` and `google_drive_fetch` cannot upload files or move files between folders. All writes go through the Drive Upload Apps Script.
- **Google Drive MCP does not index PDFs** — only Google Docs appear in search results. Prefer the `fileId` / `url` returned by the Apps Script over re-searching after upload.
- **Notion API does not support external URLs in Files properties** — the Diligence Materials property field is populated via Notion's internal API (`/api/v3/saveTransactions`), driven by osascript + Chrome by default (or Chrome MCP's `javascript_tool` if exposed). This is reliable and deterministic. Only skip if Chrome itself isn't running.
- **DocSend `docsend2pdf` pip package fails on CSRF** — use the Python `requests` + `Pillow` approach instead.

## Integration with Other Skills

### Called by pipeline-agent (Task 5)

The pipeline agent's Materials Scanner sub-agent delegates to this skill by passing: company name, Notion page ID, and any pre-identified Gmail message IDs. Gmail attachments are saved via the Apps Script endpoint, so Chrome is only needed for the Notion Diligence Materials property field (and is skipped if unavailable).

### Called by add-to-crm (Step 6)

The add-to-crm skill's materials handling step references this skill for the full download-upload-link flow. It passes: company name, Notion page ID, and specific material URLs or Gmail message IDs extracted during CRM entry creation.

### References docsend-to-pdf

For DocSend conversion specifically, this skill follows the proven Python approach documented in the `docsend-to-pdf` skill. Read `/Users/tomseo/.claude/skills/docsend-to-pdf/SKILL.md` at runtime for the conversion script and data room handling instructions.

## Behavior Rules

### Always run the Gmail Attachment Saver on every target message

Never rely on `plaintextBody` alone to determine whether attachments exist. Gmail MCP's `get_thread` / `search_threads` do not surface attachment filenames in the message body — a PDF deck can sit on the message and be completely invisible in plaintext.

**Mandatory probe on any target message:**
1. Re-run the search with `has:attachment` to confirm attachment presence (or check the thread's presence in the attachment index).
2. Run the Gmail Attachment Saver Apps Script **unconditionally** — the response tells you exactly what's attached. Cost is one API call; missing a deck costs Tom trust.

**Why:** On 2026-04-20, Kinza's Chief Rebel email was processed and the attached pitch deck (`Chief Rebel pitch deck vf_compressed.pdf`, 7.8MB) was missed — the plaintext mentioned the demo Drive link but not the deck. Tom caught it. This is the exact failure the skill is supposed to prevent.

**How to apply:**
- In this skill, investor-update, add-to-crm, and pipeline-agent materials scanner — always call the Gmail Attachment Saver on any target message, even if the plaintext body looks complete.
- Default action for "grab the materials from this email": run the attachment saver **first**, then parse body for additional links (DocSend, Drive, Dropbox). Never the other way around.
- If `has:attachment` filter returns the thread, there IS an attachment somewhere — find it.

### osascript + Chrome is PRIMARY for Notion Files property writes

Writing external URLs to Notion Files/Media properties (Diligence Materials, Online Presence, any future Files property) must use `osascript` driving the active Chrome tab against Notion's internal `/api/v3/saveTransactions` endpoint. This is the **primary** path, not a fallback.

**Why:** The public Notion API and the MCP tool both reject external URL writes to Files properties — they only accept file uploads. Tom explicitly said "prioritize osascript automation" on 2026-04-19. Do not report "Chrome unavailable, skipped — update next session."

**How to apply:**

1. **Canonical reference**: `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md` has the generalized `addLinkToFilesProperty(pageId, propertyName, url, displayName)` function plus the osascript bridge pattern (navigate → wait 4s → fire async IIFE → poll result from `window._notionResult`).

2. **One function for all Files properties** — the generalized function discovers the property key by searching the collection schema for `name == propertyName && type == 'file'`. Works for Diligence Materials, Online Presence, and any new Files property without per-property code.

3. **Property-key fallback** — schema discovery occasionally fails with "Cross-cell memcached access" on repeated `getRecordValues` calls in the same tab session. Mitigate by navigating fresh to the target page before firing the script. Known keys captured in the shared ref.

4. **Do NOT skip the property-field write** because "Chrome MCP isn't available." osascript IS the bridge on Tom's Mac. Only skip if Chrome itself cannot run — and even then try `open -a "Google Chrome"` first.

5. **Chrome prerequisites** (already set up on Tom's Mac):
   - Chrome: View → Developer → "Allow JavaScript from Apple Events" enabled
   - System Settings → Accessibility: `node` allowed
   - Tom signed in to Notion in Chrome (session cookies authenticate internal API)

6. **Required fetch headers for internal API calls** (without user + space headers, requests hit 502 `MemcachedCrossCellError`):
   - `Content-Type: application/json`
   - `x-notion-active-user-header: c6058e42-a6df-4155-9e7e-9a2a5b6af322` (Tom's user ID)
   - `x-notion-space-id: ebe97bdb-2635-4ecc-abee-968e61632450` (Inverted space ID)
   - `notion-audit-log-platform: web`

7. **Call per URL** — function is self-contained read-modify-write. Multiple URLs = sequential calls. Deduplication built in — checks for existing URL before appending. Safe to re-run.

Consumers: this skill, first-pass-diligence, neg1-enricher (Step 4.5 Online Presence), future skills needing Files property writes.
