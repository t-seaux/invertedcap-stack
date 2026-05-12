---
name: investor-update
description: "Process investor update emails AND board materials (board meetings/decks/updates) from PORTFOLIO COMPANIES (startups Tom has invested in) and attach them to the corresponding Notion Opportunities database entry. Board materials count as formal comms outreach and route through this skill — they live in the same `Update Type = Formal` track as regular investor updates, but use a distinct title naming convention (`Board Meeting` instead of `Update`). There is NO separate `Board` Update Type; board materials are Formal entries with a different title pattern. This is NOT the same as 'investor letters' — see disambiguation below. This skill operates in three modes. (A) Scheduled sweep — reconciliation pass that catches what the investor-update-inbound webhook missed; scans Tom's Gmail inbox for new investor update or board emails from the past 24 hours. (B) Webhook — invoked per inbound message via the claude-job-queue primitive, processes one specific email instead of scanning the inbox. (C) Manual trigger — auto-detects when the user forwards or pastes an investor update email or board deck/notification in conversation (look for phrases like 'investor update', 'quarterly update', 'monthly update', 'portfolio update', 'board meeting', 'board deck', 'board update', 'board materials', company update newsletters, Google Slides share notifications for portfolio decks, or any email that reads like a periodic business/financial update or board communication from a startup founder to their investors/directors). In all modes, extract the company name, locate the matching Active Portfolio opportunity in Notion, create a new page in the Company Updates database with the email content, save a PDF archive, and link the update to the Opportunity via a dual relation."
---

# Investor Update Processor (Portfolio Companies)

Process investor update emails **and board materials** from portfolio companies and store them in the dedicated Company Updates Notion database, linked to the correct Opportunity via a dual relation.

> **Scope — investor updates AND board materials.** This skill handles two flavors of formal portfolio comms, both stored as `Update Type = Formal`:
> - **Investor updates** — periodic founder-authored letters with metrics. Title format: `[Company] - [Mon] [YYYY] Update`.
> - **Board materials** — board meetings, board decks, board updates, board materials shared with directors (commonly via Google Slides share notifications). Title format: `[Company] - [Mon] [YYYY] Board Meeting`.
>
> Both are `Formal` entries — there is no separate `Board` Update Type. The only divergence is the title naming convention. Summary and Traction are extracted the same way for both: parse the underlying artifact (email body for investor updates; deck content for board materials) and populate canonical aggregate metrics. Never default to `N/A`/"detail in deck" when deck content is accessible. The track design lives in `~/.claude/skills/shared-references/company-updates-db.md`.

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

All entries are `Update Type = Formal`. Title format depends on what kind of artifact this is:

**Investor updates:**
- **Monthly:** `[Company] - [Mon] [YYYY] Update` — e.g., `Soap Payments - Feb 2026 Update`, `Signal7 - Dec 2025 Update`
- **Annual:** `[Company] - [YYYY] Update` — e.g., `Slip.stream - 2025 Update`
- Use annual format when an update explicitly covers a full calendar year.

**Board materials:**
- **Board meetings/decks/updates:** `[Company] - [Mon] [YYYY] Board Meeting` — e.g., `Outmarket - May 2026 Board Meeting`
- The month is the meeting month (taken from the email subject, board calendar invite, or document title — e.g., "Board Meeting May 5th 2026" → `May 2026`). Day-of-month is intentionally dropped from the title; meeting day is captured in the `Update Date` property if needed.

Where `[Mon]` is the three-letter abbreviated month (Jan, Feb, Mar, Apr, May, Jun, Jul, Aug, Sep, Oct, Nov, Dec).

This applies to both the Notion page title (`Name` property) and the PDF filename.

## Workflow

