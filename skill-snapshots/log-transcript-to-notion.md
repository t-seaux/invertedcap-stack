---
name: log-transcript-to-notion
description: Rip a YouTube (or other video) transcript and save it as a new entry in the Notion Notes database. Trigger whenever the user says "log this transcript", "save this transcript to Notion", "add this transcript to Notion", "log the transcript", "transcript to Notion", or provides a YouTube URL with any intent to archive or log it. Also trigger when the user has already ripped a transcript in the current conversation and asks to save or log it. Always trigger inline — no confirmation needed before acting.
---

# Log Transcript to Notion (Notes Database)

Rip a transcript from a video URL (or accept a pre-existing transcript from the current conversation), then create a new page in the ✏️ Notes database.

**Notes database data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`

---

## Step 1: Get the Transcript

**If a transcript was already ripped in the current conversation**, use that text directly — do not re-rip.

**If a URL is provided (and no transcript is already in context)**, rip it using `yt-dlp`:

```bash
pip install yt-dlp --break-system-packages -q

yt-dlp --no-check-certificate \
  --write-auto-sub --skip-download \
  --sub-format ttml --convert-subs srt \
  -o "/home/claude/transcript" \
  "<URL>"
```

Then clean the raw `.srt` output into plain text:

```python
import re

with open('/home/claude/transcript.en.srt', 'r') as f:
    content = f.read()

lines = content.split('\n')
cleaned = []
for line in lines:
    line = line.strip()
    if re.match(r'^\d+$', line): continue
    if re.match(r'^\d{2}:\d{2}:\d{2}', line): continue
    if line:
        line = re.sub(r'<[^>]+>', '', line).strip()
        if line:
            cleaned.append(line)

# Deduplicate consecutive identical lines (auto-caption artifact)
deduped = []
prev = None
for line in cleaned:
    if line != prev:
        deduped.append(line)
        prev = line

transcript_text = re.sub(r'  +', ' ', ' '.join(deduped))
```

If `yt-dlp` fails (private video, no captions, SSL error that persists), inform the user and stop.

---

## Step 2: Extract Metadata

Run `yt-dlp --dump-json` to pull structured metadata from the video:

```python
import subprocess, json

result = subprocess.run(
    ['yt-dlp', '--no-check-certificate', '--dump-json', '<URL>'],
    capture_output=True, text=True
)
meta = json.loads(result.stdout)

# Extract publish date
upload_date = meta.get('upload_date', '')  # format: YYYYMMDD
if upload_date and len(upload_date) == 8:
    from datetime import datetime
    publish_date = datetime.strptime(upload_date, '%Y%m%d').strftime('%b %d, %Y')
    # e.g. "Feb 27, 2026"
else:
    publish_date = None

# Extract show/channel info
uploader = meta.get('uploader', '')
channel = meta.get('channel', '')
title = meta.get('title', '')
```

From the metadata and the transcript content itself, extract:

- **Source title**: The video title or show name. Use `uploader` or `channel` from metadata to identify the show/source.
- **Speaker / Guest**: The primary speaker or interviewee. Identify from transcript introductions, the video title, or channel metadata.
- **Source URL**: The original video URL provided by the user.
- **Publish date**: From `upload_date` in the `--dump-json` output, formatted as `Mon DD, YYYY` (e.g. `Feb 27, 2026`).
- **Topic summary**: A 1-2 sentence summary of the transcript's subject matter. Infer from the content — do not ask the user.

If the `--dump-json` step was skipped because a pre-existing transcript was used, run it now against the URL to get the publish date. Only omit the publish date if no URL is available at all.

---

## Step 3: Determine the Note Title

**Required title format:**
```
Transcript: <Speaker> — <Show/Source (Mon DD, YYYY)>
```

Examples:
- `Transcript: Stan Druckenmiller — Hard Lessons (Morgan Stanley, Feb 27, 2026)`
- `Transcript: Bill Gurley — Above the Crowd Podcast (Feb 14, 2026)`
- `Transcript: <Speaker> — <Video Title (Mon DD, YYYY)>` (fallback if show name unclear)

**The publish date is required.** Always pull it from `yt-dlp --dump-json` (the `upload_date` field), formatted as `Mon DD, YYYY` (e.g. `Jul 15, 2025`). Do not omit or approximate it. If the dump-json step was skipped because a pre-existing transcript was used, run it now against the URL to get the date. Only omit if no URL is available at all.

If the user specifies a title, use it verbatim. Do NOT ask the user to confirm the inferred title — proceed directly.

---

## Step 4: Identify Related Opportunity (optional, best-effort)

If the transcript is clearly about a specific company or deal in Tom's portfolio or pipeline, search for the matching Opportunity:

```
Tool: notion-search
Query: <company name>
```

If a confident single match is found in the Opportunities database (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`), include its URL in the `Opportunity` property. If ambiguous or no match, leave it blank.

