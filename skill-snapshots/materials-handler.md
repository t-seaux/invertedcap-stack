---
name: materials-handler
description: >
  Download diligence materials (decks, memos, term sheets, investor updates) for a pipeline company, save to Google Drive, and link in Notion (page body + Diligence Materials property field). Handles Gmail attachments via Apps Script, DocSend links via Python conversion, direct file URLs, and body-only emails (investor updates, memos written inline) rendered to PDF via Chrome headless. Trigger on "save materials for [company]", "add [X] as diligence material for [company]", "download materials", "grab the deck", "save the deck", "save this investor update as a material", "materials for [company]", or any variant wanting diligence materials saved and linked to a Notion opportunity. Also triggers when pipeline-agent or add-to-crm delegates materials handling. Trigger if context involves saving email attachments, email bodies, or deck files to Drive and linking them to a deal, even without the word "materials". Always trigger inline — no confirmation needed.
---

# Materials Handler

Download diligence materials for a pipeline company, save them, and link them in the corresponding Notion opportunity (page body + Diligence Materials / Deal Docs property fields).

This skill exists in three invocation modes (per the A/B/C convention), all sharing the Steps 1–5 pipeline below:

- **Mode A (sweep)** — N/A; not currently scheduled.
- **Mode B (webhook)** — auto-fired by `gmail-webhook/materials-detect.js` when a founder on a pipeline Opp's Contact field sends an email containing materials. See "Mode B: Webhook entry" below for inputs, idempotency, and alert format.
- **Mode C (manual)** — Tom invokes directly ("save materials for [company]", forwarded email, etc.). Also called as a subroutine by pipeline-agent (Task 5) and add-to-crm (Step 6).

The processing logic (Steps 1–5) is shared across modes. Mode-specific deltas are scoped to the section below; everything else flows through the canonical pipeline.

## Mode B: Webhook Entry

