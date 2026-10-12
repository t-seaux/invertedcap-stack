---
name: log-thread-to-notes
description: Log a shared TEXT/CHAT thread — iMessage/WhatsApp/SMS screenshot(s) (incl. a stitched multi-shot) or pasted messages — to the Notion ✏️ Notes DB in the text-thread shape: giver-first title, a Summary on top, and the thread transcribed VERBATIM under ## Raw Thread. A person's read on an Opportunity is collated into their ONE feedback note via feedback_write.py (same writer as the text lane). CONTEXT-DEPENDENT trigger — "log" / "log to notes" / "log in notes" / "log thread" routes HERE only when the object is a shared/attached text-thread screenshot or pasted third-party thread. If the object is a video/interview/YouTube URL → log-transcript-to-notion; a letter/report/memo link or PDF → log-document-to-notes (investor-firm letter → log-investor-letter-to-notion). If Tom says a bare "log" during an ongoing Claude session with NO object attached (we've been conversing), that's add-conversation-to-notion (logs the Claude session), NOT this. Also the "log" half of "stitch and log" (stitch first, then this). Always trigger inline — no confirmation needed.
---

# Log Thread to Notes

Create a ✏️ Notes DB row for a text/chat thread Tom shares, transcribed verbatim with a summary on top. This is the generalization of the feedback-note text-thread shape to ANY thread (diligence, portfolio, personal-professional), per [[text-thread-image-notes-transcribe-verbatim]].

**Notes data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb` · **Opportunities:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`

## Step 0 — Route: is this someone's read on an Opp? (feedback lane, same brain as text)

A thread where a person gives their read on an Opportunity is a **feedback note**, not a standalone thread note, and it
goes through the ONE feedback writer every lane uses — `shared-references/feedback_write.py` (contract:
`feedback-note-format.md`; text twin: `deal-text-scanner/references/feedback-lane.md` Direction A). Never hand-write it
with `notes_create.py` or the Notion MCP: a hand-made page breaks one-note-per-(person, Opp) collation, skips the
`📣 Pending Feedback` clear, and diverges from what the text lane writes for the same thread.

| The thread is… | Path |
|---|---|
| **Solicited** — Tom asked this person for a read on the Opp (in this thread or elsewhere), or the person is on the Opp's `📣 Pending Feedback` | `log-response` (below) |
| **Volunteered**, and this person already has a note on the Opp | `log-response` (below) — volunteered feedback still collates onto the existing note |
| **Volunteered**, no existing note (e.g. `Bobby Kwon: EvenUp Series F Thoughts`), or no confident single Opp | Standalone thread note — Steps 1–4 |

1. Resolve the Opp (`notion-search`, confident single match only) and each giver's People row (People DB guardrails:
   missing row → `people_db_ask.py`, never create). **One run per giver** — a group thread with two people giving reads
   is two `log-response` calls, each body summarizing THAT person's read over the same verbatim Raw Thread.
2. Classify each giver's read: `substantive` / `decline` / `deferral` (rubric: `feedback-outreach-scanner` Step 2.3).
3. Write the Response body to a scratch file: a short bulleted summary of the read, then `### Raw Thread` with every
   message as a `**[YYYY-MM-DD HH:MM] Speaker:** text` paragraph (Step 1 rules; voice notes → `--channel "voice note"`).
4. Run:
   ```
   python3 ~/.claude/skills/shared-references/feedback_write.py log-response --opp-id <opp> --person-id <People page> \
       --kind substantive|decline|deferral --channel text --date <YYYY-MM-DD of the read> --body-file <file> \
       --surface text|slack --manual
   ```
   `--manual` because Tom asked for this log himself (the contract's one Status-gate override; never in a scheduled
   lane). `--surface slack` only when Tom asked from Slack; a Claude session or a text uses `text`.
   - **exit 0** → reply with its `alert` VERBATIM (deferral / `already_logged` with no changes: no `alert` → one line
     `✓ Logged to <url>`). If `gate_overridden` is set, add one line naming the Opp's status.
   - **exit 3** → no note yet. Solicited → `feedback_write.py create --opp-id <opp> --person-id <P> --kind feedback|reference
     --subject "<Opp co | person referenced>" --outreach-file <Tom's ask, verbatim from the thread; if the ask isn't in the
     shared thread, the single line `_Ask made outside this thread._`> --outreach-date <date of the ask>
     --outreach-tag "text thread" --manual`, then re-run `log-response`. Volunteered → standalone thread note (Steps 1–4).
   - **exit 5** → fix the body per `failures` and re-run · **4 / 2** → it did NOT log; say so and why, never ✓.

