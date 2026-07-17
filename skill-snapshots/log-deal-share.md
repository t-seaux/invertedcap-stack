---
name: log-deal-share
description: Log a deal share — a pipeline/dealflow list an investor or firm sends Tom for reciprocal sourcing — as a note in the Notion ✏️ Notes database, with the shared materials converted to PDF and embedded inline at the top of the note. Trigger when Tom says "log this deal share", "log the deal share note", "save this dealflow", "log the pipeline [X] sent", "add this to Notion as a deal share", or forwards/points at an email where a VC, angel, or fund shares their deal flow, pipeline, or portfolio list and asks for referrals in return. Also trigger on "save down this email and attachment and add it to a Notion note" when the email is a dealflow share. Distinct from investor-update (portfolio company reporting to Tom), materials-handler (per-company diligence docs), and add-to-crm (creating an Opportunity) — a deal share creates a Notes row only, never an Opportunity.
---

# Log Deal Share

An investor shares their dealflow pipeline with Tom, usually as a reciprocal-sourcing gesture ("send us anything raising pre-seed/seed and we'll do the same"). This skill captures the email **and** the shared materials as a single self-contained Notion note, so the pipeline is searchable later without going back to the sender's link — which will have gone stale or been overwritten in place.

**Notes database data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`
**People database data_source_id:** `1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`

**Core principle: the note must survive the link dying.** Deal share links are rolling documents — the sender overwrites them every few weeks. Always transcribe the pipeline contents into the note body *and* embed the PDF. Never leave the note as a bare link.

---

## Step 1: Get the Email

If Tom names a sender ("the email from Will at Portfolio Ventures"), find it:

```
mcp__Gmail__search_threads  →  query: "from:<sender or domain> newer_than:30d"
mcp__Gmail__get_thread      →  threadId, messageFormat: FULL_CONTENT
```

Use `get_thread` with `FULL_CONTENT`, not the search snippets — you need the full body and the links inside it. From the message, extract:

- **Sender**: name, title, firm, email address (the signature block usually has all four)
- **Date received**
- **Gmail permalink**: `https://mail.google.com/mail/u/0/#inbox/<messageId>`
- **The materials link** — DocSend, Papermark, Google Drive, Notion, or a native attachment
- **Cadence** — how often the sender says they refresh the pipeline

---

## Step 2: Convert the Shared Materials to PDF

The "attachment" on a deal share is usually **a link, not a file**. Check what you're dealing with:

| Link shape | Path |
| --- | --- |
| `docsend.com/view/<slug>` | `docsend-to-pdf` skill, single-doc flow |
| `docsend.com/view/s/<slug>` | `docsend-to-pdf` skill, data room flow |
| `papermark.com/view/<id>` | `docsend-to-pdf` skill, Papermark recipe |
| Native Gmail attachment | Download via the Gmail attachment path in `materials-handler` |
| Google Doc / Sheet / Slides | Export to PDF via the Drive API path in `materials-handler` |

Read `/Users/tomseo/.claude/skills/docsend-to-pdf/SKILL.md` and follow it — do not re-derive the conversion. Save to `/Users/tomseo/Downloads/<Firm> - <Doc Title> (<YYYY-MM-DD>).pdf`.

**Do NOT upload to Drive.** Deal shares are firm-level, not company-level — they don't belong in `Diligence/<Company>/` or `Deal Docs/<Company>/`, and inventing a third Drive namespace for them is not worth it. The PDF lives embedded in the Notion note. The local Downloads copy is transient.

### Read the PDF before writing the note

DocSend PDFs are **page images with no text layer** — `pdftotext` returns nothing. Use the `Read` tool with a `pages` range to read them visually. You must actually read the deck; the note's value is the transcription, and you cannot transcribe what you haven't read.

---

## Step 3: Look Up the Sender in the People DB

Search for the sender by name. If they have a row in the **People** database (`collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`), hyperlink their name in the Source table to their People page.

```
mcp__Notion__notion-search  →  query: "<sender name> <firm>"
```

Verify the hit is actually a People-DB row (its `<ancestor-path>` shows `parent-data-source … name="People"`) — a search for a person's name will also surface call notes *about* them.