**Triggered by:** `gmail-webhook/materials-detect.js` enqueues a `materials-handler` job when an inbound email passes its gate (sender on an Opp's Contact field + at least one material signal).

**Inputs:**

```json
{
  "messageId": "<gmail message id>",
  "oppId": "<notion page id>",
  "oppName": "<company name>",
  "threadId": "<gmail thread id>"
}
```

The skill is bound to a specific message + Opp; do not search Gmail freshly (Step 2 is replaced by the inputs).

**Idempotency:** per-message gate uses the `claude/materials-processed` Gmail label — see the canonical "Step 2.5: Per-message Idempotency Gate" below, which applies to all modes. For Mode B, the message set is every message in `threadId`; the trigger message is in the delta set by definition (the gate fired *because* of new content). If the delta set is empty (every message in the thread is already labeled), exit cleanly — no Notion writes, no Slack alert.

**Mode B steps:**

- Skip Step 1's full lookup — `oppId` and `oppName` are passed in — but **still fetch the Opp page to read Status** and apply the Step 0 Status Guard before any further work. Portfolio-set statuses (`Active Portfolio`, `Portfolio: Follow-On`, `Exited`) belong to `investor-update`, not here. On a guard hit, exit cleanly: no Notion writes, no Slack alert, and log the skip via `logEvent`-equivalent so the dedup trail is visible.
- Skip Step 2 (Gmail search) — the message set is `threadId`'s messages.
- Run Step 2.5 (idempotency gate) to filter to the unlabeled subset.
- Run Steps 3 + 4 normally on the delta set, scoped to the thread.
- Apply the outcome label (`claude/materials-processed` on full success, `claude/materials-failed` on any fatal per-item failure) per Step 2.5 after processing.
- Replace Step 5 with the Slack alert format below.

**Slack alert (Mode B only) — fires only on success with ≥1 new artifact:**

Posted via the `send-alert` skill. Format (GFM — `send-alert` converts to Slack Block Kit):

```
**📎 Materials: [{Opp Name}](https://www.notion.so/{oppId}) ([Email](https://mail.google.com/mail/u/0/#all/{messageId}))**

- **Page Body:** {comma-separated body section names — plain text, no links}
- **Diligence Materials:** {comma-separated chip labels, each WRAPPED as [label](url)}
- **Deal Docs:** {comma-separated chip labels, each WRAPPED as [label](url)}
```

**Alert rules:**

- Write GFM only — `send-alert/send.sh` converts to Slack Block Kit. Do NOT hand-write Slack mrkdwn (`*bold*`, `<url|text>`); it ships as literal text and breaks link tap targets.
- Header line is fully bolded with `**...**`. Two clickable segments: the Opp name (links to Notion `https://www.notion.so/{oppId}`) and the literal word `Email` wrapped in parens (links to the Gmail deep link `https://mail.google.com/mail/u/0/#all/{messageId}`). Use parens, not brackets.
- Bullet lines use `- ` (GFM list). Each bullet's field name is bolded with `**...**`, followed by `:`, then a comma-separated list — no inner bullets, no per-artifact lines.
- **Page Body items are plain text** (no links — the Opp link in the header already covers it).
- **Page Body bullet reflects actual writes only** — list `Company Blurb` ONLY if the section was newly written on this run (not skipped due to existing section, not a no-op rewrite of identical content, not a precondition-fail per the Step 4 hard-precondition gate). If the section existed before and was untouched, omit the entire `Page Body:` bullet. Mis-reporting a no-op as a write is a bug, not a cosmetic issue — Tom uses these alerts to audit what changed.
- **Diligence Materials and Deal Docs items are each individually linked** via `[label](url)`:
  - Drive-uploaded files → `[{filename}]({driveFileUrl})` (use the `url` returned by Drive Upload Apps Script — `https://drive.google.com/file/d/{fileId}/view`).
  - Link-only / interactive demos → `[{label}]({externalUrl})` (Figma, Loom, demo URLs, etc.).
  - For demo chips with credentials, the **Notion chip label** carries the full credentials (`Inlets Demo (login: demo@inlets.ai; pw: Password124!)`), but **any Slack alert text redacts the password**: render the chip in the alert as `Inlets Demo (login: demo@inlets.ai; pw: ***)`. The full credential lives ONLY in the Notion chip label — never in Slack.
- **Omit a bullet entirely** if no artifacts landed in that field on this run. (E.g. a delta with only a term sheet shows ONLY the `Deal Docs` bullet.)
- **Follow-up runs** (subsequent messages in an already-processed thread) use the exact same format — bullets only show what's new in this run, not the cumulative state.
- Skip the alert entirely on no-op runs (delta set was empty, or processing failed for every artifact).
- If processing partially fails (some chips landed, some didn't), still alert on what landed; note failures in the local skill log, not the Slack alert.

**Worked example (matches Emily/Inlets):**

```
**📎 Materials: [Inlets](https://www.notion.so/34800beff4aa81a5ba9dca2b550eb002) ([Email](https://mail.google.com/mail/u/0/#all/19dcf6ceb1af7c41))**

- **Page Body:** Company Blurb
- **Diligence Materials:** [Inlets - One-Pager (2026)](https://drive.google.com/file/d/.../view), [Inlets - Oncology Case Study](https://drive.google.com/file/d/.../view), [Inlets Demo (login: demo@inlets.ai; pw: ***)](https://app.inlets.ai/)
```

**Demo chip label format reminder:** `[Company] Demo (login: <email>; pw: <password>)` — see Step 3F.

**Idempotency dedup at the queue layer** is keyed on `materials-handler-{messageId}`, so a Pub/Sub re-delivery of the same message never re-processes. The Gmail-label check is the per-thread layer that prevents re-processing already-handled artifacts even when a different (later) message in the same thread fires the gate.

## Critical Operating Principles

1. **Act autonomously** — never ask Tom clarifying questions. Make reasonable decisions and proceed. If something is ambiguous, pick the most likely interpretation and note the assumption in the summary.
2. **Notion links go in TWO places, always** — the page body `📎 Diligence Materials` section AND the Diligence Materials Files property field. Never skip the property field.
3. **Public Notion API is the path for Files-property writes** — shell out to `~/.claude/scripts/notion_files_property.py`. As of 2026-05-13 Notion's public API supports external-URL writes to Files properties directly (PATCH `/v1/pages/{id}` with `files: [{name, external: {url}}]`); the prior internal-API / token_v2 / Chrome+osascript workarounds are obsolete. See `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md` for the canonical interface and exit codes.
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

Property-field writes use the public Notion API via `~/.claude/scripts/notion_files_property.py` — no Chrome required. The helper reads `$NOTION_API_TOKEN` (or falls back to the SOPS file at `~/code/notion-backup/.notion-token.enc.txt`).

All file uploads go through the Drive Upload Apps Script and the Gmail Attachment Saver Apps Script. Body-only email rendering uses Chrome headless (`/Applications/Google Chrome.app/Contents/MacOS/Google Chrome --headless --disable-gpu --no-pdf-header-footer --print-to-pdf=...`).

## Step 0: Status Guard (always runs)

This skill is for **pipeline opportunities only**. Before doing any Gmail searching, Drive uploading, or Notion writing, verify the resolved Opp's `Status` is NOT in the portfolio set:

- `Active Portfolio`
- `Portfolio: Follow-On`
- `Exited`

These statuses are `investor-update`'s territory — formal-comms artifacts (board decks, investor updates, fund reports) route to the Company Updates DB, not Diligence Materials. If the resolved Opp matches any of them, **abort immediately**: no Notion writes, no Drive uploads, no Slack alert. Log the skip with reason `portfolio-status-guard` and the matched status name.

**`Committed` is NOT in the portfolio set** — Tom often runs final diligence (materials, references, term-sheet review) while an Opp sits at Committed before flipping to Active Portfolio. Treat Committed as pipeline; let materials flow through normally.

This guard applies in all modes:
- **Mode B (webhook)** — fetch the Opp page from `oppId` (Mode B otherwise skips Step 1) just to read Status before proceeding.
- **Mode C (manual / delegated)** — Step 1 already loads the page; extract Status there and gate on it before Step 2.

If Tom invokes manually with explicit "save this as a diligence material for [portfolio company]" intent, surface a one-line note pointing him at `investor-update` instead of writing — don't override the guard silently.

## Step 1: Resolve the Company in Notion

Search for the opportunity by company name using `notion-search` with `data_source_url: "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"`.

Extract from the opportunity page:
- **Page ID** (for later update)
- **Name** (title)
- **Status** (for the Step 0 guard — abort if portfolio-set)
- **Contact** (founder email addresses)
- **🏁 Founder(s)** (founder names from the relation, for Gmail search)
- **Existing page content** (to check for an existing Diligence Materials section)

If the company is not found in Notion, inform the user and stop. If the Status fails the Step 0 guard, abort per the guard's rules.

## Step 2: Search Gmail for Materials

Run targeted Gmail searches combining the company name, founder names, and contact emails with attachment and link signals. Use these queries (adjust based on available founder info):

1. `from:(<founder_email>) has:attachment newer_than:7d`
2. `"<company_name>" has:attachment newer_than:7d`
3. `from:(<founder_email>) ("docsend" OR "dropbox" OR "drive.google" OR "figma.com" OR "miro.com" OR "loom.com" OR "pitch.com" OR "notion.site" OR "canva.com" OR "deck" OR "memo" OR "pitch") newer_than:7d`

Use `maxResults: 10` per query. Deduplicate by message ID across queries.

For each hit, call `gmail_read_message` to confirm relevance:
- **Include**: Pitch decks, financial models, memos, one-pagers, term sheets, side letters, data room links, blurbs.
- **Exclude**: SaaS marketing, newsletters, calendar invites, receipts.

Classify each relevant email's materials into **delivery categories** (how to fetch the file) and **destination categories** (which Notion Files property to write to).

**Delivery category** (drives Step 3 sub-path):
- **Gmail attachment** (binary file attached to the email)
- **DocSend link** (URL matching `docsend.com/view/`)
- **Direct file URL** (Google Drive share link, Dropbox link, raw PDF URL)
- **Data room link** (DocSend `/view/s/` or similar multi-doc container)
- **Link-only / non-convertible** (Figma, Miro, Loom, Pitch.com, Canva, Notion.site, **Brieflink (`brieflink.com`)**, etc.) — interactive/hosted materials that cannot be cleanly downloaded or converted to PDF. These get linked as-is; the external URL is the canonical artifact.

**Destination category** (drives Step 4 chip routing — see "Property Routing" below):
- **Diligence Materials** (default) — evaluation artifacts: pitch decks, memos, one-pagers, case studies, investor updates, financial/operating models, customer references, product demos.
- **Deal Docs** — transaction artifacts: term sheets, SAFEs, convertible notes, side letters, subscription/stock-purchase agreements, pro forma cap tables, closing docs, wire instructions.

### Property Routing — Diligence Materials vs Deal Docs

Every saved artifact lands in EITHER `Diligence Materials` OR `Deal Docs`, never both. Default is Diligence Materials; route to Deal Docs only when the filename or content matches a transaction-artifact signal.

**Route to `Deal Docs` when filename or first-page text matches any of:**
- "term sheet", " TS ", "TS_", "TS-", "TS v", "TS draft" (term sheets)
- "SAFE", "convertible note", "conv note", "side letter"
- "subscription agreement", "stock purchase", " SPA ", "SPA_", "stockholder", "shareholder"
- "pro forma", "cap table", "cap_table", "captable", "ownership table"
- "closing", "closing binder", "wire instructions", "funds flow"
- Legal-doc signatures: "WHEREAS", "Per Share Price", "Liquidation Preference", "Pre-Money Valuation" appearing on the first page

**Route to `Diligence Materials` (default) for everything else** — decks, memos, one-pagers, case studies, investor updates, financial models (operating projections, NOT cap tables), customer references, product demos, etc.

**Ambiguous case** — if a doc is borderline (e.g. "Acme Round Overview.pdf" that includes both deal terms and a deck-style narrative), route to whichever signal dominates the first 2 pages. If still unclear, default to Diligence Materials and note the assumption in the Step 5 summary so Tom can re-route.

The chip-add helper (`notion_files_property.py`) takes `--prop "Deal Docs"` or `--prop "Diligence Materials"` as appropriate — same script, no per-property code needed (see `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md`).

## Step 2.5: Per-message Idempotency Gate (`claude/materials-processed`)

**Applies in all modes (B and C).** This gate is the canonical "this message was already handled" check — without it, a webhook+manual race or two delegated invocations against the same email produce duplicate Drive uploads and duplicate chips on the Opp (each upload gets a fresh fileId, so URL-based dedup at the chip-write layer misses).

**Before processing each message in the working set:**

1. Read its labels. If it carries `claude/materials-processed`, drop it from the working set — already handled. A message carrying only `claude/materials-failed` (see below) STAYS in the working set — it's the retry surface; `notion_files_property.py`'s URL-idempotency prevents duplicating the chips that already landed on the prior attempt. (Mode B: the trigger message itself is exempt only when the webhook just fired *because* of new content on that message — in practice the trigger is unlabeled by construction; do not bypass the check.)
2. After processing, keep only the messages that lack the label — the "delta set."
3. If the delta set is empty, exit cleanly: no Notion writes, no Drive uploads, no Slack alert. Log the skip with reason `already-processed`.

**After processing completes for a message** (chips written to Notion + page-body update + Step 4.5/4.6 if applicable), apply ONE outcome label to that specific message:

- **`claude/materials-processed`** — ONLY when ALL of the message's items landed: every chip written successfully, or covered by an explicit fallback notation (e.g. a logged Gmail deep-link fallback per Step 3A's error handling). Full success = processed.
- **`claude/materials-failed`** — when ANY item fatally failed (upload failed after retry, chip write exited 1, conversion broke with no fallback recorded). The failed label keeps the message visible for a retry run without re-processing the successes (chip idempotency covers those). On a later fully-successful retry, apply `claude/materials-processed`; the processed label is what the Step 2.5 read-gate keys on, so a lingering `materials-failed` label is harmless history.

```bash
/Users/tomseo/.claude/scripts/gmail-label.py --label claude/materials-processed <messageId> [<messageId> ...]
# or, on any fatal per-item failure:
/Users/tomseo/.claude/scripts/gmail-label.py --label claude/materials-failed <messageId> [<messageId> ...]
```

The helper round-trips through `gmail-webhook/label-endpoint.js`, which holds `gmail.modify` scope (the Gmail MCP does not). Multiple message IDs in one call are fine; the label is created if it doesn't exist. Label only after writes complete — labeling before writes risks marking a message processed when the run actually failed.

**Mode-specific notes:**
- **Mode B (webhook):** working set = all messages in the inbound `threadId`. Apply this gate as the first thing after Step 0 / Status Guard.
- **Mode C (manual / delegated):** working set = Step 2's Gmail search hits, deduped by messageId. Apply this gate immediately after Step 2, before any Drive uploads or processing.
- **Partial-success runs:** if some chips landed for a message and others fatally failed, apply `claude/materials-failed` (NOT `materials-processed`) — the message stays visible for retry, and the chip-write idempotency in `notion_files_property.py` protects the chips that already succeeded from duplication on the retry.

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

### 3E: Link-only / Non-convertible Materials (Figma, Miro, Loom, Pitch.com, Canva, Notion.site, Brieflink, etc.)

Use this path for interactive/hosted materials that can't be cleanly downloaded or PDF-rendered. Skip Drive entirely — the external URL itself is the canonical artifact and goes directly into both Notion locations (page body bullet AND Diligence Materials property field).

1. **Do NOT attempt to download, headless-render, or DocSend-convert these URLs.** Figma decks and Miro boards don't print usably via Chrome headless, and Loom/Pitch.com/Brieflink require auth or JS interactivity that breaks conversion. Trying wastes time and produces a broken artifact.
2. **Derive a display label** from the email subject or URL slug: `[Company] - Deck (Figma)`, `[Company] - Brainstorm (Miro)`, `[Company] - Walkthrough (Loom)`, `[Company] - Deck (Brieflink)`, etc. Keep the parenthetical platform tag — it tells a future reader why the link is external instead of Drive-hosted.
3. **Page body bullet** — see format in Step 4.
4. **Property field** — the external URL goes into the Diligence Materials Files property directly (Step 4's "always link the specific file URL" rule is relaxed for this path; see Step 4 for details).

### 3F: Product Demos (login URL + credentials)

Use this path when the source email includes a live product demo — typically an `app.<company>.<tld>` URL plus a login email and password (e.g. `https://app.inlets.ai/` + `Login: demo@inlets.ai` + `Password: Password124!`). The demo URL is the canonical artifact; credentials go inline in the chip label.

**Single chip, credentials in the label:**

1. **Do NOT download, render, or screenshot the app.** The live URL is the artifact.
2. **Display label format** — `[Company] Demo (login: <email>; pw: <password>)`. Examples:
   - `Inlets Demo (login: demo@inlets.ai; pw: Password124!)`
   - `Acme Demo (login: investor@acme.app; pw: Demo2026)`
3. **Property field** — pass the demo URL and the label above to `addLinkToFilesProperty` exactly like a Step 3E link-only material. One chip per demo.
4. **Page body** — do not duplicate the demo into the page body. The chip carries everything.
5. **Multiple credential pairs** — if the founder provides separate logins for different roles (admin/viewer/etc.), create one chip per pair with role appended: `Inlets Demo - Admin (login: ...; pw: ...)`.
6. **Demo URL with no credentials** (public sandbox / unauth'd app) — fall back to Step 3E. Label: `[Company] - Product Demo`.
7. **Detection signal** — look for an `app.*` / `demo.*` / `staging.*` URL in proximity to lines starting `Login:`, `Username:`, `User:`, `Email:`, `Password:`, `PW:`, `Pass:` (case-insensitive). Skip the path entirely if no credentials are present — that's a Step 3E case.
8. **Slack redaction** — when a demo chip is referenced in ANY Slack alert text, redact the password as `pw: ***` (login email may stay). The full credential appears only in the Notion chip label.

## Step 4: Update the Notion Opportunity Page

The canonical home for material links is the **Diligence Materials property field** (Files property chips at the top of the page). The page body stays clean and contains only the Note section plus an optional Company Blurb section (when the source email includes one) — do NOT add a `📎 Diligence Materials` body section by default.

### Page Body — Company Blurb (when source email includes one)

When a founder/sender includes a company blurb in the email body — a paragraph-level company description, typically opening with the company name + "is..." or introduced as "Here's a brief overview" / "About us" / similar — prepend a Company Blurb section to the page body (above the Note section if both exist).

**Format:**

```
*Company Blurb*

[blurb text — preserve every inline hyperlink from the source verbatim]
```

**Rules:**

- Header is `*Company Blurb*` — italicized, regular weight, no heading style.
- Body sits directly below the header.
- Preserve every inline hyperlink (e.g. `[inlets.ai](https://inlets.ai)`) exactly as written in the source — don't strip, flatten, reformat, or add tracking params.
- Include only the description paragraph(s). Skip cover prose ("Great speaking with you"), logistics ("Attached is..."), credentials ("Login:..."), conference plugs, and signoff lines.
- Place the Company Blurb section ABOVE the Note section if both exist — blurb is the higher-signal artifact for a future reader scanning the page.
- If the email body has no clear blurb (just logistics or attachments), skip this step — do not fabricate one.

**Hard preconditions — ALL must hold before writing. If any fails, skip silently and do NOT include `Company Blurb` in the Slack alert's Page Body bullet:**

1. **Idempotency — page must not already have one.** Before writing, scan the page body for an existing `*Company Blurb*` section (case-insensitive match on the literal header `*Company Blurb*` or `**Company Blurb**`). If one already exists, skip. Never overwrite, never append a second. The prior pass — usually `add-to-crm` Step 6 — owns that section; Mode B is purely additive on subsequent emails and must not rewrite history.
2. **Source must be the delta-set email body, verbatim.** The blurb text must be a contiguous extract from the `plaintextBody` (or its hyperlink-preserved HTML equivalent) of one of the delta-set messages. NOT the website, NOT the deck PDF, NOT WebFetch output from Step 4.6, NOT the existing Notion page content, NOT the model's paraphrase. If the proposed blurb is not a substring of any delta-set message body (modulo whitespace/markdown wrapping of inline links), skip.
3. **Email must contain a recognizable blurb opener.** At least one of: company name + " is " / " is a " / " is the " / " helps " / " builds " / " makes " (within the first 200 chars of a candidate paragraph), OR an explicit framing line like "About us", "Here's a brief overview", "Quick context on [company]", "What we do:". A forwarded "stepping away from X, building something new" preamble does NOT qualify — that's founder backstory, not a company description. Pure logistics ("Thanks for the intro", "grab time on my calendar", "looking forward to chatting") never qualifies.

If you skip on this gate, log the reason in the Step 5 summary (`Company Blurb: skipped — already present` / `skipped — no email-body source` / `skipped — no blurb opener detected`) so a future debug can trace the no-op.

### Page Body — DO NOT add a Diligence Materials section by default

Skip the page body entirely for materials. The property-field chips are visible at the top of every opportunity page and are the canonical surface for material links. Adding a duplicate body section just restates what's already one scroll-up, and clutters the Note section below it (see `add-to-crm/references/schema.md` — body is Note-only by default).

**Exception — only add a body section when:**
- The Note section doesn't reference the materials at all (founder attached a binary deck and didn't link it inline), AND
- There's per-material context that won't fit on the chip display label (conversion provenance like "converted from DocSend (22 pages)", or a ⚠️ status flag).

If you're writing an exception-case body section, use the bullet format below. Otherwise skip.

```
- [**[Filename]**](<URL>) — [per-material context]
```

### Notion Files Property Field — Diligence Materials OR Deal Docs (public API)

**This step always runs.** Append each saved Drive link to the appropriate Files property on the opportunity page — either **Diligence Materials** or **Deal Docs** per the routing rules in Step 2's "Property Routing" section. Term sheets, SAFEs, side letters, pro forma cap tables, etc. → Deal Docs. Decks, memos, models, demos, etc. → Diligence Materials.

Shell out to the public-API helper:

```bash
python3 ~/.claude/scripts/notion_files_property.py \
    --page-id <opportunity_page_id> \
    --prop "Diligence Materials" \
    --url "<drive_or_external_url>" \
    --label "<display_label>"
```

Exit 0 = success (including idempotent skip when URL already present); exit 1 = hard failure (log + body-only fallback). See `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md` for the full interface.

**Critical: Always link the specific file URL** (`https://drive.google.com/file/d/<fileId>/view`), never the folder URL. The file ID comes from the Drive Upload Apps Script `upload` response — use it directly, don't re-search.

**Link-only materials (Step 3E) are the explicit exception** — for Figma, Miro, Loom, Pitch.com, Canva, Notion.site etc., pass the external URL itself (e.g. `https://figma.com/deck/...`). These materials have no Drive counterpart; the external URL is the canonical artifact and MUST still be written to the property field — "no Drive URL" is not a reason to skip.

For each file saved to Drive in the preceding steps, call the helper with the opportunity page ID, the Drive file URL, and a descriptive display name that matches the PDF filename (e.g., `Chief Rebel - Week 20 Investor Update.pdf`). For link-only materials, use the external URL and a label like `Bloom - Deck (Figma)`. For multiple files, call once per URL — the helper is idempotent on URL.

Skip this step only if the helper exits 1 (hard failure). In that case, note it in the summary and continue — the page body link is the interim record.

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

## Step 4.6: Extract Round Details from Materials

After materials are saved/linked, check whether the Opp's `Round Details` property is empty. If so, mine the saved materials for round-size signals and write the result back. The classifier in `inbound-deal-detect` only sees the email body — when the founder/referrer keeps round details out of the email and tucks them into the deck (e.g. a `SEED · $2M · 2026` cover slide), Round Details ends up blank without this step.

### When to skip this step entirely

- Opp's `Round Details` is already populated — classifier or an earlier pass beat us; never clobber.
- Zero materials saved.

### How to read each material

- **PDFs available locally** (Step 3B DocSend, 3C Dropbox/raw, 3D email-body-to-PDF): re-use the `pdftotext -layout` output from Step 4.5 — same stdout, no need to re-run.
- **Gmail Attachment Saver PDFs** (Step 3A — straight to Drive): reuse the Drive-byte download from Step 4.5 if it succeeded; skip otherwise.
- **Link-only materials** (Step 3E — includes Vercel-hosted decks, static brand sites, Brieflink, Figma, Miro, Loom, Pitch, Canva, Notion.site): use **WebFetch** with the prompt `"What round size and valuation is this deck/site raising? Look for explicit fundraising terms like 'Raising $Xm' or '$Xm on $Ym cap/post' or stage-amount cover slides like 'SEED · $2M'. Quote exactly. Return 'NO_ROUND_FOUND' if nothing explicit appears."`. WebFetch is the right tool here even when Chrome is around — it works in unattended/headless contexts and is cheap. JS-only SPAs (Figma, Pitch) may return empty — that's fine, skip with a note.

**Scope of WebFetch output — Round Details ONLY.** The WebFetch response is consumable by this step and only this step. Do not feed its output back into the Step 4 Company Blurb decision (Step 4's hard precondition #2 already forbids this, but stating it here too because the temptation is real — the model has just read the deck, has the marketing copy in context, and can be tempted to "helpfully" add it as a blurb). If Step 4 ran and skipped the blurb because the email body had none, that decision stands — Step 4.6's deck read does NOT reopen it.

### What to extract

Look across the combined material text for round-size patterns. Examples that should match:

- `Raising $2m` / `Raising $2-3m`
- `$2m raise` / `$3m round` / `Seed round of $2m`
- Cover-slide stage-amount lines: `SEED · $2M · 2026`, `Pre-Seed · $1M`, `Series A — $10M`
- Finalized rounds with valuation: `$3m on $20m post`, `$2m at $15m cap`, `$5m SAFE at $30m post-money`

### How to format Round Details

Match the `inbound-deal-detect` classifier format exactly:

- **Unfinalized** (no cap/post stated): `Raising $Xm` (single number) or `Raising $X-Ym` (range). Lowercase `m`/`k`.
- **Finalized** (cap/post stated): `$Xm on $Ym post` or `$Xm on $Ym cap`.

If the deck states only stage + amount (e.g. `SEED · $2M`), reformat to `Raising $2m` — stage-amount cover slides almost always describe an open raise, not a closed one.

### How to write the update

Use `notion-update-page` with `command: "update_properties"` to set `Round Details` on the Opp page. Single string value.

If the deck also states a clearer **Stage** than what's currently on the Opp (e.g. Opp shows blank or `Pre-Seed` but the deck says `Seed`), override Stage as well. Conservative rule: only override if currently empty OR if the deck and current value disagree by exactly one stage AND the deck signal is a cover-slide title (not buried mid-deck).

### Logging

Note in the Step 5 summary: extracted (with the value written) / no-match (deck mined, nothing explicit) / skipped (Round Details already populated, or no readable materials).

## Step 4.7: Apply the Outcome Label (`claude/materials-processed` / `claude/materials-failed`)

After processing for a message completes, apply the outcome label per the rules in Step 2.5: `claude/materials-processed` only when ALL items landed (chips on the property, page body, Contact/Round Details updates if applicable — or explicit logged fallback notations), `claude/materials-failed` when any item fatally failed so the message stays retry-visible. This closes the idempotency loop in **all modes** — Mode B, Mode C manual, and delegated invocations from `pipeline-agent` / `add-to-crm`. Without this step, a follow-up run against the same Opp re-finds the same email in Gmail search and re-processes it, creating duplicate Drive uploads and duplicate chips (the canonical-filename dedup in `notion_files_property.py` catches the chip duplication, but the wasted Drive uploads remain).

## Step 5: Report Summary

Return a concise summary:

```
📎 MATERIALS HANDLER — [Company Name]

Drive folder: Diligence/[Company Name]/
Found: [N] materials across [M] emails
  - [filename1] → Diligence Materials ✅ (Gmail Attachment Saver)
  - [filename2] → Diligence Materials ✅ (Drive Upload Apps Script)
  - [filename3] → Deal Docs ✅ (term sheet routed)
  - [filename4] → Diligence Materials ✅ (email body → Chrome headless PDF)
  - [filename5] → Pending ⚠️ (upload failed, retry or manual)
  - [Figma link] → Diligence Materials ✅ (link-only, not downloadable)
  - [Demo URL] → Diligence Materials ✅ (interactive demo, creds in label)
DocSend: [N] converted and uploaded
Contact extraction: upgraded tom@old.com → tom@company.com ✅ / kept existing (custom domain) / nothing found
Round Details extraction: wrote "Raising $2m" ✅ / no-match (mined deck, nothing explicit) / skipped (already populated)
Notion: Page body updated ✅ | Diligence Materials updated ✅ | Deal Docs updated ✅ (public API) / failed ⚠️
```

## Constraints and Known Limitations

- **Gmail MCP cannot download binary attachments** — there is no attachment download endpoint. The Gmail Attachment Saver Apps Script (`/Users/tomseo/.claude/skills/shared-references/gmail-attachment-saver.md`) is the primary path for saving Gmail attachments to Drive.
- **Google Drive MCP is read-only** — `google_drive_search` and `google_drive_fetch` cannot upload files or move files between folders. All writes go through the Drive Upload Apps Script.
- **Google Drive MCP does not index PDFs** — only Google Docs appear in search results. Prefer the `fileId` / `url` returned by the Apps Script over re-searching after upload.
- **Notion Files property writes use the public API** — `~/.claude/scripts/notion_files_property.py` does the PATCH. No Chrome dependency, no token_v2 cookie, no internal endpoints.
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

### Public API is PRIMARY for Notion Files property writes

Writing external URLs to Notion Files/Media properties (Diligence Materials, Deal Docs, Online Presence, any future Files property) uses the public Notion API via `~/.claude/scripts/notion_files_property.py`. As of 2026-05-13 the public API supports external-URL writes directly — the prior internal-API / token_v2 / Chrome+osascript workarounds are obsolete and have been removed.

**How to apply:**

1. **Canonical reference**: `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md` has the helper interface and the importable Python entrypoint.

2. **One helper for all Files properties** — pass `--prop "Diligence Materials"` or `--prop "Online Presence"` or any other Files property name. The helper validates that the named property exists and is of type `files`.

3. **No Chrome, no cookies, no special headers.** Auth is the standard Notion `Authorization: Bearer <PAT>`. Token is read from `$NOTION_API_TOKEN` first, then the SOPS file at `~/code/notion-backup/.notion-token.enc.txt`.

4. **Idempotent on URL** — re-running with the same URL returns `{"skipped": true}` instead of duplicating the entry. Safe to re-run.

5. **Call per URL** — each invocation is a self-contained read-modify-write (GET page → check existing → PATCH if missing). Multiple URLs = sequential calls.

6. **Exit codes**: 0 = success (including idempotent skip), 1 = hard failure (page not found, property doesn't exist, API error). Log the failure and continue with the page-body fallback.

Consumers: this skill, first-pass-diligence, neg1-enricher (Step 4.5 Online Presence), meeting-note-processor (Artifacts), future skills needing Files property writes.