## Step 1 — Get the thread verbatim
Source is either screenshots or pasted text.
- **Screenshots:** read EVERY image at full resolution (not a downscaled stitch/tile) so the transcription is exact. Speakers: gray/left bubble = the other party; blue/right = Tom. Capture every message including attachments/links/charts.
- Never fabricate timestamps: if per-message times aren't shown, use date-only prefixes and say so.

## Step 2 — Title (giver-first)
`<Giver Name>: <Subject> Thoughts` for an unsolicited take (e.g. `Bobby Kwon: EvenUp Series F Thoughts`). Solicited diligence feedback never reaches this step — Step 0 routes it to `feedback_write.py`, which owns the `Feedback` / `Reference` title. Use Tom's title verbatim if he gives one. Don't ask to confirm.

## Step 3 — Body (read the canonical shape first)
Standalone thread notes reuse the text-thread shape from `~/.claude/skills/shared-references/feedback-note-format.md` §"Text-thread feedback" (there the Raw Thread is a `###` under the dated Response; here it is a top-level `##`):
- **Top line (if a source image was saved to Drive):** `📎 [<title> — source](<drive file url>)`. (Plain "log" with no Drive image → skip the link; the transcript is the record.)
- **`## Summary`** — bulleted synthesis of the substance/read, not a play-by-play.
- **`## Raw Thread`** — verbatim: each message a `**[YYYY-MM-DD HH:MM] Speaker:**` paragraph, blank line between, **never a fenced code block**. Bracket attachments/links `[attachment — …]` / `[link — …]`, and TRANSCRIBE any chart/table as readable text (Tom reads the note, not just the image).

## Step 4 — Create, classify, confirm

**One command does the write — never `notion-create-pages` / `notion-update-page` for this page** (Costanoa 10/3: a
hand-rolled MCP create passed `parent_id`, the page landed private at workspace root, and the full text, Category and
icon were all skipped while "✅ Logged" went out anyway). Write the title and body from the steps above to a markdown
file in your scratch dir, then:

```
python3 ~/.claude/skills/shared-references/notes_create.py --kind thread --title "<title>" --body <file.md> \
    [--url <source url>]... [--opp <Opportunities page id>] --category <C> [--title-verbatim]
```

The script does dedup, the title and body shape checks, the create (parent hardcoded to the Notes DB), ⭐️ off,
Opportunity, Category (note-classifier code when `--opp` is set), the Claude icon, and the readback. Pass `--opp` only on a
confident single Opportunities match (`notion-search`); `--title-verbatim` when Tom dictated the title.

- **exit 0** → send its `confirm` line VERBATIM. That is the only path to a ✓.
- **exit 10** → already logged: `✓ Already logged: **<existing.title>** → <existing.url>`. Create nothing.
- **exit 3** → fix the CONTENT that `failures` names (missing section, title shape, fenced block) and re-run.
- **exit 4 / 2** → it did NOT log (exit 4 means the page was archived again). Tell Tom it failed and why. Never ✓.
- **`--category`** (required when there's no `--opp`): diligence threads → Diligence; otherwise note-classifier Steps 4/5. Thread notes get no Claude icon (they're the giver's words); the script handles that.

## Step 5 — Intro landed in the thread → `Connected`
If the logged thread is tied to an existing Opp (`--opp`) and shows the intro actually landing (the referrer's "please
meet X" with both parties in it, the founder texting Tom directly), flip Status in the same run — the rule and the
command are owned by `materials-handler/SKILL.md` Step 4.8 (`opp_status.py next --event intro_landed` → `set`; exit 3 =
no move). Ambiguous ("I'll intro you at some point") → no flip.

## Chaining — "stitch and log"
When Tom says "stitch and log" (or stitches then says "log it"):
1. **Stitch** the screenshots (`~/.claude/scripts/stitch/`). Because the note must link a clickable image, upload the stitched PNG to **Drive** (not the Downloads default): the relevant company's diligence folder `Diligence/<Company>/` (diligence root `1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d` → `createFolder` the subfolder), inferred from the thread's Opp — or wherever Tom names. Capture the returned Drive file URL.
2. **Log** via Step 0 routing, then Steps 1–4 (standalone) or `log-response` (feedback), using that Drive URL as the `📎 source` line — first line of the note body, or first line of the Response body file on the feedback path — and the same transcription for the Raw Thread.

This is exactly the EvenUp/Bobby Kwon flow: stitched image → `Diligence/EvenUp/`, note `Bobby Kwon: EvenUp Series F Thoughts` with Summary + verbatim Raw Thread linking it.
