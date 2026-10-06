---
name: log-investor-letter-to-notion
description: Save an investor letter from another firm to the Notion Notes database. Trigger whenever Tom shares an investor letter — as a URL, pasted text, or uploaded file — from any external investment firm (hedge funds, public equity managers, VC firms, family offices, etc.). Trigger phrases include "log this letter", "add this letter to Notion", "save this investor letter", "add to notes", "log this to Notes", or when Tom pastes or forwards letter content with intent to archive it. Also trigger when Tom provides a URL to an investor letter or memo. SCOPE: this skill is for external INVESTMENT-FIRM letters (hedge fund / public-equity / VC / family-office). A general report / research doc / white paper / non-investor business letter → log-document-to-notes instead; a video/interview URL → log-transcript-to-notion. Always trigger inline — no confirmation needed before acting.
---

# Log Investor Letter to Notion (Notes Database)

Accept an investor letter from an external firm — as a URL, pasted text, or uploaded file — and create a new page in the ✏️ Notes database with raw text, letter metadata, and an extracted Frameworks section.

**Notes database data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`

**Dedup guard (before any fetch):** run `python3 ~/.claude/skills/shared-references/notes_dedup.py check --url <source url>` FIRST — exit 10 → STOP and reconfirm the existing page (skips the extraction). `notes_create.py` re-checks URL + title at write time.

---

## Step 1: Acquire the Letter Text

**If the letter text was pasted directly into the conversation**, use it as-is — do not attempt to fetch it.

**If a URL is provided**, fetch the page content:

```bash
# Try plain fetch first
curl -sL "<URL>" -o /home/claude/letter_raw.html
```

Then extract readable text from the HTML using Python:

```python
from html.parser import HTMLParser
import re

with open('/home/claude/letter_raw.html', 'r', errors='replace') as f:
    html = f.read()

# Strip script/style blocks
html = re.sub(r'<(script|style)[^>]*>.*?</\1>', '', html, flags=re.DOTALL)
# Strip all remaining tags
text = re.sub(r'<[^>]+>', ' ', html)
# Collapse whitespace
text = re.sub(r'[ \t]+', ' ', text)
text = re.sub(r'\n{3,}', '\n\n', text).strip()

with open('/home/claude/letter_text.txt', 'w') as f:
    f.write(text)
```

If the URL is behind a paywall, returns a login wall, or is otherwise inaccessible, inform the user and ask them to paste the letter text directly.

**If an uploaded PDF or document file is provided**, extract text using `pdfplumber` or `python-docx` as appropriate (see the `pdf-reading` and `file-reading` skills for extraction patterns). Save the extracted text to `/home/claude/letter_text.txt`.

**Hold onto the local file path** — Step 1.5 uploads it to Drive so the Notion source link can point at the canonical PDF instead of being omitted or (worse) self-referential.

---

## Step 1.5: Upload Source File to Drive (ALWAYS — file or link)

**Tom 2026-10-05: every logged letter gets a PDF Tom owns.** A URL source is first turned into a local PDF —
`python3 ~/.claude/skills/shared-references/source_pdf.py fetch "<url>" --out <scratch>/<name>.pdf` (exit 5 →
`docsend-to-pdf`; exit 3 → log with the original link only and tell Tom in one line). Then, file or link, upload it to the **Non-Inverted Letters** Drive folder so the source link in Step 5 points at a real artifact.

- Target folder: `LP Letters / Non-Inverted Letters` — folder ID `1f4GM9uQRtH-hhQeLIPki92es_CvstcHn`
- **As code (2026-10-04)** — upload with the shared CLI (spec: `shared-references/drive-upload.md`):

```bash
python3 ~/.claude/scripts/drive_upload.py upload "<local_path>" --folder non-inverted-letters \
    --name "<Firm> - <letter subtitle>.pdf"   # e.g. "SCGE Quarterly Review - Q4 2025 Letter.pdf"