1. Find investor update emails (scan inbox or parse forwarded email)
2. For each email: identify the portfolio company
3. Locate the matching Notion opportunity (Active Portfolio only)
4. Save a local PDF archive to `~/Library/CloudStorage/GoogleDrive-tom@invertedcap.com/My Drive/Portfolio/Investor Updates/` (mounted workspace folder on Tom's Mac)
5. Create a page in the Company Updates database with a plain-text file path to the local PDF and the email content formatted below
6. Report results

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

**Domain alias table:** Some portfolio companies send updates from legacy or alternate domains that differ from their current brand (rebrands, corporate-vs-marketing splits, country-TLD variants). When searching by `from:` domain, always include alias domains in addition to the primary domain.

**Canonical source:** `~/code/gmail-webhook/Code.js → DOMAIN_ALIAS_GROUPS`. Read that array when you need the current list — do not maintain a duplicate table here, it will drift. The webhook gate's `expandDomainAliases()` already consults this list, so any pair registered there is automatically respected by `investor-update.js`, `calendar-scheduled-detect.js`, and the portfolio Contact audit.

**When to extend:** add a new pair to `DOMAIN_ALIAS_GROUPS` whenever a portfolio Opp's `Contact` email domain doesn't match the `Website` domain (founder uses `@x.io` while website is `@x.com`, rebrand from `oldbrand.com` to `newbrand.com`, etc.). Without the alias entry: (a) the calendar attendee domain filter will drop the founder's email as suspected pollution, (b) inbound from that domain may miss the investor-update gate. Confirm both sides are owned by the same company / founder before adding — the table is a trust anchor, not a fuzzy match.

**Step A2: Search Gmail for each portfolio company.**

For each active portfolio company, run a targeted Gmail search using the company name (and domain if known):

**Company name resolution:** When an opportunity has a DBA or alias name (e.g., "Argo (dba Runner)"), run all subject/name-based queries below for **each** name variant — both the legal name and the DBA name. Extract DBA names from parenthetical suffixes like "(dba X)", "(fka X)", or "(now X)". For "Argo (dba Runner)", run queries for both "Argo" and "Runner".

- `q: "from:{company_domain} newer_than:1d"` — if the company's email domain is known. **Also run this query for each alias domain** listed in the domain alias table above.
- `q: "subject:{company_name} update newer_than:1d"` — search by company name in subject with "update"
- `q: "subject:{company_name} newer_than:1d"` — search by company name in subject (no keyword restriction). This catches non-standard subject lines like "Runner's log #2", "Signal7 Feb Recap", "Clerq Quarterly Digest", etc.
- `q: "{company_name} update newer_than:1d"` — broad fallback search by company name
- `q: "{company_name} board newer_than:1d"` — board materials catch-all (board meeting / board deck / board update). Hits subject and body. Also catches Google Slides share notifications (subject typically `Presentation shared with you: "{Company} Board Meeting {Date}"`).

Founders use a wide variety of subject line formats beyond "update" — including "log", "newsletter", "recap", "report", "digest", "news", "check-in", and numbered series like "#2" or "vol. 3". Board materials surface as "Board Meeting", "Board Deck", "Board Update", "Board Materials", or Google Slides share notifications (`Presentation shared with you: ...`). The subject-only and board queries are intentionally broad; false positives are filtered out in Step A4.

This portfolio-driven approach ensures updates are caught regardless of whether the subject line contains standard keywords like "investor update" or "monthly update."

**Step A3: Also scan for forwarded updates.**

In addition to the per-company searches, run these catch-all queries to find updates Tom forwarded to himself from alternate addresses:

- `q: from:me "Fwd:" newer_than:1d`
- `q: from:me "Forwarded" newer_than:1d`

Inspect forwarded email bodies for investor-update signals (founder/CEO sender, company metrics, addressed to investors/stakeholders). Match the forwarded content's original sender to the Active Portfolio list from Step A1.

**Step A4: Deduplicate, classify, and validate.**

Deduplicate all results by message ID. For each unique message, use `read_gmail_message` to fetch the full content. Discard any email that is clearly not portfolio comms (e.g., marketing newsletters, product update notifications from SaaS tools, etc.).

Two valid kinds pass validation. Both are `Update Type = Formal`; the kind only affects the title naming convention.

- **Investor update** — comes from a founder or CEO, references metrics like revenue/ARR/burn/runway, addressed to investors or stakeholders. Title: `[Company] - [Mon] [YYYY] Update`.
- **Board material** — references a board meeting, shares a board deck, or is a Google Slides share notification for a board presentation. Sender is typically a founder, CEO, or COO; recipients are typically board directors. Subject often contains "Board Meeting", "Board Deck", "Board Update", "Board Materials", or `Presentation shared with you:` (Slides share template). Body length can be short (Slides share notifications carry minimal email body — the substance is in the deck, which MUST be read for Summary/Traction extraction). Title: `[Company] - [Mon] [YYYY] Board Meeting`.

If neither kind fits, discard. If no emails pass validation across all searches, report "No new investor updates or board materials found in the past 24 hours" and stop.

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
- Otherwise proceed through Steps 2–5 unchanged. The Slack alert in Step 5 is the single notification for this update — the webhook does not post its own alert in this path.
- Single-message alert format (override Step 5's batch format): one line, same shape as a row in the batch — `📬 **<Company>** — "<subject or period>" — <PDF source>. <Notion page link>`. Skip the "Portfolio / Non-Portfolio / Needs review" section headers since there's only ever one entry.

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

### HARD GATE — Portfolio-only

**REFUSE to create a Company Updates entry if the matched Opportunity's `Status` is not in the portfolio set:**

- ✅ Allowed Statuses: `Committed`, `Active Portfolio`, `Portfolio: Follow-On`, `Exited`
- ❌ Forbidden Statuses (refuse to write): `Pass (Met)`, `Pass (DNM)`, `Pass Note Pending`, `NR / Missed`, `Lost`, `Qualified`, `Outreach`, `Connected`, `Scheduled`, `Exploration`, `Active`, `Track`, `Assigned`, `N/A`

The Company Updates DB exists to track investments-Tom-actually-made. Pipeline activity, passed deals, and unmatched senders all belong elsewhere. If the matched Opp is non-portfolio, log the email under "Non-Portfolio (filtered)" in Step 5's Slack alert and exit without writing — DON'T create the entry.

This rule applies in all three modes (sweep / webhook / manual). In Mode B (webhook), the cheap gate already filters by founder lookup, but always re-verify the Status before writing — Statuses can move (e.g., a company moves from Active Portfolio → Pass over time, or vice versa). The Status check at write-time is the source of truth.

**Why the hard gate matters:** Soap Payments was Status=Pass (Met) but had two updates land in Company Updates DB before the gate was tightened (manually archived 2026-05-01). The DB should never have non-portfolio rows.

### Content-vs-Opp validation (referral guard)

Even when the sender resolves to an Active Portfolio Opp via Contact match, the email content may be **about a different company** — a referral, intro, or cold pitch the sender forwarded or wrote up for Tom. Contact membership alone is not proof that the email is THAT Opp's investor update.

Two shapes to reject:

1. **Forwarded cold pitch.** Subject is `Fwd:`/`FW:`, body carries an inner `From: ... <email>` whose domain doesn't match the matched Opp's website domain or the outer sender's domain, and the body contains pitch markers (`raising`, `pitch`, `deck`, `pre-seed`, `seed`, `series A/B/C`, `intro`, `warm intro`). The webhook gate (`investor-update.js → isForwardedReferral`) catches most of these before the skill runs — this rule is the belt-and-suspenders for sweep (Mode A) and manual (Mode C).

2. **Fresh note about a different company.** Sender is in the matched Opp's Contact (e.g. coinvestor on the cap table, advisor, or even a portfolio founder) and writes a fresh email tipping Tom on a different company — no `Fwd:` prefix, but the body's primary subject is some OTHER company by name, with pitch/intro framing. Identify by: the body names a company that is NOT the matched Opp, plus pitch markers, plus the email reads as third-person commentary about that company rather than first-person reporting on the matched Opp's metrics.

When either shape is detected, log under "Non-Portfolio (filtered)" in Step 5's Slack alert with reason "referral — content does not match Opp" and exit without writing.

Common drivers Tom flagged: (a) portfolio founders sometimes refer Tom to investment opportunities; (b) coinvestors who attended a joint meeting may sit in the Opp's Contact and later forward or write up unrelated deals.

The upstream root-cause fix lives in `calendar-scheduled-detect.js → mergeAttendeesIntoContact`, which now domain-filters meeting attendees so non-founder emails don't pollute Contact in the first place. This skill-level guard catches messages that slip through (e.g. addresses Tom added manually, or pre-existing pollution).

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

> **⚠️ ALWAYS probe for attachments first.** Run the Gmail Attachment Saver on the target message unconditionally *before* considering Case B / C / D / E — even when the email body shows a Google Doc/Slides link or appears to be pure prose. Tom frequently forwards investor updates with the PDF attached (and texts himself board deck PDFs alongside the Slides share notification). The `plaintextBody` returned by the Gmail MCP hides attachments behind the `￼` (object-replacement) placeholder glyph, so the only reliable signal is probing the message with the Attachment Saver. Jumping straight to Case B/D when you see a Google Doc/Slides link duplicates work Tom already did and is a regression. If the saver returns a PDF, use Case A and stop. Only fall through to Case B / Case C / Case D / Case E if the saver returns zero files.

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

### Case B: Email contains a Google Doc or Sheet link

Some founders share their investor update as a Google Doc (or, less commonly, a Google Sheet — e.g., a financial plan). Detect Google Doc / Sheet URLs in the email body (patterns: `docs.google.com/document/d/{DOC_ID}`, `docs.google.com/spreadsheets/d/{SHEET_ID}`, Google Drive share links).

**Routing depends on whether the email is direct vs. forwarded from `tom@dashfund.co`:**

- **Direct email** (recipient is `tom@invertedcap.com` and the message is not a `Fwd:` from `tom@dashfund.co` / `thomas.seo@outlook.com`): the founder shared the Doc/Sheet with Tom's invertedcap address, so the Drive MCP (OAuth'd to invertedcap) should have read access. Proceed with the MCP fetch path below.
- **Forwarded from `tom@dashfund.co`** (the Doc/Sheet was permissioned to dashfund only and Tom forwarded the share notification to invertedcap): the MCP will 403. Use the dashfund-slides-proxy `copyFile` action — it runs under the dashfund identity and copies the file into a Drive folder readable by invertedcap, preserving Google-native types as editable Docs/Sheets.

Conversion path:

1. **Direct-email path — Drive MCP fetch.** Use `google_drive_fetch` (or `mcp__claude_ai_Google_Drive__read_file_content`) with the document ID to get the file content. Generate a PDF via weasyprint (same approach as Case C, substituting the Doc/Sheet content for the email body).
2. **Forwarded-from-dashfund path — call dashfund-slides-proxy `copyFile`.** Do NOT attempt the MCP fetch first — call the dashfund-slides-proxy with `action: copyFile`, passing `sourceFileId = <doc/sheet id>` and `targetFolderId = <company-subfolder-id>` (resolved via the `createFolder` step in the Drive Upload Apps Script). The proxy returns the copy's `fileId` in the company subfolder. Then read the copy via `mcp__claude_ai_Google_Drive__read_file_content` on that `fileId` and generate a PDF via weasyprint (same as Case C).
3. **Direct-email MCP failure fallback.** If the direct-email path unexpectedly 403s, retry via dashfund-slides-proxy `copyFile` — the file may have been shared with dashfund instead.
4. **Final fallback — alert.** If both paths fail (proxy returns `Source file not accessible by dashfund identity`), surface a one-line Slack alert via `send-alert` asking Tom to download the PDF and forward it back. Do NOT create the Notion page yet.

Name the resulting PDF: `[Company] - [Mon] [YYYY] Update.pdf` (e.g., `Quiet AI - Jan 2026 Update.pdf`).
Upload the PDF to the company's subfolder on Drive via the **Drive Upload Apps Script** (see `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`). When the original was a native Google Doc/Sheet, link BOTH the original (or the copied) file URL and the generated PDF URL in the Artifacts property at Step 4.5 — the Signal7 Financial Plan precedent.

### Case D: Email contains a Google Slides link (typically board decks)

Board materials are usually shared as Google Slides — either as a share notification (`Presentation shared with you: "..."`) or via a `docs.google.com/presentation/d/{SLIDES_ID}` link in the body. The email body itself carries little substance; the deck is the artifact.

**Routing depends on whether the email is direct vs. forwarded from `tom@dashfund.co`:**

- **Direct email** (recipient is `tom@invertedcap.com` and the message is not a `Fwd:` from `tom@dashfund.co` / `thomas.seo@outlook.com`): the founder shared the deck with Tom's invertedcap address, so the Drive MCP (OAuth'd to invertedcap) should have read access. Proceed with the MCP fetch path below.
- **Forwarded from `tom@dashfund.co`** (the deck was permissioned to dashfund only and Tom forwarded the share notification to invertedcap): the MCP will 403. Use the dashfund-slides-proxy below — it runs under the dashfund identity and bridges the deck into a Drive folder readable by invertedcap.