**Source line format:**

- **In the People DB:** `[<Name>](<people page url>), <Title> @ <Firm>`
- **Not in the People DB:** `<Name>, <Title> @ <Firm>` — plain text, no link

Do **not** create a People row as part of this skill. If Tom wants the sender added, that's `add-to-contacts`.

---

## Step 4: Upload the PDF to Notion

⚠️ **The Notion MCP connector and the internal API token are two different integrations.** A `file_upload` created with one is invisible to the other. If you upload with the internal token and then reference `file-upload://<id>` in `notion-create-pages`, the MCP returns:

> `object_not_found: Could not find file_upload with ID: ... Check that you have access and that you're authenticated to the correct workspace.`

This is not a permissions bug to debug — it is the expected behavior of two separate integrations. The working sequence is **create the page with the MCP → upload and attach with the internal token.** The MCP's `notion-create-attachment` tool cannot help either: it only accepts inline text content or a *publicly reachable* HTTPS URL, and a local PDF is neither.

Uploads expire in **1 hour** if never attached — do Steps 4 through 6 in one pass.

---

## Step 5: Create the Note

Use `notion-create-pages` against the Notes data source with the body from Step 6.

```json
{
  "parent": { "data_source_id": "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" },
  "pages": [{
    "properties": {
      "Name": "<Firm>: Deal Flow Share (<Month DD, YYYY>)",
      "Category": "Other"
    },
    "content": "<page body from Step 6>"
  }]
}
```

- **Title format:** `<Firm>: Deal Flow Share (<Month DD, YYYY>)` — colon after the firm, full spelled-out date of the email. E.g. `Portfolio Ventures: Deal Flow Share (July 14, 2026)`. The date matters: these arrive on a cadence and Tom accumulates one row per share from the same firm. Never overwrite a prior share — each is a point-in-time snapshot.
- **Category:** `Other`. A deal share is neither Research, Diligence, Portfolio, nor Artifact.
- **Opportunity relation:** leave blank by default. A deal share is firm-level. Only set it if the *email itself* is substantively about one of Tom's deals — e.g. a co-investor confirming they're committing to a company Tom led — and even then, ask before linking.

Then set the icon — read `/Users/tomseo/.claude/skills/shared-references/claude-note-icon.md` and apply `:claude-color:` via `notion-update-page`. Required for every Claude-authored note.

---

## Step 6: Page Body Structure

Four sections, in this order. **Attachment comes first** — the deck is the artifact, and Tom wants it above the fold.

```markdown
## Attachment

## Source

| | |
| --- | --- |
| **Shared by** | [<Name>](<people page url>), <Title> @ <Firm> |
| **Date received** | <Month DD, YYYY> |
| **Channel** | Email — ["<Subject>"](<gmail permalink>) |
| **Cadence** | <how often the sender refreshes the pipeline> |

## <Deck Title — verbatim from the deck, e.g. "PV Dealflow — July 2026">

### <Sender's own sub-heading, e.g. "Pre-Seed Deals">
- **<Company>** — <one-line description>. <Status: term sheet agreed / who led / traction.>

### <Sender's next sub-heading, e.g. "Seed Deals">
- **<Company>** — <one-line description>. <Status.>

### <Every other sub-heading the deck has, e.g. "Companies and founders we've heard great things about">
- <verbatim list>

## Original email

**From:** <Name> (<email>)
**Date:** <Month DD, YYYY>
**Subject:** <subject>

<The email body, verbatim, as plain paragraphs and lists. Keep the sender's own
hyperlinks. Strip the legal/confidentiality footer. Replace the materials link
with a pointer to the PDF attached above — the link is transient, the PDF is the
artifact.>
```

**Mirror the sender's own breakdown.** Do not impose your own taxonomy on their pipeline. If they split it Pre-Seed / Seed / "heard great things about", the note uses those exact sub-headings in that order. If they split by sector, or by conviction, or not at all — follow them. The section header is the deck's own title.

**Transcribe the whole pipeline.** Every company, its one-liner, and its status (term sheet agreed, who led the prior round, any traction number). That transcription is the reason this note exists — it makes the pipeline greppable in Notion long after the sender has overwritten the link.