---

## Step 5: Build the Page Content

Structure the page body as follows:

```
🎬 **Source:** [<Source Title>](<source_url>)
👤 **Speaker:** <Speaker Name>
📝 **Summary:** <1-2 sentence topic summary>

---

**Frameworks**

<frameworks content — see below>

---

**Transcript**

<formatted transcript — see below>
```

If the source URL is unavailable, omit the link and just write the source title as plain text. If the speaker is unknown, omit that line.

---

### Frameworks Section

Use `**Frameworks**` as a bolded text label (not a Markdown header `##`). Under it, identify 3-6 key mental models, theses, or analytical frameworks the speaker articulates in the transcript. For each framework:

- State it as a bolded short title (e.g. `**Liquidity as the Dominant Variable**`)
- Follow with 2-4 sentences explaining the framework in the speaker's own logic
- Ground it with a specific example or quote from the transcript (paraphrased or briefly quoted) to anchor it to the actual content — do not write frameworks in the abstract

---

### Transcript Section

Use `**Transcript**` as a bolded text label (not a Markdown header `##`). Format the transcript body as follows:

- **Speaker labels**: Bold initials-based labels followed by a colon — e.g. `**SD:**`, `**IB:**`, `**Host:**`. Derive initials from speaker names identified in Step 2. If only one speaker is known, use their initials; label others as `**Q:**` or `**Host:**` as appropriate.
- **Turn structure**: Each speaker turn is its own paragraph, separated by a blank line. Do not run turns together into a single block.
- **Narrator / music cues**: Render in italics inline — e.g. *[music]*, *[applause]*, *[narrator: intro segment]*.
- **No `>` blockquote prefixes**: Never use Markdown blockquote `>` syntax. Speaker turns are regular text paragraphs, not blockquotes.
- **No fake dialogue**: Do not invent speaker attribution not present or clearly inferable from the captions. If speaker identity is ambiguous, use `**Speaker:**` as a generic label.

If the transcript is very long and truncation is unavoidable, note at the bottom:
`[Note: transcript truncated due to length]`

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

> ✓ Transcript logged to Notes: **[Note Title]** → [Notion page URL]

---

## Error Handling

- **yt-dlp SSL failure (persistent):** Inform the user. Suggest running `yt-dlp --no-check-certificate` manually and pasting the transcript text directly.
- **No captions available:** Inform the user the video has no auto-generated captions. Suggest they provide a transcript manually.
- **Notion MCP unavailable:** Inform the user and suggest they log the transcript manually.
- **Title unclear:** Default to `Transcript: [video ID]` rather than asking.


---

## Final Step: Classify the Note

After the Notion page has been created and the Opportunity relation (if any) has been set, run the `note-classifier` skill to assign the correct `Category` field.

Read the skill at `/Users/tomseo/.claude/skills/note-classifier/SKILL.md` and follow its classification logic against the note just created. Do not skip this step — every note created by this skill must have a Category set.
