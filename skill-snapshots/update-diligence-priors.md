---
name: update-diligence-priors
description: >
  Update priors on a first-pass diligence analysis. Identifies new information (call notes,
  transcripts, materials, backchannel feedback) added since the last analysis, assesses impact
  on existing priors, and prepends a dated update section to the Notion diligence page. Trigger
  on "update first pass", "update priors on [company]", "update diligence on [company]",
  "refresh diligence", "new info on [company] — update the analysis", "incorporate the new call
  notes for [company]", "revise priors on [company]", "I just had another call with [company],
  update the analysis", or any variant requesting a prior update on an existing diligence
  analysis. Always trigger inline — no confirmation needed.
---

# Update Diligence Priors

Incorporate newly available information into an existing first-pass diligence analysis and prepend
a dated update section to the Notion page. This skill is designed to be run multiple times over the
life of an opportunity — each run adds a new update section at the top of the page, creating a
chronological record of how the thesis has evolved as new evidence emerges.

The update should be opinionated. The point is not to summarize new information neutrally — it is
to assess whether the new evidence strengthens or weakens specific priors from the original
analysis, and to say so clearly.

---

## Step 1: Locate the Existing Diligence Page

Search the Notes database (`collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) for the existing
first-pass diligence page. Use `notion-search` with the query "Claude: [Company Name] First-Pass
Diligence".

If no existing diligence page is found, tell Tom and suggest running the `first-pass-diligence`
skill first. Do not proceed.

Fetch the full page content with `notion-fetch` — you need the original analysis text to understand
what priors were established.

---

## Step 2: Identify New Information

Fetch the Notion Opportunity page for the company from the Opportunities DB
(`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Compare the current state of the opportunity against
what was available when the diligence page was last written or updated.

### What counts as new information

Fetch every linked item in the opportunity's relation fields and compare against what the
existing diligence page references in its Sources section:

- **✍️ Notes relation** — fetch each linked note. Any note not cited in the existing diligence
  page's Sources section is new. These are typically call notes, backchannel references, research
  threads, or Claude analysis pages.
- **Diligence Materials** — check for new Google Drive files, PDFs, or attachments not referenced
  in the existing Sources. Fetch and read any new materials using the type-specific access methods
  documented in the `first-pass-diligence` skill (Step 1c): Google Drive PDFs via `read_pdf_bytes`,
  Google Docs via `google_drive_fetch`, Notion-hosted attachments (`attachment:{UUID}:{filename}`)
  via the Chrome-based pipeline (getSignedFileUrls API → navigate to signed URL → pdf.js text
  extraction), and DocSend links (`docsend.com/view/...`) via the `docsend-to-pdf` skill. If
  Chrome is unavailable for Notion attachments, note them as unreadable and flag the gap.
- **Page body changes** — the opportunity page body itself may have been updated with new context
  (round details, team changes, status updates).
- **Property changes** — check for changes to Status, Stage, Round Details, or other properties
  that signal deal progression.

For each new item found, fetch its full content. Call notes and transcripts are especially
important — read them carefully for founder signal, new data points, and answers to previously
open questions.

If the current conversation contains information Tom has shared directly (e.g., "I just learned
that..." or forwarded content), treat that as new information too.

### If no new information is found

If nothing has changed since the last analysis or update, tell Tom directly: "No new materials,
notes, or changes found for [Company] since [date of last analysis/update]. Nothing to update."
Do not create an empty update section.

---

## Step 3: Assess Impact on Priors

This is the analytical core of the skill. For each piece of new information, evaluate its
impact on the priors established in the original analysis (or the most recent update).

Work through these questions for each new data point:

1. **Which specific prior does this affect?** Map new information to the Need to Believe items,
   framework assessments, open questions, or section-level conclusions from the existing analysis.
2. **Does it reinforce or challenge the prior?** Be direct. "This reinforces..." or "This
   challenges..." or "This is orthogonal to the existing analysis but introduces a new
   consideration."
3. **How material is the update?** Not all new information changes the thesis. A second call that
   confirms what the first call established is reinforcing but not thesis-altering. A finding that
   directly contradicts a Need to Believe item is material.
4. **Does this resolve any open questions?** Check the Open Questions from each section of the
   original analysis. If new information answers one, note that explicitly.
5. **Does this introduce new questions?** New information often raises new questions that weren't
   on the radar before.

---

## Step 4: Write the Update Section

Compose the update section. The writing should be analytical and direct — same voice and standards
as the original first-pass diligence. Each update is a self-contained section that a reader can
understand without re-reading the full original analysis, though it references specific sections
and priors from the original.

### Format

```markdown
---

## Update — [Month Day, Year]

### New Information Processed

Group items into categories using bold sub-headers. Standard categories (use only those
that apply — omit any category with zero items):

**Research & Analysis**
- [Pre-mortems, Claude analysis threads, market research, diligence question frameworks]

**References**
- [Backchannel reference calls, formal reference checks]

**Customer Feedback**
- [Customer/prospect feedback notes, NPS data, churn interviews]

**Call Notes**
- [Founder calls, team calls, investor calls]

**Materials**
- [Updated decks, memos, data rooms, financial models]

Order: Research & Analysis → References → Customer Feedback → Call Notes → Materials.

Within each bullet, **bold the label text before the em dash**. For example:
- **Pre-Mortem: Tuor** — March 18, 2026. Structured failure mode analysis...
- **John Cotter (Harness, ex-Arthur AI)** — March 16, 2026. Enterprise sales leader...

### Prior Assessment

[Analytical prose organized by the priors that are affected. Use subsections if multiple
distinct priors are impacted. For each affected prior:]

**[Prior / Section Reference] — [Reinforced | Challenged | Revised | New Consideration]**

[2-3 paragraphs of analytical prose explaining what the new evidence says, how it maps to
the original assessment, and what it means for the investment thesis. Ground every claim
in specific evidence from the new materials.]

[If information-dense, include a supporting table — e.g., comparing original assumptions
to updated data points, before/after metrics, or a revised competitive landscape.]

### Revised Open Questions
- [Any open questions that have been answered, marked as resolved]
- [Any new open questions that emerged from the new information]

### Net Assessment

[One paragraph synthesis: on balance, does the new information make you more or less
convicted in the thesis? Be specific about what moved and what didn't.]

---
```

### Guidance

- **Include tables when the update involves data.** If the new call provided specific metrics
  (loss rates, conversion rates, pipeline numbers), create a table comparing the original
  assumptions or open questions to the new data points. This makes the delta visible at a glance.
- **Reference the original analysis by section.** Say "The original Market section (1.1) noted..."
  or "Need to Believe item 2 (cash flow underwriting) is directly addressed..." so the reader
  can cross-reference.
- **Be explicit about confidence levels.** "This substantially de-risks NTB item 3" is more
  useful than "This is encouraging."
- **Don't repeat the original analysis.** The update section should be additive — new information
  and its implications only. Don't restate what the original analysis already covered.

---

## Step 5: Prepend to the Notion Page

Use `notion-update-page` with the `update_content` command to insert the new update section at
the top of the page, immediately after the page title and before the first existing section
(which will be either a previous update section or the "Framework Mapping" header).

Find the first `---` or `## Framework Mapping` or `## Update —` in the existing content and
insert the new update section before it.

The update sections should stack chronologically — newest at the top, oldest at the bottom,
with the original analysis below all updates. This means a page that has been updated three times
will read: Update 3 → Update 2 → Update 1 → Original Analysis.

When prepending the first update to a page that has never been updated before, also insert a
section divider and header before the original first-pass content:

```markdown
---
# Original First-Pass Memo — [Original Publication Date]
---
```

This goes immediately before `## Framework Mapping — Inverted Lens` (or whatever the first H2
of the original analysis is). On subsequent updates, this header already exists and should not
be duplicated.

After prepending the update, update the page title to reflect the most recent update date.
Use `notion-update-page` with `update_properties` to set the title to:
`Claude: [Company Name] First-Pass Diligence — MM.DD.YYYY Update`

This replaces whatever date suffix was previously in the title (either the original first-pass
date or a prior update date). The title should always reflect the most recent update — do not
stack multiple date suffixes. Strip the old date portion entirely and replace with the new one.

---

## Step 5b: Build the Updated PDF

Generate a new PDF containing the full updated analysis — the new update section(s) prepended
to the complete original analysis. This is a standalone document, separate from the original
first-pass PDF. Do NOT overwrite or modify the original.

### Filename convention

```
[Company]_First_Pass_Diligence_MM.DD.YYYY_Update.pdf
```

Where `MM.DD.YYYY` is today's date (the update date). The original first-pass date is dropped
entirely from the filename — the update date is the only date in the name. Example:
`Tuor_First_Pass_Diligence_04.02.2026_Update.pdf`

### Content

The PDF should contain:
1. **The full update section(s)** — all updates prepended since the original, newest first,
   rendered exactly as they appear on the Notion page.
2. **A page break** — insert a `PageBreak()` flowable before the original memo section.
3. **An H1 header** — `Original First-Pass Memo — [Original Publication Date]` rendered as
   an underlined H1, marking the start of the original analysis.
4. **The full original first-pass analysis** — everything from Framework Mapping through Sources,
   rendered in the same format as the original PDF.

The document is a complete, self-contained artifact. A reader should be able to understand the
full diligence picture from the PDF alone without referencing the Notion page.

### PDF header

- **Title (14pt bold):** `[Company Name] — First-Pass Diligence`
- **Subtitle (9pt italic, dark gray):** `Latest Update: [Update Date] | Initially Published: [Original First-Pass Date] | Notion`
  (where "Notion" is a clickable hyperlink to the Notion page URL). Both dates should be present
  so the reader knows when the original analysis was written and when the most recent update was applied.

### Formatting

Follow the same PDF format spec as the first-pass-diligence skill. Install reportlab if needed:
`pip install reportlab --break-system-packages -q`

Read the formatting spec at `references/pdf-format-spec.md` (located in the `first-pass-diligence`
skill directory) for exact style definitions. Apply the same post-table/post-chart `Spacer(1, 10)`
spacing rule.

**Page numbers:** Add centered page numbers (8pt Helvetica) at the bottom of every page using a
`canvas` callback on `SimpleDocTemplate`:
```python
def page_number(canvas, doc):
    canvas.saveState()
    canvas.setFont("Helvetica", 8)
    canvas.drawCentredString(PAGE_W / 2, 0.4 * inch, f"{doc.page}")
    canvas.restoreState()
```
Pass `onFirstPage=page_number, onLaterPages=page_number` to `doc.build()`.

**Markdown-to-reportlab conversion:** All prose text must pass through a `clean_text()` function
that converts markdown formatting to reportlab XML before being wrapped in `Paragraph()`:
- `***text***` → `<b><i>text</i></b>`
- `**text**` → `<b>text</b>`
- `*text*` → `<i>text</i>`
- `[text](url)` → `<a href="url" color="black"><u>text</u></a>`
This ensures no stray asterisks appear in the rendered PDF.

Save the PDF to:
`/sessions/loving-modest-fermat/Users/tomseo/Downloads/[Company]_First_Pass_Diligence_MM.DD.YYYY_Update.pdf`

### Upload and link in Notion

Upload the updated PDF to the same company subfolder in Google Drive used by the original
first-pass PDF (under Diligence root `1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`). Use the Apps
Script endpoint — never Zapier. Read the full reference at
`/Users/tomseo/.claude/skills/shared-references/drive-upload.md`.

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"
DILIGENCE_ROOT = "1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d"

# createFolder is idempotent — returns existing folder if it already exists
folder_resp = requests.post(DRIVE_URL, json={
    "action": "createFolder",
    "folderName": "<COMPANY_NAME>",
    "parentFolderId": DILIGENCE_ROOT
}, allow_redirects=True, timeout=60)
subfolder_id = folder_resp.json()["folderId"]

with open(pdf_path, 'rb') as f:
    pdf_b64 = base64.b64encode(f.read()).decode('utf-8')

upload_resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": "[Company]_First_Pass_Diligence_MM.DD.YYYY_Update.pdf",
    "fileBase64": pdf_b64,
    "mimeType": "application/pdf",
    "folderId": subfolder_id
}, allow_redirects=True, timeout=120)
file_url = upload_resp.json()["url"]
```

After upload, link the updated PDF in two places on the Notion opportunity page:

1. **Page body** — append to the `## 📎 Diligence Materials` section:
   ```
   - [**[Company]_First_Pass_Diligence_MM.DD.YYYY_Update.pdf**](https://drive.google.com/file/d/<fileId>/view) — Claude diligence priors update, [update date]
   ```