Conversion path:

1. **Texted-PDF / forwarded PDF check (always first).** Look in iMessage attachments and Gmail attachments on the message for a PDF Tom may have already provided. If a PDF is found, route through Case A and stop.
2. **Direct-email path — Drive MCP fetch.** If the email is a direct invertedcap share (per routing rule above), call `google_drive_fetch` with the Slides document ID and request PDF export. Save the resulting PDF, name it `[Company] - [Mon] [YYYY] Board Meeting.pdf`, and upload to Drive via the Drive Upload Apps Script.
3. **Forwarded-from-dashfund path — call dashfund-slides-proxy.** If the email is a `Fwd:` from `tom@dashfund.co` (or `thomas.seo@outlook.com`), do NOT attempt the MCP fetch — call the dashfund-slides-proxy Apps Script (see `~/.claude/skills/shared-references/dashfund-slides-proxy.md`). The proxy runs under the dashfund identity, exports the Slides to PDF, and writes it into the company's subfolder under `Investor Updates` (which is shared with both accounts). Pass `slidesId`, `fileName = [Company] - [Mon] [YYYY] Board Meeting.pdf`, and `targetFolderId = <company-subfolder-id>`. On success, the response carries the PDF's Drive URL — proceed to Step 4 normally.
4. **Direct-email MCP failure fallback.** If the direct-email path unexpectedly 403s (e.g., the founder shared with the wrong account), retry via dashfund-slides-proxy — the deck may have been shared with dashfund instead.
5. **Final fallback — alert.** If both the MCP fetch AND the dashfund-slides-proxy fail (proxy returns `Slides not accessible by dashfund identity`, meaning the deck is shared with neither account), surface a one-line Slack alert via `send-alert` asking Tom to download the PDF and text/forward it back. Do NOT create the Notion page yet. The webhook's idempotency key tracks the message so the next sweep won't re-alert.