# → {"ok":true,"fileId":"…","url":"https://drive.google.com/file/d/…/view"}  → `url` is the Source URL in Step 5
```

Exit **0** → use `url`. Exit **1/3** → retry once; still failing → fall through to Source precedence #2/#3
below and say so in the reply (never a self-referential Source). Exit **2** → the local path is wrong.

**Source URL precedence for Step 5:**
1. Uploaded Drive URL (Step 1.5 result) — always first; when the source was a link, append ` · [original](<url>)`
2. Original public URL — only when Step 1.5 could not produce a PDF (exit 3)
3. None — render Source as plain bold text, no hyperlink (see Step 5 guard)

**If only pasted text was provided** (no file, no URL), skip this step — there is nothing to upload.

---

## Step 2: Extract Metadata

From the letter text (and the URL/file if available), extract the following fields. Infer from the content — do not ask the user unless a field is truly unresolvable.

- **Firm**: Name of the investment firm that authored the letter (used in the title and Source label).
- **Author**: The named portfolio manager, CIO, or partner who signed the letter (used in the title only). If unsigned or firm-attributed, use the firm name here.
- **Letter type**: Classify as one of: `Annual Letter`, `Quarterly Letter`, `Q1/Q2/Q3/Q4 Letter`, `Market Commentary`, `Investment Memo`, or `Other` as appropriate (used in the title).
- **Publication date**: The specific date the letter was published or dated (e.g. "March 17, 2026"). Use the date at the top of the letter or the signatory date. This populates the `📅 **Date:**` metadata line.
- **Source URL**: The original URL provided by the user, if any. If the letter was pasted as text with no URL, omit.
- **Descriptive source label**: A short, readable label for the Source hyperlink (e.g. "Sixth Street Specialty Lending Stakeholder Letter", "Oaktree Capital Market Commentary Q1 2026"). Should identify the letter at a glance.
- **Topic summary**: 2–3 sentences summarizing the letter's central argument, market view, or primary thesis. Infer from content — do not ask.

---

## Step 3: Determine the Note Title

**Required title format:**
```
Letter: <Firm (full legal name or common name)> — <Descriptive Subtitle, Month Year>
```

Examples:
- `Letter: Sixth Street Specialty Lending (TSLX) — Stakeholder Letter, March 2026`
- `Letter: Third Point — Q4 2025 Letter to Investors`
- `Letter: Oaktree Capital — Howard Marks Market Commentary, Q1 2026`
- `Letter: Baupost Group — Annual Letter, Full Year 2025`

Include the author's name in the subtitle when they are well-known or the letter is personally attributed (e.g. Howard Marks memos). For firm-attributed letters without a prominent individual signatory, omit the author name from the title. Always include a time reference (month, quarter, or year) in the subtitle.

---

## Step 4: Identify Related Opportunity (optional, best-effort)

If the letter references a specific company that matches a deal in Tom's portfolio or pipeline, search for it:

```
Tool: notion-search
Query: <company name>
```

If a confident single match is found in the Opportunities database (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`), include its URL in the `Opportunity` property. If ambiguous or no clear match, leave it blank. Most investor letters will not have a matching Opportunity — skip this step if the letter is a broad market/portfolio commentary.

---

## Step 5: Build the Page Content

Structure the page body in this exact order:

```
📄 **Source:** [<Descriptive Source Title>](<source_url>)     ← omit link if no URL; use plain text if no URL
📅 **Date:** <Publication date, e.g. "March 17, 2026">
📝 **Summary:** <2-3 sentence topic summary>

---

**Frameworks**

<frameworks content — see below>

---

**Letter**

<raw letter text — see below>
```

Keep the metadata block tight — three lines maximum. Firm and author are captured in the title and Source link label; do not add separate `Firm:` or `Author:` lines. The Source label should be descriptive enough to identify the letter at a glance (e.g. "Sixth Street Specialty Lending Stakeholder Letter"). If no source URL is available, render the source as plain bold text with no hyperlink.

**HARD GUARD — no self-referential source link.** The Source URL must be EITHER (a) the Drive URL returned by Step 1.5, OR (b) the original public URL the user provided. It must NEVER be the Notion page's own URL (`notion.so/<this_page_id>`) — that yields a circular reference. If neither (a) nor (b) is available, render Source as plain bold text with no hyperlink. Before writing the page, sanity-check that the resolved source URL does not contain the substring `notion.so` or `notion.site` — if it does, strip the link.

---

### Frameworks Section

Use `**Frameworks**` as a bolded text label (not a Markdown header `##`). Identify 3–6 key mental models, investment theses, or analytical frameworks the author articulates in the letter. For each framework:

- State it as a bolded short title (e.g. `**Value Compression in Late-Cycle Markets**`)
- Follow with 2–4 sentences explaining the framework in the author's own logic, using the letter's specific arguments — not generic abstractions
- Ground it with a concrete example, position, or argument from the letter to anchor it to actual content

The Frameworks section should read as a dense intellectual extract — something Tom can scan to quickly internalize the manager's core thinking without reading the full letter.

---

### Letter Section

Use `**Letter**` as a bolded text label (not a Markdown header `##`). Render the full raw letter text below it, preserving paragraph breaks and section structure as faithfully as possible.

- Do not summarize, paraphrase, or editorialize within the Letter section — this is the raw source text
- Preserve section headers, salutations, and signatures as they appear in the original
- If the letter was fetched from HTML, minor formatting artifacts (e.g. extra whitespace, stripped italics) are acceptable
- If the letter text is extremely long and must be truncated to fit Notion's block limits, note at the bottom: `[Note: letter truncated due to length — full text available at source URL]`

---

## Step 6: Create, classify, confirm

**One command does the write — never `notion-create-pages` / `notion-update-page` for this page** (Costanoa 10/3: a
hand-rolled MCP create passed `parent_id`, the page landed private at workspace root, and the full text, Category and
icon were all skipped while "✅ Logged" went out anyway). Write the title and body from the steps above to a markdown
file in your scratch dir, then:

```
python3 ~/.claude/skills/shared-references/notes_create.py --kind investor-letter --title "<title>" --body <file.md> \
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

---

## Error Handling

- **URL behind paywall or login wall:** Inform the user the letter could not be fetched. Ask them to paste the letter text directly.
- **No author identified:** Use the firm name in the title and embed the letter type in the subtitle. Do not guess a name.
- **Date unclear:** Use the most specific date inferable from the letter body (e.g. publication date at top of letter, or quarter/year if no specific date). If genuinely unresolvable, use `Undated`.
- **notes_create.py exit 2 (Notion unreachable):** tell Tom it did not log; never ✓.
- **Title unclear:** Default to `Letter: [Firm] — [file/URL identifier]` rather than asking.
