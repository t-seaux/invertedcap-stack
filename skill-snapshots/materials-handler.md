---
name: materials-handler
description: >
  Download diligence materials (decks, memos, term sheets, investor updates) for a pipeline company, save to Google Drive, and link in Notion (page body + Diligence Materials property field). Handles Gmail attachments via Apps Script, DocSend links via Python conversion, direct file URLs, and body-only emails (investor updates, memos written inline) rendered to PDF via Chrome headless. Trigger on "save materials for [company]", "add [X] as diligence material for [company]", "download materials", "grab the deck", "save the deck", "save this investor update as a material", "materials for [company]", or any variant wanting diligence materials saved and linked to a Notion opportunity. ALSO the canonical entry point for logging a transaction/deal document to an EXISTING Opp: "log the [term sheet / side letter / SAFE / SPA / cap table] in the [company] opp", "log this in [company]", "add to [company] opp", "add this to the [company] opp", "add [X] to [company] opp", "save the executed term sheet to [company] deal docs", "add the side letter to [company]", "file this SAFE under [company]" — this skill's Step 2 Property Routing auto-sorts each artifact to the correct Files property (transaction docs → Deal Docs; everything else → Diligence Materials), so a bare "add to [company] opp" / "log this in [company] opp" is sufficient — the doc type determines the destination. Disambiguation: "add to [company] opp" targets an EXISTING opportunity (this skill); "add to crm" / "log this deal" / "add this opportunity" CREATES a new opportunity (that's add-to-crm). When the named Opp already exists and a document/attachment is in context, route here. Also triggers when pipeline-agent or add-to-crm delegates materials handling. Trigger if context involves saving email attachments, email bodies, or deck files to Drive and linking them to a deal, even without the word "materials". Always trigger inline — no confirmation needed.
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
- The Slack alert is emitted automatically by the Step 4 `--batch-json` call — do NOT send it yourself.

**Slack alert — code-enforced by the write, NOT model-executed:**

The consolidated `#claude-alerts` ping fires deterministically from inside `notion_files_property.py` whenever the `--batch-json` (or a non-`--no-alert` single `--url`) call lands ≥1 new chip on Diligence Materials / Deal Docs. This is the fix for the silent-append bug (Cline, 2026-08-25): the alert can no longer be skipped because it's a side effect of the write, not a step the model has to remember. Your only jobs are to (a) pass `--email-message-id` so the footer carries the `email` link, and (b) NOT compose or send any separate materials alert. The format the helper emits (for reference only — alert-grammar compliant: plain-subject headline, no bullet glyphs, links on the footer line):

```
🔍 <u>**Materials: {Opp Name}**</u>
**Page Body:** {comma-separated body section names — plain text, no links}
**Diligence Materials:** {comma-separated chip labels, each WRAPPED as [label](url)}
**Deal Docs:** {comma-separated chip labels, each WRAPPED as [label](url)}
[opp](https://www.notion.so/{oppId}) · [email](https://mail.google.com/mail/u/0/#all/{messageId})
```

**Alert rules:**