Alert format (Step 5 fallback only — sent via `send-alert`):
```
🎯 BOARD DECK NEEDS PDF — {Company}
Slides link: {slides_url}
Tried: invertedcap MCP (403), dashfund-slides-proxy ({error}). Deck not shared with either account.
Action: download as PDF and text/forward — I'll resume the workflow when the PDF lands.
```

Name the resulting PDF: `[Company] - [Mon] [YYYY] Board Meeting.pdf` (e.g., `Outmarket - May 2026 Board Meeting.pdf`).
Upload to the company's subfolder on Drive via the **Drive Upload Apps Script** (see `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`) — except in the Step 3 dashfund-proxy path, which handles its own write.

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

### Case E: Interactive content — Loom links, video files, audio demos

Founders sometimes ship updates as interactive content rather than written prose: Loom walkthroughs, screen-recorded product demos, video board updates, audio recordings. Frequent on iMessage (Mode C) but possible on Gmail too. Detect by URL pattern (`loom.com/share/{ID}`, YouTube share links, attached `.mp4`/`.mov`/`.m4a` files, etc.) or by message body that gestures at "watch this", "demo video", "I recorded a quick walkthrough", etc.

**There is no PDF to save in this case.** The artifact is the video itself. Skip the PDF / Drive Upload step. The substance lives in the spoken content, and the page body MUST carry a structured summary of what's in the video — bare Loom links in the body are a regression. Tom cannot search, skim, or recall video content months later from a link alone; the summary IS the archive for retrieval.

