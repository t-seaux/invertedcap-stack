---
name: investor-update
description: "Process investor update emails from PORTFOLIO COMPANIES (startups Tom has invested in) and attach them to the corresponding Notion Opportunities database entry. This is NOT the same as 'investor letters' — see disambiguation below. This skill operates in three modes. (A) Scheduled sweep — reconciliation pass that catches what the investor-update-inbound webhook missed; scans Tom's Gmail inbox for new investor update emails from the past 24 hours. (B) Webhook — invoked per inbound message via the claude-job-queue primitive, processes one specific email instead of scanning the inbox. (C) Manual trigger — auto-detects when the user forwards or pastes an investor update email in conversation (look for phrases like 'investor update', 'quarterly update', 'monthly update', 'portfolio update', company update newsletters, or any email that reads like a periodic business/financial update from a startup founder to their investors). In all modes, extract the company name, locate the matching Active Portfolio opportunity in Notion, create a new page in the Company Updates database with the email content, save a local PDF archive, and link the update to the Opportunity via a dual relation."
---

# Investor Update Processor (Portfolio Companies)

Process investor update emails **from portfolio companies** and store them in the dedicated Company Updates Notion database, linked to the correct Opportunity via a dual relation.

Read `references/schema.md` for database field details and matching logic before proceeding.

> **⚠️ Disambiguation — "Investor Updates" vs. "Investor Letters"**
>
> Tom uses two distinct terms that must not be confused:
>
> - **Investor updates** (this skill) — Periodic emails from **founders/CEOs of startups Tom has invested in**, sent to their investors. These come via Gmail, reference company-specific metrics (revenue, ARR, burn, runway), and get stored in the Company Updates Notion database linked to the matching Opportunity. Trigger phrases: "investor update", "portfolio update", "company update".
>
> - **Investor letters** (separate task: `daily-investor-letters`) — Published letters/memos from **other investment firms** (hedge funds, PE firms, public market managers) where Tom is an LP or follows their thinking. Examples: Berkshire Hathaway shareholder letters, Howard Marks memos, Pershing Square annual presentations. These are found via **web search** (not Gmail), and the digest is sent to Tom via Signal Note to Self. Trigger phrases: "investor letters", "scan for letters", "fund manager letters".
>
> If the user says "investor letters" or "letters from other firms," use the `daily-investor-letters` scheduled task — NOT this skill.

## Naming Convention

All investor update pages and PDF files use the format:

- **Monthly updates:** `[Company] - [Mon] [YYYY] Update` — e.g., `Soap Payments - Feb 2026 Update`, `Signal7 - Dec 2025 Update`
- **Annual updates:** `[Company] - [YYYY] Update` — e.g., `Slip.stream - 2025 Update`

Where `[Mon]` is the three-letter abbreviated month (Jan, Feb, Mar, Apr, May, Jun, Jul, Aug, Sep, Oct, Nov, Dec). Use the annual format when an update explicitly covers a full calendar year rather than a specific month.

This applies to both the Notion page title (`Name` property) and the local PDF filename.

## Workflow

