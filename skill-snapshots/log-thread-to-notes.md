---
name: log-thread-to-notes
description: Log a shared TEXT/CHAT thread — iMessage/WhatsApp/SMS screenshot(s) (incl. a stitched multi-shot) or pasted messages — to the Notion ✏️ Notes DB in the text-thread shape: giver-first title, a Summary on top, and the thread transcribed VERBATIM under ## Raw Thread. CONTEXT-DEPENDENT trigger — "log" / "log to notes" / "log in notes" / "log thread" routes HERE only when the object is a shared/attached text-thread screenshot or pasted third-party thread. If the object is a video/interview/YouTube URL → log-transcript-to-notion; a letter/report/memo link or PDF → log-document-to-notes (investor-firm letter → log-investor-letter-to-notion). If Tom says a bare "log" during an ongoing Claude session with NO object attached (we've been conversing), that's add-conversation-to-notion (logs the Claude session), NOT this. Also the "log" half of "stitch and log" (stitch first, then this). Always trigger inline — no confirmation needed.
---

# Log Thread to Notes

Create a ✏️ Notes DB row for a text/chat thread Tom shares, transcribed verbatim with a summary on top. This is the generalization of the feedback-note text-thread shape to ANY thread (diligence, portfolio, personal-professional), per [[text-thread-image-notes-transcribe-verbatim]].

**Notes data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb` · **Opportunities:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`

## Step 1 — Get the thread verbatim
Source is either screenshots or pasted text.
- **Screenshots:** read EVERY image at full resolution (not a downscaled stitch/tile) so the transcription is exact. Speakers: gray/left bubble = the other party; blue/right = Tom. Capture every message including attachments/links/charts.
- Never fabricate timestamps: if per-message times aren't shown, use date-only prefixes and say so.

## Step 2 — Title (giver-first)
`<Giver Name>: <Subject> Thoughts` for an unsolicited take (e.g. `Bobby Kwon: EvenUp Series F Thoughts`); `Feedback` / `Reference` when it's solicited diligence feedback (mirrors feedback-note-format). Use Tom's title verbatim if he gives one. Don't ask to confirm.

## Step 3 — Body (read the canonical shape first)
Read `~/.claude/skills/shared-references/feedback-note-format.md` §"Text-thread feedback — summary on top, ## Raw Thread underneath" and follow it:
- **Top line (if a source image was saved to Drive):** `📎 [<title> — source](<drive file url>)`. (Plain "log" with no Drive image → skip the link; the transcript is the record.)
- **`## Summary`** — bulleted synthesis of the substance/read, not a play-by-play.
- **`## Raw Thread`** — verbatim: each message a `**[YYYY-MM-DD HH:MM] Speaker:**` paragraph, blank line between, **never a fenced code block**. Bracket attachments/links `[attachment — …]` / `[link — …]`, and TRANSCRIBE any chart/table as readable text (Tom reads the note, not just the image).

## Step 4 — Create + relate
`notion-create-pages` into the Notes data source with `Name`, `⭐️`="__NO__", and the body. Best-effort `Opportunity` relation: `notion-search` the company/subject; on a confident single Opportunities match, set it at creation (else leave blank for the auto-tagger). **Verify by readback** (`notion-fetch`) — Notion MCP writes can silently no-op (see [[reference_notion_tooling]]).

## Step 5 — Classify + confirm
Run `note-classifier` (`~/.claude/skills/note-classifier/SKILL.md`) to set `Category` (diligence threads → Diligence). Then one line: `✓ Saved to Notes: **<title>** → <url>`.

## Chaining — "stitch and log"
When Tom says "stitch and log" (or stitches then says "log it"):
1. **Stitch** the screenshots (`~/.claude/scripts/stitch/`). Because the note must link a clickable image, upload the stitched PNG to **Drive** (not the Downloads default): the relevant company's diligence folder `Diligence/<Company>/` (diligence root `1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d` → `createFolder` the subfolder), inferred from the thread's Opp — or wherever Tom names. Capture the returned Drive file URL.
2. **Log** via Steps 1–5, using that Drive URL as the Step-3 `📎 source` line, and the same transcription for `## Raw Thread`.

This is exactly the EvenUp/Bobby Kwon flow: stitched image → `Diligence/EvenUp/`, note `Bobby Kwon: EvenUp Series F Thoughts` with Summary + verbatim Raw Thread linking it.
