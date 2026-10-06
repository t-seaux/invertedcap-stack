---
name: log-document-to-notes
description: Log a LETTER or REPORT (or memo / research doc / white paper) that Tom shares as a LINK or PDF to the Notion ✏️ Notes DB — capturing a Summary, an extracted Frameworks section, the full document text, and a source link. CONTEXT-DEPENDENT "log": routes HERE when the object is a document link or PDF of a letter/report/memo. Two carve-outs: an external INVESTMENT-FIRM letter (hedge fund / public-equity / VC / family-office letter) → log-investor-letter-to-notion instead; a video/interview/YouTube URL → log-transcript-to-notion. Trigger phrases: "log this report", "log this letter", "log this doc/memo", "log this" or "log to notes" when a document link/PDF is the object, or Tom forwards/uploads a report/letter with intent to archive it. Always trigger inline — no confirmation needed.
---

# Log Document to Notes (Notes Database)

Capture a letter/report/memo (link or PDF) as a ✏️ Notes DB entry: source link, summary, traceable Frameworks, and the full text. Sibling of `log-transcript-to-notion` (video) and `log-thread-to-notes` (chat) — same DB, same house style.

**Notes data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb` · **Opportunities:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`

## Dedup guard (do FIRST, before any fetch) — [[shared-references/notes-dedup-guard]]
**Dedup guard (before any fetch):** run `python3 ~/.claude/skills/shared-references/notes_dedup.py check --url <source url>` FIRST — exit 10 → STOP and reconfirm the existing page (skips the extraction). `notes_create.py` re-checks URL + title at write time.

## Routing check (do FIRST)
- External **investment-firm letter** (SCGE/Oaktree/Baupost-style LP/fund/manager letter) → hand to **`log-investor-letter-to-notion`** (it files to the Non-Inverted Letters folder with investor framing). Not this.
- **Video/interview/YouTube URL** → `log-transcript-to-notion`. Not this.
- Otherwise (a company report, research/industry report, white paper, general business letter, memo) → continue here.

## Step 1 — Acquire the text
- **PDF/file provided:** extract text with `pdfplumber` (see the `pdf-reading` / `materials-handler` patterns). For a substantive body, prefer the real text; for scanned/image PDFs, note that OCR wasn't run.
- **Link provided:** fetch and strip to readable text (same curl + tag-strip pattern as `log-investor-letter-to-notion` Step 1). If paywalled/login-walled, tell Tom and ask him to send the PDF.
- **Open every linked/attached source before concluding** (per [[feedback_open_linked_material_before_concluding]]) — don't summarize from the title.

## Step 2 — Owned PDF, always (Tom 2026-10-05: "always save to a PDF that I own — don't want to keep clicking someone else's link")
Every logged document gets a PDF in Tom's Drive — **link sources too**, not just uploaded files. Never ask first.
1. **Link source → local PDF** (code, harness `shared-references/tests/test_source_pdf.py`):
   ```
   python3 ~/.claude/skills/shared-references/source_pdf.py fetch "<url>" --out <scratch>/<name>.pdf
   ```
   Google Slides / Docs / Sheets → export (public, else Drive API as Tom); raw PDF → download; web page → Chromium print.
   Exit **5** → DocSend / Papermark: run `docsend-to-pdf`, use its PDF · **3** → link-walled: log anyway with the original
   link and tell Tom one line ("couldn't pull a PDF — send the file?") · **2** → wrong skill (video) or folder.
   PDF provided → use it as-is.
2. **Upload** (`drive_upload.py upload … --name "<Author/Firm> - <Title> (<date>).pdf"`), filed by context:
   - Company/deal-tied → `--folder diligence --company "<Company>"`, and set the Opp relation.
   - Outside report / presentation / research (firm decks, bank research, market updates — e.g. Costanoa, Caplight,
     Morgan Stanley) → `--folder non-inverted-letters`. That folder IS the home for outside writing; don't ask.
3. **Source line = Tom's PDF first, original second:**
   `📄 **Source:** [<Title> (PDF)](<drive url>) · [original](<url>)` (original only when the source was a link).

## Step 3 — Title
`Report: <Source/Author> — <Title> (Mon DD, YYYY)` or `Letter: <Source/Author> — <Title> (Mon DD, YYYY)`. Pull the date from the document; omit only if genuinely undated. Use Tom's title verbatim if he gives one; don't ask to confirm.

## Step 4 — Body
```
📄 **Source:** [<Title> (PDF)](<Tom's Drive PDF url>) · [original](<original url, link sources only>)
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

## Step 5 — Create, relate, classify, confirm

**One command does the write — never `notion-create-pages` / `notion-update-page` for this page** (Costanoa 10/3: a
hand-rolled MCP create passed `parent_id`, the page landed private at workspace root, and the full text, Category and
icon were all skipped while "✅ Logged" went out anyway). Write the title and body from the steps above to a markdown
file in your scratch dir, then:

```
python3 ~/.claude/skills/shared-references/notes_create.py --kind document --title "<title>" --body <file.md> \
    [--url <source url>]... [--opp <Opportunities page id>] [--title-verbatim]
```

The script does dedup, the title and body shape checks, the create (parent hardcoded to the Notes DB), ⭐️ off,
Opportunity, Category (note-classifier code when `--opp` is set), the Claude icon, and the readback. Pass `--opp` only on a
confident single Opportunities match (`notion-search`); `--title-verbatim` when Tom dictated the title.

- **exit 0** → send its `confirm` line VERBATIM. That is the only path to a ✓.
- **exit 10** → already logged: `✓ Already logged: **<existing.title>** → <existing.url>`. Create nothing.
- **exit 11** → a page with the SAME TITLE exists but was logged from a different link (`logged_urls`): open it and compare with this document. Same document → treat as exit 10. Different document → create it (make the title distinguishable if needed) – never report "already logged" for a different document.
- **exit 3** → fix the CONTENT that `failures` names (missing section, title shape, fenced block) and re-run.
- **exit 4 / 2** → it did NOT log (exit 4 means the page was archived again). Tell Tom it failed and why. Never ✓.