**No blockquotes in the Original email section.** Plain paragraphs, not `>` prefixes.

**No live DocSend URLs anywhere in the body.** Per `docsend-to-pdf`: once converted, the PDF is the permanent artifact and the DocSend link is transient input.

---

## Step 7: Embed the PDF Under the Attachment Heading

`PATCH /v1/blocks/{page_id}/children` appends to the **end** of a page by default. To land the PDF under the `## Attachment` heading at the top, find that heading's block id and pass it as `after`.

Use the **internal token** — the same integration that will make the upload.

```python
import sys, requests
sys.path.insert(0, '/Users/tomseo/.claude/scripts')
from notion_files_property import _load_token

token = _load_token()
H  = {"Authorization": f"Bearer {token}", "Notion-Version": "2022-06-28"}
HJ = {**H, "Content-Type": "application/json"}

# 1. Find the "Attachment" heading to anchor against
blocks = requests.get(f"https://api.notion.com/v1/blocks/{PAGE_ID}/children?page_size=100",
                      headers=H, timeout=60).json()["results"]
anchor = next(b["id"] for b in blocks
              if b["type"].startswith("heading")
              and "".join(t["plain_text"] for t in b[b["type"]]["rich_text"]).strip().lower() == "attachment")

# 2. Upload the PDF
uid = requests.post("https://api.notion.com/v1/file_uploads", headers=HJ,
                    json={"filename": FNAME, "content_type": "application/pdf"},
                    timeout=60).json()["id"]
with open(PATH, "rb") as f:
    requests.post(f"https://api.notion.com/v1/file_uploads/{uid}/send", headers=H,
                  files={"file": (FNAME, f, "application/pdf")}, timeout=120)

# 3. Attach it directly below the heading
requests.patch(f"https://api.notion.com/v1/blocks/{PAGE_ID}/children", headers=HJ, json={
    "after": anchor,
    "children": [{"object": "block", "type": "pdf", "pdf": {
        "type": "file_upload", "file_upload": {"id": uid},
        "caption": [{"type": "text", "text": {"content": FNAME}}]}}]
}, timeout=60)
```

A `pdf`-type block renders as a **scrollable inline preview**, not a download chip — the right call when the deck IS the content.

---

## What this skill does NOT do

- **No cross-referencing against the Companies or Opportunities DBs.** Tom does the matching by eye. Do not check which of the named companies he already knows, and do not flag overlaps.
- **No auto-adding to Companies, the -1 Scanner, or the CRM.** A deal share is somebody else's pipeline, not Tom's. Creating rows from it would pollute his DBs with low-conviction names. If Tom wants a specific company from the list added, he'll say so — then use `add-to-companies` or `add-to-crm`.
- **No editorializing.** No "Why it matters", no "The ask", no summary of the sender's focus areas. The email says all of it and the email is reproduced at the bottom. Capture, don't interpret.
- **No Drive upload.** See Step 2.
- **No People row created** for the sender. See Step 3.
- **No reply drafted.** If Tom wants to reciprocate, that's a separate ask.

---

## Step 8: Confirm to Tom

One line:

> ✓ Deal share logged: **[<Firm>: Deal Flow Share (<Month DD, YYYY>)]**(<url>) — <N> pre-seed, <N> seed, PDF embedded.

If the email contained something materially new beyond the pipeline — a commitment to one of Tom's deals, a change in the sender's focus — surface it in your reply. Tom should not have to open the note to learn the one thing that actually changed.

---

## Error Handling

- **DocSend email gate fails:** the sender may have restricted the link to a verified email. Fall back to driving Tom's authenticated Chrome tab (see the data-room section of `docsend-to-pdf`).
- **`object_not_found` on a `file-upload://` reference in the MCP:** you uploaded with the wrong integration. See the warning in Step 4 — this is the expected failure, not a bug.
- **Upload expired (1 hour):** re-run the upload to mint a fresh id, then re-attach.
- **PDF has no text layer:** expected for DocSend. Use `Read` with a `pages` range, not `pdftotext`.
- **Sender not found in People DB:** use the plain-text `<Name>, <Title> @ <Firm>` format. Do not create the row.