2. **Diligence Materials Files property field** — follow the shared reference at
   `/Users/tomseo/.claude/skills/shared-references/add-link-to-diligence-materials.md`. Pass the opportunity
   page ID, the Drive file URL, and display name
   `[Company]_First_Pass_Diligence_MM.DD.YYYY_Update.pdf`.

   If Chrome is unavailable, skip the property field and rely on the page body link.

Act autonomously — do not ask for permission. Report what was done in the summary.

---

## Step 6: Send Signal Alert

Read the `send-alert` skill (`**/send-alert/SKILL.md`) for the chatID and guardrails.

Use `send_message` (Beeper MCP) with the chatID from the send-alert skill. Compose the body:

```
🔄 [Company Name] — Diligence Priors Updated

New information processed:
- [Source 1 one-liner]
- [Source 2 one-liner]

Net assessment: [One sentence — more convicted, less convicted, or thesis unchanged]

Notion: [Notion page URL]
PDF: [Google Drive URL to updated PDF]
```

---

## Quality Bar

A good prior update:

- Maps every piece of new information to a specific prior, open question, or section from the
  original analysis
- Is direct about whether evidence reinforces or challenges the thesis — no hedging for comfort
- Uses tables when the new information includes quantitative data or metrics
- Resolves open questions explicitly rather than leaving them hanging
- Introduces new questions that the new information raises
- Ends with a clear net assessment that a reader can act on

A bad prior update just summarizes the new call notes or materials without connecting them back
to the original framework. If you find yourself writing "the founder discussed X" without then
saying "which [reinforces/challenges] the prior that Y," stop and add the analytical layer.