**Transcript fetch — Loom (preferred path):**
1. Use `yt-dlp` (already installed at `/opt/homebrew/bin/yt-dlp`) to pull the auto-generated subtitle track:
   ```bash
   cd /tmp && mkdir -p loom-transcripts && cd loom-transcripts
   yt-dlp --write-auto-subs --write-subs --sub-langs "en.*,en" --skip-download \
     --convert-subs vtt -o "video.%(ext)s" "https://www.loom.com/share/{ID}"
   ```
2. Read the resulting `video.en.vtt`. WebFetch on Loom share URLs is unreliable (bot-detection page) — go straight to yt-dlp.

**Transcript fetch — other sources:**
- YouTube share links: same yt-dlp invocation works.
- Direct video file (`.mp4` / `.mov`): if a transcript isn't available, note that in the page body summary and ask Tom whether to skip or to spin up local transcription. Don't silently emit a stub summary.

**Page body shape (Step 4 override for Case E):**
- Open with the iMessage / email metadata block (From / Channel / Date) and the founder's accompanying message verbatim.
- For each video, render a `## Demo N — <topic>` section containing: the Loom/video link, an approximate runtime in parens, and 1–3 short paragraphs (or a bulleted breakdown) summarizing what's covered, derived from the transcript. Capture the substantive product / business / financial content — not just the structure of the demo.
- If multiple videos share a unifying frame (e.g., a pitch positioning vs. competitors that the founder restates across both), lift that into a `## Pitch frame` section above the per-video sections so it isn't buried in only one demo.
- Preserve any distribution restrictions or context the founder gave (e.g., "don't share with anyone else") — these are material for downstream handling.

**Properties (Step 4 override for Case E):**
- `Source Email` — `N/A` if origin is iMessage; otherwise the Gmail thread URL.
- `Traction` — usually `N/A` for product-demo videos (no aggregate metric disclosed). For founder-recorded video board updates that DO disclose ARR/revenue, extract per the standard rules.
- `Period` — the month the videos were sent in, unless they explicitly cover prior-month data (apply normal Period extraction rules).
- `Artifacts` — after page creation, call the headless helper (Step 4.5) once per demo or material URL (Loom, YouTube, product demo link, slide link, any external resource that is the substantive artifact). Use the URL as `--url` and a short descriptive label as `--label` (e.g., `Demo 1 — AP inbox walkthrough`). These links also remain in the page body — both locations are populated.



For Cases A, B, and D — where substance can live in a separate artifact (PDF attachment, Google Doc/Sheet, Google Slides) rather than only the email body — read the saved artifact before drafting the Notion page in Step 4. This is what enables the Summary and Traction fields to reflect actual update content rather than defaulting to "see attached" / `N/A`. Founders routinely put the headline number in the artifact, not the email body — especially board decks (Case D) and Doc-based updates (Case B).

