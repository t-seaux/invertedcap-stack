---
name: log-investor-letter-to-notion
description: Save an investor letter from another firm to the Notion Notes database. Trigger whenever Tom shares an investor letter — as a URL, pasted text, or uploaded file — from any external investment firm (hedge funds, public equity managers, VC firms, family offices, etc.). Trigger phrases include "log this letter", "add this letter to Notion", "save this investor letter", "add to notes", "log this to Notes", or when Tom pastes or forwards letter content with intent to archive it. Also trigger when Tom provides a URL to an investor letter or memo. Always trigger inline — no confirmation needed before acting.
---

# Log Investor Letter to Notion (Notes Database)

Accept an investor letter from an external firm — as a URL, pasted text, or uploaded file — and create a new page in the ✏️ Notes database with raw text, letter metadata, and an extracted Frameworks section.

**Notes database data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`

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

## Step 1.5: Upload Source File to Drive (when a file is provided)

**When the user provides a PDF or document file** (path in conversation, not a URL or pasted text), upload it to the **Non-Inverted Letters** Drive folder so the source link in Step 5 points at a real artifact.

- Target folder: `LP Letters / Non-Inverted Letters` — folder ID `1f4GM9uQRtH-hhQeLIPki92es_CvstcHn`
- Use the `drive-save` upload pattern (`shared-references/drive-upload.md`):

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"

filename = f"{firm} - {letter_subtitle}.pdf"  # e.g. "SCGE Quarterly Review - Q4 2025 Letter.pdf"

with open(local_path, "rb") as f:
    b64 = base64.b64encode(f.read()).decode()

resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": filename,
    "fileBase64": b64,
    "mimeType": "application/pdf",
    "folderId": "1f4GM9uQRtH-hhQeLIPki92es_CvstcHn",
}, allow_redirects=True, timeout=120).json()

drive_url = resp["url"]  # use as Source URL in Step 5
```

**Source URL precedence for Step 5:**
1. Uploaded Drive URL (Step 1.5 result) — preferred when a file was provided
2. Original public URL provided by the user (when no file, only a URL)
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

## Step 6: Create the Notion Page

Use `notion-create-pages` with:

```json
{
  "parent": { "data_source_id": "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" },
  "pages": [{
    "properties": {
      "Name": "<title from Step 3>",
      "Opportunity": "<opportunity page URL if found, else omit>",
      "⭐️": "__NO__"
    },
    "content": "<page body from Step 5>"
  }]
}
```

---

## Step 7: Set Claude Icon

Read the shared reference at `/Users/tomseo/.claude/skills/shared-references/claude-note-icon.md` and follow its instructions to set the custom Claude logo emoji as the page icon on the newly created note. This is required for all Claude-generated notes — do not skip.

---

## Step 8: Confirm to User

After successful creation, respond with one line:

> ✓ Letter logged to Notes: **[Note Title]** → [Notion page URL]

---

## Error Handling

- **URL behind paywall or login wall:** Inform the user the letter could not be fetched. Ask them to paste the letter text directly.
- **No author identified:** Use the firm name in the title and embed the letter type in the subtitle. Do not guess a name.
- **Date unclear:** Use the most specific date inferable from the letter body (e.g. publication date at top of letter, or quarter/year if no specific date). If genuinely unresolvable, use `Undated`.
- **Notion MCP unavailable:** Inform the user and suggest they log the letter manually.
- **Title unclear:** Default to `Letter: [Firm] — [file/URL identifier]` rather than asking.


---

## Final Step: Classify the Note

After the Notion page has been created and the Opportunity relation (if any) has been set, run the `note-classifier` skill to assign the correct `Category` field.

Read the skill at `/Users/tomseo/.claude/skills/note-classifier/SKILL.md` and follow its classification logic against the note just created. Do not skip this step — every note created by this skill must have a Category set.