1. Find investor update emails (scan inbox or parse forwarded email)
2. For each email: identify the portfolio company
3. Locate the matching Notion opportunity (Active Portfolio only)
4. Save a local PDF archive to `~/Library/CloudStorage/GoogleDrive-tom@invertedcap.com/My Drive/Portfolio/Investor Updates/` (mounted workspace folder on Tom's Mac)
5. Create a page in the Company Updates database with a plain-text file path to the local PDF and the email content formatted below
6. Unflag (unstar) processed emails in Gmail
7. Report results

## Investor Type Prioritization

Tom's preference: **prioritize updates from public market and private equity portfolio companies** over venture capital-backed companies. When processing multiple updates in a single scan, handle public/PE company updates first. If the scan finds a large batch and context is constrained, it is acceptable to defer VC-backed company updates and flag them as "deferred — lower priority" in the results summary, processing only the public/PE updates in that run. This applies to scan ordering and reporting only — VC-backed updates should still be processed when capacity allows, but they are never the priority.

## Step 1: Find Investor Update Emails

### Alternate Email Addresses

Tom receives investor updates at multiple email addresses — not just `tom@invertedcap.com`. Updates may also arrive at `thomas.seo@outlook.com` or `tom@dashfund.co`. When that happens, Tom forwards the email to `tom@invertedcap.com` so the skill has visibility. These forwarded emails should be treated identically to updates that arrive at the primary address.

During scanning, treat any email **forwarded by Tom to himself** as a legitimate investor update if the forwarded content matches the investor-update pattern (founder/CEO sender, company metrics, addressed to investors/stakeholders). To catch these:

- In scheduled scans, add `from:me "Fwd:" newer_than:1d` and `from:me "Forwarded" newer_than:1d` alongside the standard search queries, then inspect the forwarded body for investor-update signals.
- The **original sender** of the forwarded email (the founder/CEO) is the relevant sender for company identification (Step 2) and page metadata (Step 4). The **From** field in the Notion page body should reflect the original sender, not Tom's forwarding address.
- The **To** field in the Notion page body should read `Tom <tom@invertedcap.com>` regardless of which address originally received the update.

### Mode A: Scheduled Daily Sweep (primary mode)

**Step A1: Pull the Active Portfolio company list from Notion.**

Query the Opportunities database for all Active Portfolio companies. Use `notion-search` with `data_source_url: "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"` and query "Active Portfolio", or use the Agent View (`https://www.notion.so/5fa871c765d74251b8f96b63f248ef25?v=31400beff4aa80fdb2e0000c1b6ae673`) and filter for Active Portfolio status. Extract the company name and any known email domains or founder names from each entry. This is the source-of-truth list of companies to scan for.

**Domain alias table:** Some portfolio companies send updates from legacy or alternate domains that differ from their current brand. When searching by `from:` domain, always include alias domains in addition to the primary domain. Known aliases:

| Company | Primary Domain | Alias Domains |
|---|---|---|
| Argo (dba Runner) | `runner.now` | `voteagora.com` |

This table should be extended as new alias patterns are discovered. If a scan misses an update because the sender domain didn't match, add the alias here for future runs.

**Step A2: Search Gmail for each portfolio company.**

For each active portfolio company, run a targeted Gmail search using the company name (and domain if known):

**Company name resolution:** When an opportunity has a DBA or alias name (e.g., "Argo (dba Runner)"), run all subject/name-based queries below for **each** name variant — both the legal name and the DBA name. Extract DBA names from parenthetical suffixes like "(dba X)", "(fka X)", or "(now X)". For "Argo (dba Runner)", run queries for both "Argo" and "Runner".

- `q: "from:{company_domain} newer_than:1d"` — if the company's email domain is known. **Also run this query for each alias domain** listed in the domain alias table above.
- `q: "subject:{company_name} update newer_than:1d"` — search by company name in subject with "update"
- `q: "subject:{company_name} newer_than:1d"` — search by company name in subject (no keyword restriction). This catches non-standard subject lines like "Runner's log #2", "Signal7 Feb Recap", "Clerq Quarterly Digest", etc.
- `q: "{company_name} update newer_than:1d"` — broad fallback search by company name

Founders use a wide variety of subject line formats beyond "update" — including "log", "newsletter", "recap", "report", "digest", "news", "check-in", and numbered series like "#2" or "vol. 3". The subject-only query (no keyword restriction) is intentionally broad to catch these; false positives are filtered out in Step A4.

This portfolio-driven approach ensures updates are caught regardless of whether the subject line contains standard keywords like "investor update" or "monthly update."

**Step A3: Also scan for forwarded updates.**

In addition to the per-company searches, run these catch-all queries to find updates Tom forwarded to himself from alternate addresses:

- `q: from:me "Fwd:" newer_than:1d`
- `q: from:me "Forwarded" newer_than:1d`

Inspect forwarded email bodies for investor-update signals (founder/CEO sender, company metrics, addressed to investors/stakeholders). Match the forwarded content's original sender to the Active Portfolio list from Step A1.

**Step A4: Deduplicate and validate.**

Deduplicate all results by message ID. For each unique message, use `read_gmail_message` to fetch the full content. Discard any email that is clearly not an investor/portfolio update (e.g., marketing newsletters, product update notifications from SaaS tools, etc.). Genuine investor updates typically come from a founder or CEO, reference metrics like revenue/ARR/burn/runway, and are addressed to investors or stakeholders.

If no emails are found across all searches, report "No new investor updates found in the past 24 hours" and stop.

### Mode B: Webhook (single message)

Triggered by `gmail-webhook` (`investor-update.js`) on inbound mail that passes the cheap gate + portfolio founder lookup. The webhook enqueues a single job per matching message via the `claude-job-queue` primitive; the local processor invokes this skill with the args below.

**Args:**
- `messageId` (required) — Gmail message ID. Fetch the full message via Gmail MCP (`mcp__claude_ai_Gmail__search_threads` or read the message directly by ID) before proceeding to Step 2.
- `oppId` (optional) — Notion Opportunity page ID, pre-resolved by the webhook. Use as a fast-path in Step 2 (skip the search) but verify the page is still Active Portfolio before writing.
- `oppName` (optional) — Opportunity title, for log/Slack output before the Notion page is loaded.
- `forwardedSenderEmail` (optional) — set when the webhook detected `Fwd:` from Tom's own address. The original founder is in the forwarded body — use this email for company resolution, not the message's `From` header.

**Behavior in Mode B:**
- Process exactly the one message identified by `messageId`. Do **not** scan the inbox for other updates in this run.
- Run the same dedup check (Step 4 — `investorUpdatePageExists` by intended page title). If a page already exists, log and exit without re-uploading the PDF.
- Skip Step 1's portfolio-list query and per-company searches entirely — those exist for the scheduled mode.
- Otherwise proceed through Steps 2–6 unchanged. The Slack alert in Step 6 is the single notification for this update — the webhook does not post its own alert in this path.
- Single-message alert format (override Step 6's batch format): one line, same shape as a row in the batch — `📬 **<Company>** — "<subject or period>" — <PDF source>. <Notion page link>`. Skip the "Portfolio / Non-Portfolio / Needs review" section headers since there's only ever one entry.

### Mode C: Manual / Forwarded Email

If the user forwards or pastes an investor update email directly in conversation, parse it from the conversation context. Also check the uploads directory for any PDF attachments the user may have uploaded alongside the email.

## Step 2: Identify the Portfolio Company

For each email, extract the company name by checking (in order): subject line (often formatted as "[Company] Investor Update — [Month Year]"), sender display name, sender email domain, email body header/greeting.

Use `notion-search` to search for the company name among Active Portfolio opportunities. Alternatively, query the Portfolio view (see schema.md) and match against results.

- **Single strong match**: proceed directly
- **Multiple matches — follow-on vs. original**: Companies often have multiple Opportunity entries: an original investment (e.g., "Spade") and one or more follow-on entries (e.g., "Spade (Series A FO)", "Spade (Series B FO)"). **Always link investor updates to the original investment entry — never to a follow-on entry.** Follow-on entries are identifiable by a parenthetical suffix like "(Series A FO)", "(Series B FO)", "(FO)", etc. Discard these candidates entirely when selecting a match.
- **Multiple matches across funds (no follow-ons)**: When a company appears across multiple funds without follow-on suffixes (e.g., Uprise is both a Dash 1 and Dash 2 company), link to the **original (earliest) investment** — typically the lower-numbered fund.
- **Ambiguous / multiple matches**: present candidates to the user and ask for confirmation
- **No match**: flag the email to the user — the sender may not be in the portfolio database, or the company name may differ from what's in Notion

## Step 2.5: Normalize forwarded email body

Whenever the email is a forward (from any of Tom's alternate inboxes — `tom@invertedcap.com`, `tom@dashfund.co`, `thomas.seo@outlook.com` — with a `Fwd:` / `Forwarded` subject prefix, or detected via `forwardedSenderEmail` from the webhook), strip the forward chrome **before** the body is fed into either the Step 3 PDF render or the Step 4 Notion page body. The PDF and Notion page must read like the original founder update, not a forwarded email — Tom's forwarding hop is plumbing, not content.

Strip from the top of the body, in order, until the first non-chrome line:

1. The leading wrapper line — any of:
   - `Begin forwarded message:`
   - `---------- Forwarded message ---------`
   - `> Begin forwarded message:` (Apple Mail with quote prefix)
2. The metadata block that immediately follows the wrapper. These are consecutive lines starting with any of: `From:`, `Sent:`, `Date:`, `To:`, `Cc:`, `Bcc:`, `Reply-To:`, `Subject:` (plus Apple Mail's `<mailto:…>` annotations and any `>` quote-prefix variants of these). Strip the entire contiguous run.
3. Any blank lines and `\u200B` zero-width spaces that immediately follow the metadata block.
4. The object-replacement glyph `\uFFFC` (`￼`) wherever it appears (Apple Mail attachment placeholder).
5. Apple Mail quote-prefix `>` characters at the start of lines, when the entire forward is quote-prefixed (heuristic: 80%+ of non-blank lines start with `>`). Strip one leading `>` (and one optional space) per line.

Multi-hop forwards (founder→colleague→Tom→Tom) carry several forward wrappers stacked. Apply the strip recursively — after the first wrapper+metadata block is removed, if the next non-blank content is *another* wrapper line (matches step 1), strip that too. Stop when the next content is real prose.

Do **not** strip:
- The original sender's name/sign-off in the body (e.g., "— Charley")
- Any quoted text the founder included intentionally inside the update
- Inline images, links, or formatting in the actual update content

This normalization applies uniformly to Mode A (sweep), Mode B (webhook), and Mode C (manual). Where the rest of this skill says "the email body" / `body_markdown` / "the full email body", it refers to the **normalized** body produced here. The raw forwarded body is never written to the PDF or Notion page.

## Step 3: Save PDF to Google Drive

Save a PDF copy of every investor update to the company's subfolder within the **Investor Updates** folder on Google Drive. This serves as a durable backup independent of Notion.

**Google Drive folder structure:**
- Parent folder ID: `1-gsWvaUWOI5WYjKUA7OwQtoywcBJ3sP2` (Investor Updates)
- Each portfolio company has its own subfolder within this parent: `Investor Updates / [Company Name] / [filename].pdf`
- If a subfolder for the company does not exist, create one using the Drive Manager Apps Script's `createFolder` action with `parentFolderId: "1-gsWvaUWOI5WYjKUA7OwQtoywcBJ3sP2"` and the company name as `folderName`.
- The `createFolder` action returns the subfolder's `folderId` and `url`. Use this URL (not the parent folder URL) when constructing the Notion page body link.
- If the subfolder already exists, the Apps Script returns the existing folder without creating a duplicate.

Evaluate the email content to determine which case applies, checked in this order.

> **⚠️ ALWAYS probe for attachments first.** Run the Gmail Attachment Saver on the target message unconditionally *before* considering Case B or Case C — even when the email body shows a Google Doc link or appears to be pure prose. Tom frequently forwards investor updates with the PDF attached, and the `plaintextBody` returned by the Gmail MCP hides attachments behind the `￼` (object-replacement) placeholder glyph. The only reliable signal is probing the message with the Attachment Saver. Jumping straight to Case B when you see a Google Doc link duplicates work Tom already did and is a regression. If the saver returns a PDF, use Case A and stop. Only fall through to Case B / Case C if the saver returns zero files.

### Case A: PDF attachment exists

- In scheduled mode: Use the Gmail Attachment Saver Apps Script to save the attachment directly to the Investor Updates folder on Drive. Read the reference at `/Users/tomseo/.claude/skills/shared-references/gmail-attachment-saver.md` for the endpoint URL and API details.
  ```python
  import requests

  APPS_SCRIPT_URL = "https://script.google.com/macros/s/AKfycbwf_3QMmAYW8YB3WjfCW94p2pB3M-0sXCkqOqDg1BZ5_tD9eLauAJ2BEXpuusYORDBJ/exec"

  # Use the Investor Updates folder ID — look it up via google_drive_search
  resp = requests.post(APPS_SCRIPT_URL, json={
      "messageId": message_id,
      "driveFolderId": investor_updates_folder_id
  }, allow_redirects=True, timeout=60)

  result = resp.json()
  if result["success"]:
      for f in result["files"]:
          saved_filename = f["fileName"]
          drive_url = f["url"]
  ```
  After saving, if the file needs renaming to `[Company] - [Mon] [YYYY] Update.pdf`, note the original filename in the Notion page body. The Apps Script saves with the original attachment filename.
- In manual mode: locate uploaded PDFs in the uploads directory, then upload to the Investor Updates folder on Drive via the **Drive Upload Apps Script**. Read the reference at `/Users/tomseo/.claude/skills/shared-references/drive-upload.md` for the endpoint URL and Python usage. The endpoint accepts base64-encoded file content and saves directly to Drive — no Zapier or tmpfiles.org intermediary needed.

### Case B: Email contains a Google Doc link

Some founders share their investor update as a Google Doc instead of a PDF or inline email. Detect Google Doc URLs in the email body (patterns: `docs.google.com/document/d/{DOC_ID}`, Google Drive share links).

To convert the Google Doc to PDF:

1. Extract the document ID from the URL
2. Use `google_drive_fetch` with the document ID to get the doc content
3. Convert the fetched content to a PDF using weasyprint (same approach as Case C below, substituting the doc content for the email body)
4. If `google_drive_fetch` fails (permissions, etc.), fall back to Claude in Chrome: navigate to the Google Doc URL, use the browser's print-to-PDF functionality (Ctrl+P → Save as PDF), or take a full-page screenshot and convert to PDF

Name the file: `[Company] - [Mon] [YYYY] Update.pdf` (e.g., `Quiet AI - Jan 2026 Update.pdf`)
Upload to the Investor Updates folder on Drive via the **Drive Upload Apps Script** (see `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`).

### Case C: No attachment, no Google Doc — convert email body to PDF

**⚠️ CRITICAL: The PDF must contain the FULL VERBATIM email body. Never summarize, truncate, paraphrase, or omit any content. The PDF is an archive of the original email — every paragraph, bullet point, metric, ask, and sign-off must be preserved exactly as written. If the email body is long, the PDF should be long. A 2-3KB PDF from a multi-paragraph email is a red flag that content was dropped. For forwarded emails, use the normalized body from Step 2.5 — the forward chrome is gone but every word of the founder's original update is preserved.**

Use `weasyprint` + `markdown` to convert the email body to a styled PDF. First convert the email body from Markdown to HTML, then render to PDF. This preserves headers, bold, italics, lists, links, and tables — producing a PDF that closely matches what the email looks like.

The `body_markdown` parameter must be the **complete email body** — the same content that goes into the Notion page body. Read it from the Gmail message (via `gmail_read_message`) or from the forwarded email content in conversation.

```python
import markdown
import weasyprint

def email_to_pdf(subject, sender, date, body_markdown, output_path):
    """Convert a Markdown email body to a styled PDF via HTML."""
    html_body = markdown.markdown(body_markdown, extensions=['extra', 'sane_lists'])
    
    html = f"""<!DOCTYPE html>
<html><head><style>
body {{ font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif;
       font-size: 14px; line-height: 1.6; color: #1a1a1a; max-width: 700px; margin: 0 auto; padding: 40px; }}
h1 {{ font-size: 22px; font-weight: 700; margin-top: 32px; }}
h2 {{ font-size: 18px; font-weight: 600; border-bottom: 1px solid #e1e4e8; padding-bottom: 6px; margin-top: 28px; }}
.header {{ border-bottom: 2px solid #333; padding-bottom: 16px; margin-bottom: 24px; }}
.header h1 {{ margin: 0 0 8px 0; border: none; padding: 0; font-size: 24px; }}
.meta {{ color: #666; font-size: 13px; }}
ul, ol {{ padding-left: 24px; }}
li {{ margin-bottom: 6px; }}
em {{ font-style: italic; }}
strong {{ font-weight: 600; }}
a {{ color: #0066cc; }}
</style></head>
<body>
<div class="header">
    <h1>{{subject}}</h1>
    <div class="meta">From: {{sender}} | Date: {{date}}</div>
</div>
{{html_body}}
</body></html>"""
    
    weasyprint.HTML(string=html).write_pdf(output_path)
```

Name the file: `[Company] - [Mon] [YYYY] Update.pdf` (e.g., `Signal7 - Dec 2025 Update.pdf`)
Upload to the company's subfolder on Drive via the **Drive Upload Apps Script** (see `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`). Use `createFolder` to get or create the company subfolder first, then `upload` with the returned `folderId`.

## Step 4: Create Investor Update Page in Notion

Create a new page in the **Company Updates** database (`collection://bf491fb9-214f-456e-921b-5194b8187f2a`). This is the primary reading interface — Tom will click into each update page to read the content directly in Notion.

**⚠️ CRITICAL — FULL EMAIL BODY REQUIRED:** The page body must contain the **complete, verbatim email body** — every paragraph, bullet point, metric, quote, ask, and sign-off. Never summarize, truncate, paraphrase, or rewrite any part of the email. The Notion page is the primary archive of the update. If the original email is long, the Notion page should be long. Read the full email via `gmail_read_message` and use the `body` field directly. **For forwarded emails, use the normalized body produced by Step 2.5 — the forward chrome (wrapper line, metadata block, `>` quote prefixes, `\uFFFC` glyphs) is stripped, but the original founder content is preserved verbatim.**

### Page properties

| Property | Value |
|---|---|
| `Name` | `[Company] - [Mon] [YYYY] Update` (e.g., "Soap Payments - Feb 2026 Update") |
| `Company` | URL of the matched Opportunity page (e.g., `https://www.notion.so/{page_id_no_dashes}`) |
| `Update Date` | Date the email was sent (ISO-8601 date, not datetime) |
| `Source Email` | Gmail thread URL: `https://mail.google.com/mail/u/0/#inbox/{message_id}`. If the update was not delivered via email (e.g., manually provided in conversation, uploaded file, screenshot), set this field to `N/A`. |
| `Period` | The period(s) the update covers (not necessarily when sent). Format: `Mmm YYYY` for monthly; `YYYY` for annual. See extraction rules below. **This is a multi-select field** — pass as a JSON array of strings (e.g., `'["Feb 2026", "Mar 2026"]'`). For single-month updates, pass a one-element array (e.g., `'["Feb 2026"]'`). For multi-month or quarterly updates, include each month as a separate value. If an option does not yet exist in Notion, it will be created automatically. |
| `Traction` | Revenue or ARR figure for the corresponding period. See formatting rules below. |
| `Summary` | 1-2 sentence summary of the update body. Shorthand/truncated sentence structure. |

#### Period extraction

The `Period` field captures which period(s) the update is **about**, not when it was sent. This is a **multi-select** field. Determine the correct period(s) using these signals in priority order:

1. **Subject line** — often contains the period explicitly (e.g., "February 2026 Update", "Q1 2026", "2025 Annual Update")
2. **Body intro** — founders often state "Here's our update for January…", "Closing out Q4…", or "Reflecting on 2025…"
3. **Email send date as fallback** — if no explicit period is stated, assume the update covers the month immediately prior to when it was sent (e.g., email sent March 5 → `["Feb 2026"]`)
4. **Multi-month updates** — if an update covers multiple months (e.g., "February & March 2026"), add each month as a separate multi-select value: `["Feb 2026", "Mar 2026"]`. Do NOT combine them into a single value like "Feb & Mar 2026".
5. **Quarterly updates** — use the quarter label as a single value: `Q1 2026`, `Q4 2025`, etc. Do NOT expand into three separate month values. Quarter mapping: Q1 = Jan–Mar, Q2 = Apr–Jun, Q3 = Jul–Sep, Q4 = Oct–Dec.

**Format rules:**
- Monthly update → three-letter abbreviated month + year: `Jan 2026`, `Dec 2025`
- Quarterly update → quarter + year: `Q1 2026`, `Q4 2025`
- Annual update (covers a full calendar year) → year only: `2025`, `2024`
- Do not use `Dec 2025` for a full-year 2025 update — use `2025`
- Do not expand `Q1 2026` into `["Jan 2026", "Feb 2026", "Mar 2026"]` — use `["Q1 2026"]`
- Commas are not allowed inside individual select option names — always use separate values

#### Traction extraction

Scan the email body for the primary revenue or ARR metric tied to the period in `Period`. Apply this logic based on the metric type:

**Revenue-based companies** (companies that report revenue rather than ARR):
- `~$Xm (Revenue)` — e.g., `~$21m (Revenue)` for a company reporting ~$21m in annual revenue
- Use `~` prefix when the figure is approximate or rounded
- Use exact value when the update states a precise figure: `$21.3m (Revenue)`

**ARR-based companies** (companies that report ARR/CARR as their primary metric):
- **Single ARR figure, no CARR breakout** → `$X.Xm (ARR)` — e.g., `$4.2m (ARR)`
- **Contracted ARR and Live ARR broken out** → `$X.Xm (CARR); $X.Xm (Live ARR)` — e.g., `$5.1m (CARR); $4.2m (Live ARR)`
  - Never abbreviate as `(Live)` — always use `(Live ARR)` to distinguish from revenue metrics
- **Scoped or qualified ARR** — When the update explicitly qualifies the ARR figure with a modifier like "new product ARR", "platform ARR", "enterprise ARR", or any other scoping language that indicates the figure covers a subset of the business rather than total company ARR, preserve the qualifier in the Traction field. Format: `$X.Xk/m (New Product ARR)`, `$X.Xm (Enterprise ARR)`, etc. Do NOT strip the qualifier and present it as unqualified `(ARR)` — the distinction between total and scoped ARR is material. If the update reports both a scoped figure and a total figure, prefer the total but include the scoped figure after a semicolon: `$X.Xm (ARR); $X.Xk (New Product ARR)`.
- **ARR mentioned but not for the specific period** → `N/A` (do not extrapolate or carry forward from a different period)
- **Company is pre-revenue or explicitly not generating ARR** → `$0`
- **No traction information anywhere in the update** → `N/A`

**CRITICAL — CARR vs. ARR terminology:**
When an update lists both "CARR" and "ARR" as separate line items, they MUST be broken out as Contracted and Live ARR respectively:
- **CARR** (Contracted Annual Recurring Revenue) = the Contracted figure. This includes signed contracts that may not yet be generating live revenue.
- **ARR** (when listed alongside CARR) = the Live ARR figure. This is the revenue actually being collected.
- Example: email says "CARR: $560k" and "ARR: $180K" → field value = `$0.56m (CARR); $0.18m (Live ARR)`
- When only "ARR" is mentioned (no CARR), treat it as the single headline figure: `$X.Xm (ARR)`
- The distinction matters because CARR can be significantly higher than live ARR for early-stage companies with signed-but-not-yet-activated contracts. Always preserve this breakout when the data is available.

Normalize dollar amounts: values $1m and above use lowercase `m` with one decimal (e.g., "$4.2M ARR" → `$4.2m`; "$12.5 million" → `$12.5m`). Values under $1m use lowercase `k` (e.g., "$420K ARR" → `$420k`; "$728,000 ARR" → `$728k`). Zero values display as `$0` (no suffix). Always use lowercase `m` for millions and lowercase `k` for thousands.

#### Summary rules

Write a 1-2 sentence summary of the update's key substance. Focus on what matters most: revenue milestones, product launches, fundraising, hiring, strategic pivots, key metrics changes. Use shorthand aggressively — abbreviate where possible (e.g., "rev" for revenue, "HC" for headcount, "closed" for fundraising close), drop articles ("the", "a"), use truncated sentence fragments over full prose. Goal is maximum information density in minimal text. All dollar amounts use lowercase `m` for millions and lowercase `k` for thousands (e.g., `$3.5m`, `$350k`, `$916k`). Never use uppercase M or K.

Examples:
- `Crossed $5m ARR; launched enterprise tier w/ 3 design partners. Hiring 2 AEs.`
- `Closed $8m Series A led by Sequoia. Burn ~$350k/mo, 22mo runway. Product GA in Apr.`
- `Rev flat MoM at $280k MRR; lost 2 enterprise accounts. Pivoting GTM from inbound to outbound.`

### Page body content structure

The page body must follow this exact layout:

```
📁 [[Company Name] - Investor Updates](https://drive.google.com/drive/folders/[COMPANY_SUBFOLDER_ID])
[Full email body, preserving structure. Use ## for section headers,
bullet points for lists, **bold** for emphasis. Maintain the original
formatting as closely as possible.]
```

Key formatting rules:
- A clickable Google Drive folder link sits at the very top of the page body, prefixed with the 📁 (folder) emoji. The link text is `[Company Name] - Investor Updates` and the URL points to the company's subfolder within the Investor Updates folder. Example: `📁 [Quiet AI - Investor Updates](https://drive.google.com/drive/folders/1iXGMyXKs0mHEUZ2ez9_SI_4FxJZlUAT5)`. This renders as a clickable link in Notion that opens the company's Drive folder where all their update PDFs are stored.
- **No divider** between the folder link and the email body. The email content starts immediately on the next line.
- **Do NOT include a From/To/Date metadata block.** This information is already captured in the page properties (Source Email link, Update Date). Including it in the body is redundant.
- **Do NOT insert empty lines or blank paragraphs between content blocks.** Notion renders these as visible empty space.
- **Bullet points must be flat single-dash bullets** (`- item`), never double-dash (`- - item`). When the email has nested lists, flatten sub-items into the parent bullet or use indented sub-bullets with proper Notion markdown (tab + `-`).

### Creating the page

Use `notion-create-pages` with:

```
parent: { data_source_id: "bf491fb9-214f-456e-921b-5194b8187f2a" }
pages: [{
  properties: {
    "Name": "[Company] - [Mon] [YYYY] Update",
    "Company": "https://www.notion.so/[matched_page_id]",
    "date:Update Date:start": "[YYYY-MM-DD]",
    "date:Update Date:is_datetime": 0,
    "Source Email": "https://mail.google.com/mail/u/0/#inbox/[message_id] — or N/A if not from email",
    "Period": "[\"Mmm YYYY\", ...] — JSON array of month values, one per covered month. Single-month: [\"Feb 2026\"]. Multi-month: [\"Feb 2026\", \"Mar 2026\"]. Annual: [\"2025\"]",
    "Traction": "[extracted Traction or N/A or $0.0m — see Traction extraction rules]",
    "Summary": "[1-2 sentence shorthand summary — see Summary rules]"
  },
  content: "📁 [[Company Name] - Investor Updates](https://drive.google.com/drive/folders/[COMPANY_SUBFOLDER_ID])\n[formatted email body]"
}]
```

The content begins with a clickable folder link (`📁 [Company Name - Investor Updates]` as link text, company subfolder URL), followed immediately by the full email body. No divider, no From/To/Date metadata.

The dual relation automatically links the update back to the Opportunity — the `🗄️ Investor Updates` field on the Opportunity page will show this new entry.

### Deduplication

Before creating a new page, check if an update from the same company for the same month already exists:

1. Use `notion-search` with the intended page title (e.g., "Soap Payments - Feb 2026 Update") scoped to the Company Updates database
2. If an exact or near-exact match exists, skip creation and report "Update already logged" for that company

## Step 5: Unflag Processed Emails

After successfully creating the Notion page and saving the local PDF for an investor update, remove the star/flag from the original Gmail email so Tom's inbox reflects that the update has been processed.

For each email that was flagged (starred) in Gmail and has been fully processed (Notion page created + PDF saved):

1. Check whether the email is starred (it may not be — only starred emails need unflagging)
2. Remove the `STARRED` label from the message/thread to unflag it
3. If the Gmail modify tool is unavailable, use Claude in Chrome to navigate to the email and manually unstar it

Only unflag emails that were **successfully** processed end-to-end. If the Notion page creation failed or the PDF save failed, leave the flag intact so the email gets picked up on the next scan.

## Step 6: Report Results

**Notification channel:** All alerts (success, non-portfolio, misclassification review) MUST be delivered via the `send-alert` skill at `/Users/tomseo/.claude/skills/send-alert/SKILL.md` — pipe the GFM body through `~/.claude/skills/send-alert/send.sh`. This posts as the `claude` bot identity (the canonical channel for LLM-synthesized alerts; distinct from `tom` MCP and `alerts` Apps Script).

**Do NOT use `mcp__claude_ai_Slack__slack_send_message`** — that posts as `tom` (your user identity) and conflates with messages you actually sent. Mode B misclassification alerts and Mode A summaries both go via send-alert.

**Alert format (Slack):** Organize by **company**. Bold company names using GFM double asterisks (`**Quiet AI**`). Split into two sections: `**Portfolio**` (companies with Status in the Active Portfolio set, where the update was archived to Notion) and `**Non-Portfolio (filtered)**` (companies that didn't qualify — pipeline-only, unknown sender, or personal newsletter).

```
📬 PORTFOLIO UPDATES — YYYY-MM-DD

**Portfolio**
• **<Company>** — "<subject or period>" — <PDF source: original/email-converted>. <Notion page link>
• _none this run_ (if empty)

**Non-Portfolio (filtered)**
• **<Company>** — "<subject>" — <reason filtered> (e.g., Status: Scheduled → saved as Diligence Material; or Not in Opportunities DB — personal newsletter)
• (omit section if empty)

**Needs manual review**
• **<Sender/Company>** — <why ambiguous>
• (omit section if empty)
```

Rules:
- **Bold the company name** with double asterisks (GFM). The `send.sh` converter handles this correctly.
- Portfolio section = companies with Status in the Active Portfolio set (per the skill's Step 3 eligibility rule). Everything else goes under Non-Portfolio.
- For each Non-Portfolio entry, include a one-line reason so Tom can see why it didn't land in Notion as a portfolio update (e.g., "Status: Scheduled — saved as Diligence Material instead", "Not in Opportunities DB").
- The header uses the 📬 emoji, an em dash (—), and ISO date format. Example: `📬 PORTFOLIO UPDATES — 2026-03-06`.
- Internal sub-agent summary (returned to orchestrator) can be more verbose — include Gmail thread IDs, PDF paths, match attempts — but the Slack body stays to the format above.