- **Case A (PDF attachment):** read the saved PDF via `mcp__claude_ai_Google_Drive__download_file_content` using the `fileId` returned by the Gmail Attachment Saver. (Or fetch the attachment inline via the Gmail MCP and parse the PDF locally.)
- **Case B (Google Doc/Sheet):** read the file via `mcp__claude_ai_Google_Drive__read_file_content` using the `fileId` returned by `copyFile` (forwarded-from-dashfund path) or by `google_drive_fetch` (direct-email path).
- **Case D (Google Slides):** read the exported PDF via `mcp__claude_ai_Google_Drive__download_file_content` using the `fileId` returned by the dashfund-proxy or by the direct-email upload step. For image-heavy decks where text extraction comes back thin, use `mcp__claude_ai_Google_Drive__get_file_metadata` for the content snippet as a fallback.
- **Case E (Loom / video / audio):** read the `video.en.vtt` transcript pulled via yt-dlp at Case E's transcript fetch step. Strip the WEBVTT timecode lines and concatenate the spoken text. The transcript IS the artifact — there is no PDF to read.

Extract the canonical aggregate metric (Traction) and the headline narrative (Summary) per the rules in Step 4, AND draft the per-section page-body summary required by Case E. **Never default to "see deck" / "see attached" / "watch the Loom" / `N/A` when the artifact was successfully read** — the whole point of reading the artifact is to make its content searchable in Notion.

Case C doesn't require this step — the email body IS the artifact, and Step 4's body-based extraction is sufficient.

## Step 4: Create Investor Update Page in Notion

Create a new page in the **Company Updates** database (`collection://bf491fb9-214f-456e-921b-5194b8187f2a`). This is the primary reading interface — Tom will click into each update page to read the content directly in Notion.

**⚠️ CRITICAL — FULL EMAIL BODY REQUIRED:** The page body must contain the **complete, verbatim email body** — every paragraph, bullet point, metric, quote, ask, and sign-off. Never summarize, truncate, paraphrase, or rewrite any part of the email. The Notion page is the primary archive of the update. If the original email is long, the Notion page should be long. Read the full email via `gmail_read_message` and use the `body` field directly. **For forwarded emails, use the normalized body produced by Step 2.5 — the forward chrome (wrapper line, metadata block, `>` quote prefixes, `\uFFFC` glyphs) is stripped, but the original founder content is preserved verbatim.**

### Page properties

| Property | Value |
|---|---|
| `Name` | Investor update: `[Company] - [Mon] [YYYY] Update` (e.g., "Soap Payments - Feb 2026 Update"). Board material: `[Company] - [Mon] [YYYY] Board Meeting` (e.g., "Outmarket - May 2026 Board Meeting"). |
| `Update Type` | Always `Formal` — both investor updates and board materials use the Formal track. There is no separate `Board` Update Type. See `~/.claude/skills/shared-references/company-updates-db.md`. |
| `Company` | URL of the matched Opportunity page (e.g., `https://www.notion.so/{page_id_no_dashes}`) |
| `Update Date` | Date the email was sent (ISO-8601 date, not datetime). For board materials with an explicit meeting date in the subject (e.g., "Board Meeting May 5th 2026"), prefer the meeting date over the email send date. |
| `Source Email` | Gmail thread URL: `https://mail.google.com/mail/u/0/#inbox/{message_id}`. If the update was not delivered via email (e.g., manually provided in conversation, uploaded file, screenshot), set this field to `N/A`. |
| `Period` | The period(s) the update covers (not necessarily when sent). Format: `Mmm YYYY` for monthly; `YYYY` for annual. See extraction rules below. **This is a multi-select field** — pass as a JSON array of strings (e.g., `'["Feb 2026", "Mar 2026"]'`). For single-month updates, pass a one-element array (e.g., `'["Feb 2026"]'`). For multi-month or quarterly updates, include each month as a separate value. **For board materials**: use the meeting month — `["May 2026"]` for a May 2026 board meeting. If an option does not yet exist in Notion, it will be created automatically. |
| `Traction` | Revenue or ARR figure for the corresponding period. See formatting rules below. **Read every attachment to extract Traction.** Whenever an attachment is present (deck, financial sheet, memo, etc.), open it and pull the canonical aggregate metric from the artifact — do NOT default to `N/A` just because the email body doesn't restate the figure. Founders routinely put the headline number in the attachment, not the body. Use `mcp__claude_ai_Google_Drive__get_file_metadata` (content snippet) or `download_file_content` for Drive files; fetch Gmail attachments inline. Use `N/A` only if the artifact genuinely discloses no aggregate company-level metric (rare). |
| `Summary` | 1-2 sentence shorthand summary. **Read every attachment.** Whenever the email or message has an attached artifact (deck PDF, financial sheet, memo, doc, slide export — board or otherwise), open it and incorporate its substance into the Summary. Never write "detail in deck", "see attached", "see PDF", or any other placeholder phrase that defers to the artifact instead of summarizing it. The email body alone is rarely the full picture — founders put the substantive numbers and narrative in the attached document. Use `mcp__claude_ai_Google_Drive__get_file_metadata` (content snippet) or `download_file_content` for Drive files; fetch attachments inline for Gmail attachments. Apply the same shorthand rules below. Example: `Closed largest deal ever — IMA Financial ($975k ARR / $1.95m TCV); ARR $5.0m (Apr) up from $2.1m Dec — 2.4x in 4mo, 34x YoY; 16/100 Top 100 brokers customers + 18 in pipeline; NDR 105% / GDR 95%; 100+ mo runway on $18.7m cash.` |
| `Artifacts` | Files & Media property — populated AFTER page creation via the headless helper (see Step 4.5 below). The structured Artifacts field is the canonical "what files belong to this entry" list; the body's Drive folder link stays for navigation context. |