- Write GFM only — `send-alert/send.sh` converts to Slack Block Kit. Do NOT hand-write Slack mrkdwn (`*bold*`, `<url|text>`); it ships as literal text and breaks link tap targets.
- Header line is underlined + fully bolded with `<u>**...**</u>`, subject as plain text — NO links inside the headline (alert grammar: links live on the footer line). The footer is lowercase `[opp](notion-url) · [email](gmail-deep-link)`; the `· [email]` segment appears only when `--email-message-id` was passed.
- Field rows carry no bullet glyphs (alert grammar: no `- ` / `•`). Each row's field name is bolded with `**...**`, followed by `:`, then a comma-separated list — no per-artifact lines. Single-line spacing throughout, no blank line after the header.
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
🔍 <u>**Materials: Inlets**</u>
**Page Body:** Company Blurb
**Diligence Materials:** [Inlets - One-Pager (2026)](https://drive.google.com/file/d/.../view), [Inlets - Oncology Case Study](https://drive.google.com/file/d/.../view), [Inlets Demo (login: demo@inlets.ai; pw: ***)](https://app.inlets.ai/)
[opp](https://www.notion.so/34800beff4aa81a5ba9dca2b550eb002) · [email](https://mail.google.com/mail/u/0/#all/19dcf6ceb1af7c41)
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
7. **Upload autonomy — Drive Upload Apps Script, never ask** — use the Drive Upload Apps Script (see `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`) for every non-Gmail file: call `createFolder` to get or create the company folder under the routing-appropriate root (Step 3 target folder gate: Deal Docs–routed artifacts → `Deal Docs/[Company]/`, everything else → `Diligence/[Company]/`), then `upload` with the returned `folderId` and the base64-encoded file content. On failure, retry once, then note the failure in the summary. Do not ask Tom to upload files manually.
8. **Per-company subfolders in Diligence** — all Diligence Materials–routed artifacts for a given opportunity (NOT Deal Docs–routed ones; those follow rule 9) go into a dedicated subfolder: `Diligence/[Company Name]/`. Use the Apps Script's `createFolder` action to get-or-create the subfolder idempotently under the Diligence root (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`). Use the company name exactly as it appears in Notion (the opportunity title). When linking in Notion, link to the specific file URL whenever possible, and the company subfolder URL as a fallback.
9. **Deal Docs go to the canonical top-level `Deal Docs/` store — FLAT, not the Diligence tree** — anything routed to the Notion `Deal Docs` property (term sheets, SAFEs, SPAs, voting agts, IRA/ROFR/co-sale, stockholder consents, cert of incorp, wire SSI, pro forma cap tables, closing binders) goes into `Deal Docs/[Company Name]/` under the canonical Deal Docs root (`1mKStCJl9YKXObL4bBWBFjgfWxYj0vDwN`) — NOT under `Diligence/…`. Get-or-create the company folder idempotently with `createFolder` passing `parentId = 1mKStCJl9YKXObL4bBWBFjgfWxYj0vDwN`, then upload the file directly into it. **Flat by default (Tom, 2026-08-21): no round subfolder.** Only once a company has raised a NEW round do deal docs break into `<Stage> (<Mon YYYY>)` round subfolders (and at that transition the first round's docs bucket into their own round folder too). Diligence Materials chips continue to land directly in `Diligence/[Company Name]/`. Keep one copy of each distinct version in Drive (older versions/redlines stay for audit) but no byte-for-byte duplicates; the Notion `Deal Docs` property holds only the latest version of each doc. See the deal-docs layout memory.
10. **Every saved material's filename carries its SENT date (Tom, 2026-09-14).** Canonical convention: `[Company Name] - [Descriptive Title] MM.DD.YY.pdf` — the date appended at the end of the name, before the extension (e.g. `Paravel Health - Next Steps Email 09.14.26.pdf`, `Paravel Health - Deck 09.08.26.pdf`). The date is when the material was SENT to Tom — the email's internal date for attachments and body PDFs; for converted links (DocSend, Papermark, direct URLs), the date of the email that delivered the link. NOT the processing date — a run that catches up on a week-old email stamps the email's date. Chip display labels match the filename exactly. Applies to every Drive-hosted artifact (3A, 3B, 3C, 3D, 3G) on both Diligence Materials and Deal Docs routing, AND to link-only chip labels (3E/3F/3H — Figma, demos, videos): the chip label ends with the sent date, e.g. `Bloom - Deck (Figma) 09.14.26`, `Inlets Demo (login: demo@inlets.ai; pw: Password124!) 09.14.26` — Tom wants the date he was sent every material, live links included. The only undated chips are infrastructure (the `[G DRIVE]` folder pin) and diligence-output snapshots (`_Master_Diligence_*`), which keep their own established naming.
11. **Pin a Drive-folder chip at the top of Diligence Materials.** Every Opp's Diligence Materials property carries a permanent first chip linking to its whole Drive subfolder, labeled `[G DRIVE] [Company Name] Diligence Materials` and pointing at `https://drive.google.com/drive/folders/<company subfolder id>`. This gives one click to the full materials folder — including anything not individually chipped — without disturbing the per-file chips below it. Check for it (by folder URL) before any new chips are added on a run; if missing, add it first via `--prepend` so it leads the list (subsequent default-append chips then naturally land after it). See Step 4.

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
- **Email body with standalone substance** — the body itself qualifies as a material even when the email ALSO carries attachments; this is a judgment call, not an explicit-ask-only path (Tom, 2026-09-14). Route to Step 3D when the body contains content a future diligence pass would cite: linked research reports / market studies, inline metrics or traction updates, a written thesis or market narrative, a founder-curated reading list. Skip when the body is only cover text for the attachments ("attached is our deck"), scheduling, or pleasantries. Canonical yes: Paravel Health's "Next Steps" email (seven linked market reports framing the founder's market view). Canonical no: "here's the deck, happy to chat." Applies in every mode — a Mode B webhook run saving an attachment should ALSO save the body when it passes this bar.
- **DocSend link** (URL matching `docsend.com/view/`)
- **Direct file URL** (Google Drive share link, Dropbox link, raw PDF URL)
- **Data room link** (DocSend `/view/s/` or similar multi-doc container)
- **Papermark deck** (URL matching `papermark.com/view/` or `*.papermark.io/view/`) — email-gated image deck. **Convert to a Drive PDF** via the pure-HTTP extractor (Step 3G), NOT linked as-is. Works headless — no GUI/webhook fallback needed anymore (unless the link is password/agreement-gated).
- **Video link** (YouTube `youtu.be`/`watch`, Loom, Vimeo — demo walkthroughs, founder videos) — **Step 3H**. Linked verbatim as a chip AND ripped to a transcript note tagged to the Opp. Do not treat a video as a plain link-only material — it gets the extra transcript step.
- **Link-only / non-convertible** (Figma, Miro, Pitch.com, Canva, Notion.site, **Brieflink (`brieflink.com`)**, etc.) — interactive/hosted materials that cannot be cleanly downloaded or converted to PDF. These get linked as-is; the external URL is the canonical artifact.
  - **Pass the URL to `notion_files_property.py` byte-for-byte as it arrived — copy-paste from the source, never retype or reconstruct it.** Do NOT normalize, shorten, or convert between equivalent forms (e.g. `youtu.be/<id>` → `youtube.com/watch?v=<id>`, stripping `?feature=shared`, canonicalizing a Figma/Loom path). These rewrites are where video/share IDs get silently truncated and the link dies — Cline's demo was filed as `watch?v=9wKiITaLA` when the founder sent `youtu.be/069wKiITaLA`, dropping two chars (Tom, 2026-08-26). The founder's original string is the only safe input; if you can't copy it verbatim, don't file it.

**Destination category** (drives Step 4 chip routing — see "Property Routing" below):
- **Diligence Materials** (default) — evaluation artifacts: pitch decks, memos, one-pagers, case studies, investor updates, financial/operating models, customer references, product demos.
- **Deal Docs** — transaction artifacts: term sheets, SAFEs, convertible notes, side letters, subscription/stock-purchase agreements, pro forma cap tables, closing docs, wire instructions.

### Property Routing — Diligence Materials vs Deal Docs

Every saved artifact lands in EITHER `Diligence Materials` OR `Deal Docs`, never both. Default is Diligence Materials; route to Deal Docs only when the filename or content matches a transaction-artifact signal.

**Route to `Deal Docs` when filename or first-page text matches any of:**
- **Term sheets:** "term sheet", "term-sheet", "termsheet", " TS ", "TS_", "TS-", "TS v", "TS draft", "[EXECUTED]"/"[FINAL]" prefixing any of the above, MyCase/DocuSign/Dropbox-Sign "Completed:" e-sign subjects
- **Investment instruments:** "SAFE" (incl. "post-money SAFE", "pre-money SAFE", "SAFE agreement"), "convertible note", "conv note", "KISS", "promissory note"
- **Side agreements:** "side letter", "MFN", "most favored nation", "pro rata letter", "pro-rata side letter", "information rights"
- **Purchase/rights agreements:** "subscription agreement", "stock purchase", "purchase agreement", " SPA ", "SPA_", "SPA-", "stockholder", "shareholder", "voting agreement", "IRA" / "investor rights agreement", "ROFR", "right of first refusal", "co-sale", "registration rights"
- **Corporate/closing docs:** "stockholder consent", "board consent", "written consent", "certificate of incorporation", "cert of incorp", "COI", "bylaws", "closing", "closing binder", "closing set", "wire instructions", "wire SSI", "funds flow", "flow of funds", "capitalization", "cap table", "cap_table", "captable", "pro forma" (cap table), "ownership table"
- **Legal-doc signatures on the first page:** "WHEREAS", "Per Share Price", "Liquidation Preference", "Pre-Money Valuation", "This SAFE", "in consideration of", signature/notary blocks

**Named-type shortcut (highest confidence).** When Tom names the doc type in his instruction — "log the **term sheet** / **side letter** / **SAFE** / **SPA** / **cap table** in the [company] opp" — that stated type IS the routing signal; route to Deal Docs without needing a filename or first-page match. An explicit type in the ask always wins over filename heuristics.

**Deal Docs are stored as received — never mint a derived copy (Tom, 2026-08-25).** For anything routed to `Deal Docs`, the file as it arrived IS the artifact. Do not transcribe it into a Google Doc, re-render it as "machine-readable" text, or otherwise create a second copy — and never drop one into `Diligence/[Company]/`, where later diligence passes and memo drafts will read it as source material. Derived copies drift from the original, and plaintext strips the visual cues that make a transaction doc verifiable. Wire instructions are the acute case: a Doc reads as authoritative but carries none of the bank's formatting, so a tampered version is indistinguishable from a real one. If you need the contents to reason, read the original in-context and leave nothing behind.

**Route to `Diligence Materials` (default) for everything else** — decks, memos, one-pagers, case studies, investor updates, financial models (operating projections, NOT cap tables), customer references, product demos, etc.

**Ambiguous case** — if a doc is borderline (e.g. "Acme Round Overview.pdf" that includes both deal terms and a deck-style narrative), route to whichever signal dominates the first 2 pages. If still unclear, default to Diligence Materials and note the assumption in the Step 5 summary so Tom can re-route.

The chip-add helper (`notion_files_property.py`) takes `--prop "Deal Docs"` or `--prop "Diligence Materials"` as appropriate — same script, no per-property code needed (see `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md`).

## Step 2.5: Per-message Idempotency Gate (`claude/materials-processed`)

**Applies in all modes (B and C).** This gate is the canonical "this message was already handled" check — without it, a webhook+manual race or two delegated invocations against the same email produce duplicate Drive uploads and duplicate chips on the Opp (each upload gets a fresh fileId, so URL-based dedup at the chip-write layer misses).

**Before processing each message in the working set:**

1. Read its labels. **Gmail returns opaque label IDs (`Label_6`, `Label_15`), NEVER names — you cannot see `claude/materials-processed` in `labelIds` directly.** Resolve IDs → names via `list_labels` (one call, cache the map for the whole run) BEFORE comparing; comparing the name string against raw `labelIds` always misses and silently passes the gate. (2026-09-10 Ardent incident: the pipeline-materials sweep read `labelIds: ["STARRED","INBOX","Label_6"]`, concluded "no materials-processed label," and re-processed a message the webhook had correctly labeled 6 hours earlier — recreating a duplicate an interactive cleanup had removed minutes before.) If the label map can't be fetched, treat the gate as FAILED for the run — skip processing, don't guess. If it carries `claude/materials-processed`, drop it from the working set — already handled. A message carrying only `claude/materials-failed` (see below) STAYS in the working set — it's the retry surface; `notion_files_property.py`'s URL-idempotency prevents duplicating the chips that already landed on the prior attempt. (Mode B: the trigger message itself is exempt only when the webhook just fired *because* of new content on that message — in practice the trigger is unlabeled by construction; do not bypass the check.)
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

**`claude/materials-processing` — the in-flight claim (added 2026-08-04).** Apply this label **immediately before Step 3's first Drive upload**, and drop it when the terminal outcome label goes on in Step 4.7.

- It does **NOT** satisfy the read-gate in point 1. A message carrying only `materials-processing` **stays in the working set**, exactly like `materials-failed`. That preserves the rule above: a run that died mid-flight must never look successful.
- What it changes is the *posture* of the next run. `materials-processing` present without `materials-processed` means a prior attempt died between the first upload and the outcome label. Before uploading, **check Drive for the files this run would create and read the Opp's existing chips**, then upload only what is genuinely missing and repair any stale chip per Step 4.7. Do not blind re-upload.

> **Why a third label instead of moving the existing one.** Drive uploads (Step 3) are non-idempotent and land minutes before the only durable record (the outcome label, Step 4.7) — the gap spans conversion and Chrome work. Moving the outcome label earlier would close the duplicate window but reintroduce precisely the bug this skill already guards against: marking a failed run processed. A separate in-flight label makes "started" and "finished" independent facts, so a dead run leaves a tombstone that triggers verification rather than either a blank (duplicate work) or a false success (lost work). This skill has no sweep of its own, but `add-to-crm` Step 6 and `pipeline-agent` Task 5 both re-find unlabeled messages by Gmail search — a second producer is always live.

> **Residual race this gate does NOT close on its own (root-caused 2026-08-24, Fair opp).** This label check reads state at the START of a run; nothing stops two Mode B jobs for the SAME `threadId` from being *launched* close enough together that both read "nothing labeled yet" before either writes. That's what happened: `materials-detect.js` legitimately fired on two different messages in one thread seconds apart (each got its own `materials-handler-{messageId}` idempotency key, so the D1 queue correctly did NOT dedup them — by design, this label gate is what's supposed to arbitrate the second one), the local processor leased and launched both in the same batch, and both independently uploaded and chipped the same attachment before either had a chance to label it. Fixed at the queue-processor layer, not here: `~/.claude/local-agents/claude-job-queue-processor/processor.py` (`AFFINITY_SKILLS`) now serializes Mode B materials-handler jobs by `threadId` — a second job for a thread already in flight is held locally and retried once the first clears, so in practice this gate only ever has to arbitrate strictly-sequential runs. That guard covers Mode B (webhook) only. Mode C / delegated invocations (`pipeline-agent` Task 5, `add-to-crm` Step 6) don't route through the queue-processor's lease/launch path and could in principle still race a live Mode B run for the same thread — narrow, unclosed edge case; no incident of it yet.

**Mode-specific notes:**
- **Mode B (webhook):** working set = all messages in the inbound `threadId`. Apply this gate as the first thing after Step 0 / Status Guard.
- **Mode C (manual / delegated):** working set = Step 2's Gmail search hits, deduped by messageId. Apply this gate immediately after Step 2, before any Drive uploads or processing.
- **Partial-success runs:** if some chips landed for a message and others fatally failed, apply `claude/materials-failed` (NOT `materials-processed`) — the message stays visible for retry, and the chip-write idempotency in `notion_files_property.py` protects the chips that already succeeded from duplication on the retry.

## Step 3: Process Materials

**Target folder gate (run BEFORE any 3A–3D upload).** The Drive destination follows the artifact's Step 2 Property Routing — two stores, never mixed:

| Property routing | Drive parent root | Target folder |
|---|---|---|
| **Deal Docs** (term sheets, SAFEs, side letters, SPAs, wire SSI, cap tables, closing docs…) | `1mKStCJl9YKXObL4bBWBFjgfWxYj0vDwN` (top-level `Deal Docs/`) | `Deal Docs/[Company Name]/` — flat; round subfolders only for multi-round companies (rule 9) |
| **Diligence Materials** (everything else — decks, memos, updates…) | `1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d` (`Diligence/`) | `Diligence/[Company Name]/` |

Get-or-create the company folder idempotently via the Drive Upload Apps Script `createFolder` with the routing-appropriate `parentId` from the table. A mixed email (deck + term sheet) resolves the gate **per attachment**, not per email. Wherever a step below says "the target folder," it means the folder this gate selected.

### 3A: Gmail Attachments (Apps Script)

Use the Gmail Attachment Saver Apps Script to save attachments directly to the target folder on Google Drive. No Chrome required.

Read the reference at `/Users/tomseo/.claude/skills/shared-references/gmail-attachment-saver.md` for the deployment URL and full API details.

For each email containing relevant attachments:

1. **Determine the target Drive folder ID** via the Step 3 target folder gate (`createFolder` with the routing-appropriate parent). If the Apps Script is unavailable, fall back to the routing-appropriate ROOT folder ID from the gate table so the file still lands in the correct tree. Note the Gmail Attachment Saver takes ONE `driveFolderId` per call — for a mixed email, save to the majority destination, then move the minority files to their correct folder via the Drive MCP `update_file` (parentId move preserves the file ID).

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

5. **Rename to convention + dedup guard (MANDATORY — this path is the one that accumulates duplicates).** Unlike the Drive Upload Apps Script (3B–3D), the Gmail Attachment Saver keeps the attachment's **original filename verbatim and never trashes-and-replaces** — so a founder attachment literally named `Memo.pdf` lands as `Memo.pdf`, and every re-run (webhook + manual + delegated `add-to-crm` Step 6 / `pipeline-agent` Task 5) mints a *new* `Memo.pdf` with a fresh fileId. URL-idempotency at the chip layer can't catch it (new fileId = new URL), so the copies silently pile up. Close it here, per saved file:
   1. **Rename to the same convention as 3B–3D:** `[Company Name] - [Descriptive Title] MM.DD.YY.pdf`, date = the email's sent date per principle 10 (e.g. `Memo.pdf` → `Ardent - Founder Memo 09.10.26.pdf`, `deck.pdf` → `Ardent - Deck 09.10.26.pdf`). Derive the title from the attachment name / email subject; strip a redundant leading company name. This kills bare generic names, makes collisions detectable, and prevents cross-company `Memo.pdf` clashes. Batch via `drive_rename.py`:
      ```bash
      echo '[{"fileId":"<newFileId>","newName":"Ardent - Founder Memo.pdf"}]' \
          | python3 ~/.claude/scripts/drive_rename.py --batch
      ```
      **A failed rename is a fatal per-item failure** (label `claude/materials-failed`, no chip) — never fall back to chipping the raw-named file under a convention label. That fallback is what the 2026-09-10 Ardent run did when `drive_rename.py` crashed under the system python (`ModuleNotFoundError: google` — since fixed with a re-exec guard inside the script): it left a raw `Memo.pdf` in Drive, a chip labeled `Ardent - Founder Memo.pdf` pointing at it, and — because step 2 below keys on the *renamed* filename — the older duplicate survived uncollapsed. A failed rename means the dedup guard cannot run; fail loud instead of silently recreating the duplicate state.
   2. **Collapse older duplicates:** `listFolder` the target folder (Drive Upload Apps Script, `drive-upload.md` §4). If the just-renamed convention name now matches one or more **older** files (lower `createdTime`) in that folder, they are prior copies of this same artifact — trash all but the newest via `drive_rename.py --trash --confirm --file-id <olderId>`. This is the byte-identical same-name replace the Drive Upload script does silently; scope the trash strictly to an **exact convention-name match that is older than the file this run just wrote** — never a different-named file. (This is the one auto-trash exempt from the confirm-with-Tom rule, because it only ever removes a same-name older copy of the file just re-saved — identical to the Drive Upload replace path already treated as "safe and idempotent." Any broader cleanup still needs Tom's OK.)

Deterministic convention names + this list-and-trash step give 3A the same re-run idempotency 3B–3D already get from the Drive Upload endpoint. (Underlying trap for the record: the two Apps Scripts behave differently — Drive Upload trashes-and-replaces on same name, the Gmail Attachment Saver does not. If that endpoint ever gains a trash-and-replace mode, this guard becomes redundant.)

### 3B: DocSend Links (No Chrome Needed)

**Pre-converted handoff:** if the caller (add-to-crm Step 6) passed a local PDF path already converted from this DocSend URL, skip steps 1–3 below and jump straight to naming (if needed) + upload with that file — never re-convert a URL the caller already converted.

Follow the `docsend-to-pdf` skill at `/Users/tomseo/.claude/skills/docsend-to-pdf/SKILL.md` for the exact Python conversion approach:

1. Use the `requests` + `Pillow` method to convert the DocSend document to PDF.
2. Name the file using the DocSend `<meta>` title: `[Company Name] - [Document Title] MM.DD.YY.pdf` (date = when the link was sent, per principle 10). Strip redundant company name if present in the title. Fallback: `[Company Name] - Deck MM.DD.YY.pdf`.
3. Save to `/Users/tomseo/Downloads/[filename].pdf`.
4. Present to user via `present_files`.
5. **Upload to the target folder via Drive Upload Apps Script**: See `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`. First run the Step 3 target folder gate (`createFolder` with the routing-appropriate parent — decks are Diligence Materials, so normally `Diligence/[Company]/`), then call `upload` with the returned `folderId` and the base64-encoded file content. On success, use the returned `fileId` and `url` directly — no separate Drive MCP search needed. On failure, retry once, then note the failure in the summary but do not ask Tom to upload manually.
6. **Use the URL returned by the Apps Script**: The `upload` response contains `fileId` and `url` (`https://drive.google.com/file/d/<fileId>/view`). Use this link directly in the Notion page body — no separate Drive MCP lookup needed.

For **data room URLs** (`/view/s/`), follow the docsend-to-pdf skill's data room handling to extract individual document URLs first, then convert each.

### 3C: Direct File URLs

**A PDF in our own Drive folder is always the preferred chip. Never chip a founder's live link when a PDF snapshot is obtainable** — the founder can unshare, edit, or delete it at any time, and the chip rots silently. Step 4.4 does NOT rescue this case: it only fires as a consequence of adding a PDF chip, so a native link filed here without conversion never gets superseded.

- **Native Google Docs / Slides** (`docs.google.com/document/...`, `docs.google.com/presentation/...`) — **convert, don't link.** Even when the link is a clean Drive share URL:
  1. `files.export` to `application/pdf` → `Diligence/[Company Name]/[Company] - [Title].pdf`
  2. `files.export` to the original Office format (`...wordprocessingml.document` for Docs, `...presentationml.presentation` for Slides) → same folder, durable archive of the editable original
  3. Chip **only the PDF**. Do not chip the `docs.google.com` URL.
  Export is read-only against the founder's file — never modify or trash their original. Exempt: Tom-authored Docs meant to stay editable (e.g. Diligence Q&A), which keep their native chip.
- **Native Google Sheets** — never superseded, but still get a PDF snapshot alongside the live chip. See `~/.claude/skills/shared-references/spreadsheet-artifact-convention.md`.
- **Binary files on Drive** (a PDF/`.pptx`/`.xlsx` the founder uploaded, `drive.google.com/file/d/...`) — download the bytes and re-upload into `Diligence/[Company Name]/` via the Drive Upload Apps Script, then chip our copy. Chip the founder's URL directly only if the download fails.
- **Dropbox or raw PDF URLs**: Use `web_fetch` or `curl` to download the file, save to `/Users/tomseo/Downloads/`. Then upload to the target folder (Step 3 gate) using the Drive Upload Apps Script — same `createFolder` → `upload` pattern as 3B. The Apps Script returns `fileId` and `url` directly.

Drive v3 export/download runs through the `gmail-reconciler` service account with DWD as tom@invertedcap.com — `from drive_rename import _service` in `~/.claude/scripts/` gives an authenticated `drive` client.

### 3D: Email Body → PDF (Chrome Headless)

Use this path when the email *body itself* is the material. Two entry routes: (a) explicit — Tom says "save this email as a diligence material for [company]", or the email is a body-only investor update / inline memo; (b) **judgment call** — the body passes the "standalone substance" bar defined in Step 2's delivery categories (linked research reports, inline metrics, market narrative, reading list), even when the email also has attachments being saved via 3A. Route (b) needs no ask — save it and note the judgment in the Step 5 summary.

1. Fetch the full message content via `gmail_get_thread` (`messageFormat: FULL_CONTENT`) to get `htmlBody`, `plaintextBody`, subject, sender, and date. Prefer `htmlBody` as the render source when the body carries inline hyperlinks — the plaintext form mangles anchor text and URLs.
2. Render an HTML file at `/Users/tomseo/Downloads/<slug>.html` that wraps the body in a clean layout: title (subject), a meta line (company · sender · date · subject), and the body content. Preserve sections and bullets from the source — don't invent structure. **Preserve every inline hyperlink clickable, and when the body cites external links (reports, studies, articles), append a "Linked references" section listing each link's full URL spelled out** — anchor text alone dies in print, and those URLs are exactly what a later diligence pass clicks through. Use the standard print-friendly CSS (`@page { size: Letter; margin: 0.75in; }`, `-apple-system` font stack, 12pt body, bordered header).
3. Shell out to Chrome headless to convert to PDF:
   ```bash
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
     --headless --disable-gpu --no-pdf-header-footer \
     --print-to-pdf="/Users/tomseo/Downloads/<Filename>.pdf" \
     "file:///Users/tomseo/Downloads/<slug>.html"
   ```
4. Name the PDF `[Company Name] - [Descriptive Label] MM.DD.YY.pdf` (date = the email's sent date, per principle 10). For investor updates, derive the label from the subject (e.g. "Week 20 Investor Update"). Don't include emojis or special punctuation in the filename.
5. Upload to the target folder (Step 3 gate — investor updates/inline memos are Diligence Materials → `Diligence/[Company]/`) via the Drive Upload Apps Script (same `createFolder` → `upload` pattern). Use the returned `fileId` / `url` for Notion linking.

### 3E: Link-only / Non-convertible Materials (Figma, Miro, Loom, Pitch.com, Canva, Notion.site, Brieflink, etc.)

Use this path for interactive/hosted materials that can't be cleanly downloaded or PDF-rendered. Skip Drive entirely — the external URL itself is the canonical artifact and goes directly into both Notion locations (page body bullet AND Diligence Materials property field).

1. **Do NOT attempt to download, headless-render, or DocSend-convert these URLs.** Figma decks and Miro boards don't print usably via Chrome headless, and Loom/Pitch.com/Brieflink require auth or JS interactivity that breaks conversion. Trying wastes time and produces a broken artifact.
2. **Derive a display label** from the email subject or URL slug, ending with the sent date per principle 10: `[Company] - Deck (Figma) MM.DD.YY`, `[Company] - Brainstorm (Miro) MM.DD.YY`, `[Company] - Walkthrough (Loom) MM.DD.YY`, `[Company] - Deck (Brieflink) MM.DD.YY`, etc. Keep the parenthetical platform tag — it tells a future reader why the link is external instead of Drive-hosted.
3. **Page body bullet** — see format in Step 4.
4. **Property field** — the external URL goes into the Diligence Materials Files property directly (Step 4's "always link the specific file URL" rule is relaxed for this path; see Step 4 for details).

### 3F: Product Demos (login URL + credentials)

Use this path when the source email includes a live product demo — typically an `app.<company>.<tld>` URL plus a login email and password (e.g. `https://app.inlets.ai/` + `Login: demo@inlets.ai` + `Password: Password124!`). The demo URL is the canonical artifact; credentials go inline in the chip label.

**Single chip, credentials in the label:**

1. **Do NOT download, render, or screenshot the app.** The live URL is the artifact.
2. **Display label format** — `[Company] Demo (login: <email>; pw: <password>) MM.DD.YY` (sent date last, per principle 10). Examples:
   - `Inlets Demo (login: demo@inlets.ai; pw: Password124!) 09.14.26`
   - `Acme Demo (login: investor@acme.app; pw: Demo2026) 09.14.26`
3. **Property field** — pass the demo URL and the label above to `addLinkToFilesProperty` exactly like a Step 3E link-only material. One chip per demo.
4. **Page body** — do not duplicate the demo into the page body. The chip carries everything.
5. **Multiple credential pairs** — if the founder provides separate logins for different roles (admin/viewer/etc.), create one chip per pair with role appended: `Inlets Demo - Admin (login: ...; pw: ...)`.
6. **Demo URL with no credentials** (public sandbox / unauth'd app) — fall back to Step 3E. Label: `[Company] - Product Demo`.
7. **Detection signal** — look for an `app.*` / `demo.*` / `staging.*` URL in proximity to lines starting `Login:`, `Username:`, `User:`, `Email:`, `Password:`, `PW:`, `Pass:` (case-insensitive). Skip the path entirely if no credentials are present — that's a Step 3E case.
8. **Slack redaction** — when a demo chip is referenced in ANY Slack alert text, redact the password as `pw: ***` (login email may stay). The full credential appears only in the Notion chip label.

### 3G: Papermark Decks (`papermark.com/view/…`) — capture to PDF, don't link as-is

Papermark viewers are email-gated image decks whose share links expire. Snapshot the pages into a durable Drive PDF instead of linking the gated URL. **Run the pure-HTTP extractor:**

```bash
python3 ~/.claude/scripts/papermark_extract.py "<papermark_url>" --out-dir /tmp/<co>_pm --json --quiet
```

It reverse-engineers Papermark's view API (`/api/views` + `/api/views/pages`) — **no browser, no screen capture, works headless and in webhook/scheduled runs**, correct for docs of any length. Auto-passes the email gate with `tom@invertedcap.com` (sanctioned for Papermark viewers per Tom 2026-08-11). Exit 0 = PDF written (path in the `--json` output); exit 4 = blocked (password/agreement — pass `--password`, else flag for interactive capture); exit 2/5 = incomplete/parse error. Full contract + fallbacks in `/Users/tomseo/.claude/skills/shared-references/papermark-deck-capture.md`.

Then upload the PDF to `Diligence/<Company>/` via `drive-upload.md`, link the Drive file URL in Diligence Materials via `add-link-to-files-property.md`, and `--remove` any prior Papermark chip (the PDF supersedes the gated link).

**Only a password- or agreement-gated link (exit 4) still needs a fallback** — pass `--password`, or in an unattended run link the Papermark URL as-is (Step 3E) and flag `⚠️ Papermark deck linked as-is — password/NDA gated, re-capture interactively`.

### 3H: YouTube / Video Links (demo walkthroughs, founder videos) — link verbatim AND log a transcript note

Any YouTube (`youtube.com/watch`, `youtu.be/…`), Loom, Vimeo, or other hosted-video URL that arrives as diligence material gets **two** things, not one:

1. **File the link as a chip — verbatim (Step 3E rules apply).** The video URL is the canonical artifact; it goes straight into Diligence Materials via `add-link-to-files-property.md`. **Pass the URL byte-for-byte as the founder sent it — never retype, normalize, or convert between forms** (`youtu.be/<id>` → `watch?v=<id>`, stripping `?feature=shared`, etc.). That rewrite is how a video ID gets silently truncated and the chip dies (Cline demo filed as `watch?v=9wKiITaLA` when the founder sent `youtu.be/069wKiITaLA` — Tom, 2026-08-26). Label: `[Company] - <Video Title> (YouTube) MM.DD.YY` or `[Company] Demo (YouTube) MM.DD.YY` (sent date per principle 10).
2. **Rip a transcript and create a Notes-DB entry tagged to the Opp.** Delegate to the **`log-transcript-to-notion`** skill (read `/Users/tomseo/.claude/skills/log-transcript-to-notion/SKILL.md`), passing the video URL and the resolved Opportunity page URL so the note's `Opportunity` relation is set. That skill rips the captions with `yt-dlp`, builds the note (Source / Speaker / Summary + Frameworks + Transcript), sets the `:claude-color:` icon, and runs `note-classifier` (an Opp-linked pipeline note classifies as **Diligence**). Note title follows that skill's format, e.g. `Transcript: <Company> Product Demo — <Company> (Mon DD, YYYY)`.

**yt-dlp reality check (learned on the Cline rip, 2026-08-26).** Modern YouTube blocks yt-dlp's default path. The recipe that works for unlisted founder demos:
- A JS runtime is required — `deno` (installed via `brew install deno`). Without it, subtitle PO-token minting fails and captions come back empty.
- Use the `ios` player client and tolerate the missing video format (we only want captions):
  ```bash
  yt-dlp --no-check-certificate --no-update --ignore-no-formats-error \
    --extractor-args "youtube:player_client=ios" \
    --write-sub --write-auto-sub --sub-lang "en.*" --skip-download \
    --sub-format "srt/vtt/best" --convert-subs srt \
    -o "/tmp/<co>_demo" "<verbatim video URL>"
  ```
  `--ignore-no-formats-error` is essential — otherwise yt-dlp aborts on "Requested format is not available" (the video exposes only image/storyboard formats to non-JS clients) *before* it writes the subtitle file. Then clean the `.srt` with the dedup pass in `log-transcript-to-notion` Step 1.
- If captions genuinely don't exist (the metadata shows no manual subs and `has no automatic captions`), file the chip per (1), note `⚠️ no captions available — transcript not logged` in the Step 5 summary, and move on. Do not block the chip write on the transcript.

Both outputs are independent — a failed transcript rip never blocks the chip, and vice versa.

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

**Spreadsheets get TWO chips.** Any Google Sheet or `.xlsx` (financial model, plan, cap-table workbook) is chipped as a PDF snapshot *and* its live/native source, both keeping the source file's own name — read `~/.claude/skills/shared-references/spreadsheet-artifact-convention.md` for the naming and the native `files.export` path.

**Pinned Drive-folder chip (Diligence Materials only, once per company).** Before adding any other chip on this run, check whether a chip pointing at the company's Diligence subfolder URL (`https://drive.google.com/drive/folders/<folderId>`, the same `folderId` returned by Step 3's `createFolder` call) already exists on Diligence Materials. If not, add it first:

```bash
python3 ~/.claude/scripts/notion_files_property.py \
    --page-id <opportunity_page_id> \
    --prop "Diligence Materials" \
    --url "https://drive.google.com/drive/folders/<folderId>" \
    --label "[G DRIVE] [Company Name] Diligence Materials" \
    --prepend --no-alert
```

`--prepend` only matters the first time — once the chip exists, every later run's idempotency check (URL match) skips it, and normal appended chips already land after it. Never apply this to Deal Docs. **`--no-alert` is required here** — the folder-pin is infrastructure, not a material; it must never appear in the consolidated ping (see Step 5).

**Write all real materials in ONE batch call** — this fires the single consolidated `#claude-alerts` ping automatically (see Step 5). After the folder-pin, collect every material saved in the preceding steps into a `--batch-json` array of `{prop, url, label}` items (mix Diligence Materials and Deal Docs freely — the helper groups them in the alert) and make one call:

```bash
python3 ~/.claude/scripts/notion_files_property.py \
    --page-id <opportunity_page_id> \
    --batch-json '[
      {"prop":"Diligence Materials","url":"<file_url>","label":"<display_label>"},
      {"prop":"Deal Docs","url":"<file_url>","label":"<display_label>"}
    ]' \
    --email-message-id "<gmail_message_id>"
```

- `--email-message-id` adds the `(Email)` deep-link to the ping header — pass the trigger message's Gmail ID whenever available (Mode B always has it).
- The helper adds each item (idempotent on URL, preservation-safe), then fires ONE consolidated ping listing only the chips that **newly landed** on Diligence Materials / Deal Docs. Items that skip (already present) are silently excluded — a re-run that adds nothing new sends nothing. **Do not compose or send a materials alert yourself** — the batch call owns it (this is why Step 5's alert is code-enforced, not model-executed).
- Exit 0 = success (including idempotent skips); a per-item hard failure is reported in the `results` array with exit still 0 unless *every* item failed. See `/Users/tomseo/.claude/skills/shared-references/add-link-to-files-property.md` for the full interface.
- Single-add form (`--url`/`--label`/`--prop`) still works and also auto-pings for these two props unless `--no-alert` is passed — but for a materials drop always prefer the batch call so Tom gets ONE message, not one per file.

**Critical: Always link the specific file URL** (`https://drive.google.com/file/d/<fileId>/view`), never the folder URL. The file ID comes from the Drive Upload Apps Script `upload` response — use it directly, don't re-search.

**Link-only materials (Step 3E) are the explicit exception** — for Figma, Miro, Loom, Pitch.com, Canva, Notion.site etc., pass the external URL itself (e.g. `https://figma.com/deck/...`). These materials have no Drive counterpart; the external URL is the canonical artifact and MUST still be written to the property field — "no Drive URL" is not a reason to skip.

Give each file a descriptive display name that matches the PDF filename, including its sent date (e.g., `Chief Rebel - Week 20 Investor Update 09.02.26.pdf`); for link-only materials, use the external URL and a dated label like `Bloom - Deck (Figma) 09.14.26`. Assemble all of them into the single `--batch-json` call above — one call per drop, not one per file — so the consolidated ping lists them together.

Skip this step only if the batch call reports every item failed. In that case, note it in the summary and continue — the page body link is the interim record.

## Step 4.4: Materials Hygiene — PDF Snapshot Supersedes Native / Link-Only Chip

**Trigger:** this run adds a PDF chip to Diligence Materials for an artifact that ALREADY has a chip on the same property pointing at one of:
- a native Google Doc/Slides link (`docs.google.com/document/...`, `docs.google.com/presentation/...`), or
- a hosted-viewer link (Papermark, DocSend, Brieflink, or similar — matches the Step 3E link-only patterns).

...for the **same underlying content** (same title/topic — e.g. the PDF is a snapshot export of that exact Doc, or a converted download of that exact DocSend/Papermark link).

**Action, in this order:**

1. **If the superseded chip is a native Google Doc/Slides/Sheet** owned by the founder (not Tom), first archive a durable copy into the company's Diligence subfolder in its *original* format — export via the Drive v3 API (`files.export`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` for Docs, `...presentationml.presentation` for Slides) and upload the bytes to `Diligence/[Company Name]/` via the Drive Upload Apps Script, same as any other upload. This is required BEFORE removing the Notion chip — the founder's own copy can be unshared or deleted at any time, and the chip removal must not leave the artifact recoverable only through Notion history. **Do not modify or delete the founder's original file** — export is read-only against it.
   - Skip this archival sub-step for hosted-viewer links (Papermark/DocSend/Brieflink) — the PDF conversion produced in Step 3B/3E is already the durable Drive copy; there's no separate "original file" to export.
   - Skip entirely for a Tom-authored Doc (e.g. Diligence Q&A) meant to stay editable — leave its native chip alone.
   - **Live Google Sheets are never superseded either — but they DO get a PDF snapshot alongside.** Per `~/.claude/skills/shared-references/spreadsheet-artifact-convention.md`, a spreadsheet carries both chips permanently: the PDF snapshot and the live source. Add the PDF; never remove the Sheet chip.
2. **Remove the superseded chip** from Diligence Materials:
   ```bash
   python3 ~/.claude/scripts/notion_files_property.py \
       --page-id <opportunity_page_id> --prop "Diligence Materials" \
       --url "<native_or_hosted_viewer_url>" --remove
   ```
   Exit 0 (including idempotent skip if already absent) = done; exit 1 = hard failure, log it and leave both chips in place rather than risk an inconsistent state.
3. **Never remove a chip whose PDF counterpart doesn't yet exist on the property.** This step only fires as the direct result of adding a PDF snapshot chip in the same run (or a run that's explicitly doing a materials-hygiene pass) — it is not a general "clean up old links" sweep.

Log the swap in the Step 5 summary (`[title] — native chip removed, archived as .docx/.pptx to Diligence/[Company]/, PDF chip is now canonical`).

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

After processing for a message completes, apply the outcome label per the rules in Step 2.5: `claude/materials-processed` only when ALL items landed (chips on the property, page body, Contact/Round Details updates if applicable — or explicit logged fallback notations), `claude/materials-failed` when any item fatally failed so the message stays retry-visible. **Remove `claude/materials-processing` at the same time** — the in-flight claim is resolved either way. This closes the idempotency loop in **all modes**: Mode B, Mode C manual, and delegated invocations from `pipeline-agent` / `add-to-crm`. Without this step, a follow-up run against the same Opp re-finds the same email in Gmail search and re-processes it, creating duplicate Drive uploads and duplicate chips.

**Repair stale chips before labelling (added 2026-08-04).** If any `notion_files_property.py add_link` call in this run returned `"skipped": true` **with `"stale_chip": true`**, do not treat it as a no-op. That result means a chip with the same canonical filename already exists but points at a *different* URL — which is what happens when the Drive Upload Apps Script **replaces** a same-named file: the old file is trashed and the replacement gets a new fileId, so the chip is now a dead link to a trashed object. Repair it with the two verified primitives, then continue:

```bash
python3 ~/.claude/scripts/notion_files_property.py remove-link --page-id <opp> --prop "Diligence Materials" --url "<existing_url from the result>"
python3 ~/.claude/scripts/notion_files_property.py add-link    --page-id <opp> --prop "Diligence Materials" --url "<new url>" --label "<label>"
```

A plain `"skipped": true` **without** `stale_chip` is the genuine no-op (identical URL already present) — leave it alone. Note this corrects an earlier claim in this file that canonical-filename dedup "catches the chip duplication": it prevents a *duplicate* chip, but in the replace case it does so by leaving the *stale* one in place, which is worse because it looks fine until Tom clicks it.

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
Materials hygiene: [title] — native/link-only chip removed, archived as .docx/.pptx to Diligence/[Company]/, PDF chip now canonical (or omit line if nothing superseded this run)
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
