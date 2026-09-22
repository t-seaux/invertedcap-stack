---
name: log-document-to-notes
description: Log a LETTER or REPORT (or memo / research doc / white paper) that Tom shares as a LINK or PDF to the Notion ✏️ Notes DB — capturing a Summary, an extracted Frameworks section, the full document text, and a source link. CONTEXT-DEPENDENT "log": routes HERE when the object is a document link or PDF of a letter/report/memo. Two carve-outs: an external INVESTMENT-FIRM letter (hedge fund / public-equity / VC / family-office letter) → log-investor-letter-to-notion instead; a video/interview/YouTube URL → log-transcript-to-notion. Trigger phrases: "log this report", "log this letter", "log this doc/memo", "log this" or "log to notes" when a document link/PDF is the object, or Tom forwards/uploads a report/letter with intent to archive it. Always trigger inline — no confirmation needed.
---

# Log Document to Notes (Notes Database)

Capture a letter/report/memo (link or PDF) as a ✏️ Notes DB entry: source link, summary, traceable Frameworks, and the full text. Sibling of `log-transcript-to-notion` (video) and `log-thread-to-notes` (chat) — same DB, same house style.

**Notes data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb` · **Opportunities:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`

## Dedup guard (do FIRST, before any fetch) — [[shared-references/notes-dedup-guard]]
Before spending any work: `notion-search` the **exact source URL/link**; if a live Notes-DB
page (`e8afa155-…`) already references it → **STOP**, don't fetch/parse/create, and reconfirm
the existing page: `✓ Already logged: **<title>** → <existing url>`. This is the cheap
re-send guard (it also skips the multi-minute extraction). Backstop lives in Step 5.

## Routing check (do FIRST)
- External **investment-firm letter** (SCGE/Oaktree/Baupost-style LP/fund/manager letter) → hand to **`log-investor-letter-to-notion`** (it files to the Non-Inverted Letters folder with investor framing). Not this.
- **Video/interview/YouTube URL** → `log-transcript-to-notion`. Not this.
- Otherwise (a company report, research/industry report, white paper, general business letter, memo) → continue here.

## Step 1 — Acquire the text
- **PDF/file provided:** extract text with `pdfplumber` (see the `pdf-reading` / `materials-handler` patterns). For a substantive body, prefer the real text; for scanned/image PDFs, note that OCR wasn't run.
- **Link provided:** fetch and strip to readable text (same curl + tag-strip pattern as `log-investor-letter-to-notion` Step 1). If paywalled/login-walled, tell Tom and ask him to send the PDF.
- **Open every linked/attached source before concluding** (per [[feedback_open_linked_material_before_concluding]]) — don't summarize from the title.

## Step 2 — Source link (contextual filing, NO junk bucket)
- **Link source:** the source link IS the original URL — no Drive upload needed.
- **PDF source:** upload for a stable link, filed CONTEXTUALLY (per Tom's no-dedicated-bucket rule, [[project_sms_stitch_screenshots]]):
  - Company/deal-tied → that company's `Diligence/<Company>/` (diligence root `1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d` → `createFolder` the subfolder), and set the Opp relation.
  - Not tied to a company → ask Tom where to file it (one short question); don't invent a bucket.
  Use the `drive-save` upload pattern (`shared-references/drive-upload.md`). Capture the returned Drive file URL.

## Step 3 — Title
`Report: <Source/Author> — <Title> (Mon DD, YYYY)` or `Letter: <Source/Author> — <Title> (Mon DD, YYYY)`. Pull the date from the document; omit only if genuinely undated. Use Tom's title verbatim if he gives one; don't ask to confirm.

## Step 4 — Body
```
📄 **Source:** [<Title>](<source url — Drive file or original link>)
✍️ **Author / Source:** <who wrote it>
🗓️ **Date:** <Mon DD, YYYY>
📝 **Summary:** <2–4 sentence synthesis of the substance>

---

**Frameworks**

<3–6 key theses/models the document argues — bold title + 2–4 sentences each, each anchored to a specific passage. MANDATORY traceability check: every framework must trace to actual text; drop any that can't. (Same rule as log-transcript-to-notion.)>

---

**Document**

<the full cleaned text, preserving structure — headings, bullets, paragraph breaks. Never a fenced code block. If very long and truncation is unavoidable: [Note: document truncated due to length].>
```

## Step 5 — Create, relate, classify
**Dedup backstop (before create, [[shared-references/notes-dedup-guard]] Check 2):**
`notion-query-data-sources` on `e8afa155-…` filtering `Name` `contains` the distinctive
title core; if a live (non-trashed) match exists → **STOP**, reconfirm the existing page,
do not create a second. This catches the restart/interleave race the URL check can miss.

`notion-create-pages` into the Notes data source (`Name`, `⭐️`="__NO__", body). **Parent
MUST be the Notes `data_source_id`, never workspace root** — readback the parent; if it came
back `workspace`, move it into the DB (the 9/18 ICONIQ orphan was a mis-parent). Best-effort `Opportunity` relation via `notion-search` on a confident single match. **Readback-verify** (`notion-fetch`) — MCP writes can silently no-op ([[reference_notion_tooling]]). Set the Claude icon (`shared-references/claude-note-icon.md`). Run `note-classifier` to set `Category`. Confirm: `✓ Logged to Notes: **<title>** → <url>`.