#### Period extraction

**Two-field model:**
- `Update Date` = when the email was sent (the send date).
- `Period` = the **data month(s) the body actually reports on**. NOT the founder's labeling of the update.

These can diverge. Founders often send "April 2026 Update" emails on Apr 13 that report **March data** (revenue, GPV, customer counts measured at end of March). The send-month branding is just the founder's convention; the substantive content is the prior month. Period reflects what data is in the body.

Determine the correct period(s) by reading the body — look for the time period stamp on the actual numbers:
- Body says "February 2026: Revenue +6.5% MoM, GPV $22.9M..." → Period=`["Feb 2026"]` even if subject says "March Update"
- Body says "March: Revenue +30% MoM..." in an April-sent letter → Period=`["Mar 2026"]`
- Body explicitly covers two months (e.g., "Feb & Mar combined update") → Period=`["Feb 2026", "Mar 2026"]`
- Quarterly cadence reporting Jan–Mar data → Period=`["Q1 2026"]` (single quarter option)
- Annual recap covering full calendar year → Period=`["2025"]`

**Title vs. Period can diverge.** The entry's title typically matches the founder's send-month branding (e.g., "Clerq - Apr 2026 Update"), but Period may be `[Mar 2026]` if the body reports March data. That's intentional — title preserves the founder's labeling for searchability; Period preserves the data identity for filtering and grouping.

**Forecast / plan documents** (one-off forward-looking docs, not periodic updates) use a special `YYYY (Plan)` Period option. Example: Signal7 sent a "2026 Financial Plan" in Dec 2025 with EOY targets for 2026 → Period=`[2026 (Plan)]`, NOT `[Dec 2025]` (it's not a Dec update) and NOT `[2026]` (it's a forecast, not a 2026 retrospective). The `(Plan)` suffix marks the Period as forward-looking. Title in this case is typically `[Company] - YYYY Financial Plan` (or similar), matching the document's nature.

**Format rules:**
- Monthly update → three-letter abbreviated month + year: `Jan 2026`, `Dec 2025`
- Quarterly update → quarter + year: `Q1 2026`, `Q4 2025`
- Annual update (covers a full calendar year) → year only: `2025`, `2024`
- Do not use `Dec 2025` for a full-year 2025 update — use `2025`
- Do not expand `Q1 2026` into `["Jan 2026", "Feb 2026", "Mar 2026"]` — use `["Q1 2026"]`
- Commas are not allowed inside individual select option names — always use separate values

#### Traction extraction

Scan the email body for the primary revenue or ARR metric tied to the period in `Period`. Apply this logic based on the metric type:

**FORMAT: metric OUTSIDE parens, period stamp INSIDE.** Formal entries use month / quarter / year precision (no day) — the period is the meaningful anchor, and stamping it visibly lets Tom see at a glance "this is an April figure" even when scanning the entry in May. Multi-month covers go as a range with en dash. Examples:

- `$5.0m ARR (Apr)` — monthly update covering April
- `$5.0m ARR (Nov–Dec)` — multi-month update covering Nov + Dec (en dash range)
- `$5.0m ARR (Q1 2026)` — quarterly update
- `~$21m Revenue (2025)` — annual update
- `$0 (Apr)` — explicit pre-rev confirmation in an April update

**Revenue-based companies** (companies that report revenue rather than ARR):
- `~$Xm Revenue (Mon)` — e.g., `~$21m Revenue (2025)` for a company reporting ~$21m in 2025 annual revenue
- Use `~` prefix when the figure is approximate or rounded
- Use exact value when the update states a precise figure: `$21.3m Revenue (Q4 2025)`

**ARR-based companies** (companies that report ARR/CARR as their primary metric):
- **Single ARR figure, no CARR breakout** → `$X.Xm ARR (Mon)` — e.g., `$4.2m ARR (Apr)`
- **Contracted ARR and Live ARR broken out** → `$X.Xm CARR; $X.Xm Live ARR (Mon)` — e.g., `$5.1m CARR; $4.2m Live ARR (Apr)`
  - Never abbreviate as `Live` — always use `Live ARR` to distinguish from revenue metrics
  - Single date paren applies to both metrics
- **Scoped or qualified ARR** — When the update explicitly qualifies the ARR figure with a modifier like "new product ARR", "platform ARR", "enterprise ARR", or any other scoping language that indicates the figure covers a subset of the business rather than total company ARR, preserve the qualifier in the Traction field. Format: `$X.Xk New Product ARR (Mon)`, `$X.Xm Enterprise ARR (Mon)`, etc. Do NOT strip the qualifier and present it as unqualified `ARR` — the distinction between total and scoped ARR is material. If the update reports both a scoped figure and a total figure, prefer the total but include the scoped figure after a semicolon: `$X.Xm ARR; $X.Xk New Product ARR (Mon)`.
- **ARR mentioned but not for the specific period** → `N/A` (do not extrapolate or carry forward from a different period)
- **`$0`** — use ONLY when the founder explicitly states the company has zero current revenue/ARR (e.g., "we're pre-revenue", "$0 ARR", "haven't started charging yet"). The signal is an explicit zero, not absence of mention.
- **`N/A`** — use when the update simply doesn't discuss current aggregate revenue/ARR/MRR. The topic isn't covered, so we have no figure to report.

**AGGREGATE COMPANY-LEVEL VALUES ONLY.** Traction is always at the company level — total ARR, total MRR, total revenue. Individual customer ACVs (e.g., "Cardless contract at $300k ACV", "A1 closed at $250k ACV") are NOT traction data and should NEVER appear here. Per-customer figures belong in Summary or the page body. If an update only discloses per-customer values without an aggregate, Traction is `N/A`. Never derive aggregate values from per-customer disclosures (e.g., "Cardless is 40% at $300k → $750k total" is forbidden extrapolation).

**`$0` ≠ `N/A`.** They are distinct values with different meanings. `$0` says "the company has no revenue, confirmed by the founder." `N/A` says "the update didn't disclose an aggregate traction figure." Tom reviews entries with these distinct intents — never substitute one for the other.

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
    "Update Type": "Formal",
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

## Step 4.5: Populate Artifacts field with the specific PDF Drive URL

After the page is created, attach the saved PDF (Step 3) to the entry's `Artifacts` Files & Media property using the headless helper. This makes `Artifacts` the canonical structured artifact list — the body's Drive folder link stays for navigation context, but the per-entry file is in the property field.

```bash
python3 ~/.claude/local-agents/notion-internal/add_link_to_files_property.py \
  --page-id <new_entry_id> \
  --prop "Artifacts" \
  --url "<drive_pdf_view_url>" \
  --label "<filename, e.g. Quiet AI - Dec 2025 Update.pdf>"
```

Per pinned memory `feedback_notion_files_property_headless_default.md`, the headless helper is the canonical path for Files-property writes — Notion's public API can't write Files properties for arbitrary external URLs. The helper is idempotent (skips if URL already present) and uses `token_v2` from `~/.claude/local-agents/notion-internal/token_v2`. On `TOKEN_EXPIRED` (exit code 2), surface the error in Step 5's Slack alert so Tom can refresh the cookie.

If the email had multiple PDF attachments (rare but possible — e.g., letter + accompanying deck), call the helper once per file. The Artifacts property becomes the full list of files belonging to this update entry.

If the original source was a Google Doc / Sheet (Case B in Step 3) AND a PDF was generated from it, link BOTH the original Google Doc/Sheet URL and the generated PDF URL — preserving the original artifact alongside the archive copy mirrors the Signal7 Financial Plan precedent (PDF + original Sheet both in Artifacts).

**General rule (all cases):** Any URL in the email/message body that represents a material or demo — Loom, YouTube, product demo, slide deck, external doc, or similar — should also be added to Artifacts via the headless helper, in addition to living in the page body. Use a short descriptive label derived from the surrounding context (e.g., section header or founder's description). The Artifacts field is the canonical index of everything substantive attached to an update entry.

## Step 5: Report Results

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
- **Freshness rule (per `feedback_alert_freshness_framework.md`):** Sweep alert content is restricted to the run's lookback window. The window size is `RUN_LOOKBACK_HOURS` exported by `run.sh` — 24h on Tue–Fri, 72h on Mondays (catches Sat/Sun/Mon since the sweep skips weekends). Do NOT add an "Already-present" or "Webhook activity also created" line for entries the webhook handled before the window. If sweep had no work because every fresh update was already webhook-processed within the window, list those entries in the Portfolio section with the same one-line shape (the entry exists and is fresh — sweep just didn't write it). If nothing landed in the window at all (webhook or sweep), report "No new investor updates or board materials found in the past {N} hours" (interpolating the actual window) and stop — no reconciliation breakdown.
