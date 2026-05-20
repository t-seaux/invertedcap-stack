---
name: log-pass-note-guidance
description: >
  Log Tom's free-form Pass Note Guidance — scratch notes or transcribed voice notes that
  give a general read on how he wants the pass note drafted — to the body of an Opportunity
  in the Notion Opportunities DB. The guidance lands as a section at the TOP of the page
  body, formatted so `pass-note-drafter` Step 3f picks it up as authorial intent when Tom
  later flips Status to "Pass Note Pending". Trigger when Tom says "log pass note guidance
  [on/for X]", "pass note guidance for X", "add pass note guidance to X", "guidance for the
  pass note on X", "log this as pass note guidance", or any variant where he names an
  Opportunity together with scratch/voice/bulleted thoughts intended to steer the eventual
  pass note. Also trigger when Tom pastes a screenshot of bulleted notes with intro text
  like "Pass Note Guidance" and names a target Opportunity. Always trigger inline — no
  confirmation needed.
---

# Log Pass Note Guidance (Manual)

Append Tom's scratch/voice/bulleted thoughts as a **Pass Note Guidance** section at the
top of an Opportunity page body in Notion. This is the upstream writer for
`pass-note-drafter` Step 3f's mandatory guidance check — the section is later read as
direct authorial intent that outranks call notes, materials, and first-pass diligence when
substance conflicts.

## Why this exists

Tom often forms a quick read on *why* he's passing — and *how* he wants the note to land —
before he formally flips Status to "Pass Note Pending". He wants somewhere to dump those
thoughts on the Opp itself so the drafter consumes them when the time comes. This skill is
the only sanctioned writer of that section.

This is the **manual, direct** path. There is no scheduled or webhook variant — Tom always
invokes explicitly.

## The Notion Data Model

- **Opportunities DB:** `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
- **Target field:** page body — section header is a bolded paragraph `Pass Note Guidance`,
  followed by bullets. (Pass-note-drafter Step 3f accepts both `heading_2`/`heading_3`
  variants and bolded-paragraph; this skill standardizes on bolded paragraph because that's
  the format Tom uses in the screenshots/notes he hands over.)

## Section Format (canonical — must match what pass-note-drafter recognizes)

The section MUST be written so pass-note-drafter Step 3f detects it. Two acceptable forms:

**Form A — bolded paragraph header (default; use this for new sections):**
```
**Pass Note Guidance**

- {bullet 1}
- {bullet 2}
- ...
```

**Form B — heading (only if an existing section already uses a `## Pass Note Guidance`
or `### Pass Note Guidance` header from a prior hand-edit by Tom):** preserve the existing
header style when appending.

Casing/punctuation variants Tom uses interchangeably: `pass note guidance`,
`Pass-Note Guidance`, `Pass note guidance`. Treat all as the same section — do not create
a second section with a slightly different label.

---

## Execution Workflow

### Step 1: Parse Tom's Input

Extract two things from Tom's message:

1. **Target Opportunity name** — the company / Opp Tom is referencing. May be shorthand
   (e.g., "the Lex one", "Acme") or explicit (e.g., "the Acme Seed Opp").
2. **Guidance bullets** — the actual scratch/voice content. May arrive as:
   - Inline bulleted text in the message
   - A pasted screenshot of bullets (read the image, transcribe the bullets verbatim)
   - A voice-style run-on paragraph (parse into bullets — split on sentence/thought
     boundaries; preserve Tom's wording, do NOT rewrite for style)

**Transcription rule:** Tom's words are sacrosanct here. The section is supposed to capture
*his read* — pass-note-drafter will filter through voice/style on the drafting side. Do
NOT smooth, edit, or "improve" the bullets. Preserve typos, parentheticals, scare quotes,
ellipses, em/en-dash usage, etc. Trim only whitespace and obvious OCR artifacts.

**Voice-to-bullet split rule:** if input is run-on prose, split on natural thought
boundaries (sentence boundaries, "and another thing", "also", "the other thing is",
discourse markers). Each bullet should be one self-contained thought. Do not invent
content; do not merge bullets to "improve flow".

If the target Opportunity is ambiguous (Tom said "the deal we discussed"), ask once for
clarification — this is the one place ambiguity gets a question, because writing to the
wrong Opp is hard to undo cleanly.

### Step 2: Resolve the Opportunity

Search the Opportunities DB for the named company using the two-pass approach (same
pattern as `log-intro`):

**Pass 1 — Scoped DB search:**
```
notion-search with query = "<company name>" and data_source_url = "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"
```

**Pass 2 — Workspace search (if Pass 1 returns no exact match):**
```
notion-search with query = "<company name>", query_type = "internal", content_search_mode = "workspace_search"
```
Filter to Opportunities DB pages only.

**Multiple-entry disambiguation:**
- **(FO) suffix rule:** `Argo (Seed FO)` is a follow-on; guidance about a pass would
  almost never be filed against an FO entry. Default to the non-FO entry unless Tom
  explicitly names the FO. (Cross-ref auto-memory: `project_fo_suffix_routes_to_original_opp.md`.)
- **Same company across funds:** default to the most recent (highest fund number, latest
  Close Date) Opp — opposite of `log-intro`, because pass note guidance is about a
  *current* decision, not a historical relationship. Override only if Tom names a specific
  fund/round.
- **Genuinely ambiguous (e.g., two Opps with same name in same fund):** ask Tom which one.

**No Opp found:** STOP and report back. Do NOT create an Opportunity. Suggest Tom run
`add-to-crm` first if the deal isn't in the pipeline yet.

### Step 2.5: Terminal-Status Guard (soft)

Unlike `log-intro`, pass note guidance is *expected* on Opps in pre-terminal status (Track,
Diligence, etc. — Tom is forming his read before flipping to Pass Note Pending). But also
fine on `Pass Note Pending` itself (Tom adding last-minute guidance before the drafter
runs).

If the Opp's Status is already a fully-terminal pass (`Pass (DNM)`, `Pass (Met)`) or `Lost`
/ `Exited`, the pass note has already been sent (or wasn't going to be). Pause and ask Tom:
`⚠️ [Opp Name] is in terminal status [status] — the pass note has likely already been
drafted/sent. Confirm you still want to add guidance retroactively, and I'll proceed.`

Active portfolio statuses (`Active Portfolio`, `Portfolio: Follow-On`, `Committed`,
`Scheduled`) should also pause — pass note guidance on a deal Tom is invested in is almost
certainly a wrong-Opp resolution. Surface the ambiguity rather than writing.

### Step 3: Fetch the Opp Body and Detect Existing Section

Use `notion-fetch` on the Opp page (no `include_transcript` — this is a routing fetch, not
a transcript-consuming one; per `feedback_notion_fetch_always_include_transcript.md` we
skip the flag here).

Walk the block children and check for an existing **Pass Note Guidance** section
(case-insensitive match on the four canonical variants listed above). The detector should
match:

- A `heading_2` or `heading_3` block whose plain text equals the label
- A `paragraph` block whose entire content is bolded text equal to the label

Capture: (a) whether a section exists, (b) if so, the block ID of the section header and
the IDs/positions of the bullet blocks beneath it (consecutive `bulleted_list_item` blocks
until the next non-bullet block or another heading).

### Step 4: Write the Section

**Case A — No existing section (most common, first invocation):**

Prepend the new section to the TOP of the page body. Notion's REST API has no native
prepend or block-move endpoint, so use the MCP `notion-update-page` with `command:
update_content`, anchored on the first child block's rendered markdown:

```
notion-update-page
  command: update_content
  page_id: <opp_page_id>
  content_updates: [{
    old_str: "<first child markdown verbatim>",
    new_str: "**Pass Note Guidance**\n\n- <bullet 1>\n- <bullet 2>\n...\n\n<first child markdown verbatim>"
  }]
```

Fetch the first child via `notion-fetch` on the page; the first non-empty block's markdown
is the anchor. If that markdown is short/common enough to risk a non-unique match
elsewhere on the page (e.g., a bare `---` divider, a one-word heading), extend the anchor
to include the next block too so the substitution is unambiguous.

**Page is empty (no children):** fall back to `PATCH /v1/blocks/{opp_page_id}/children`
directly with the new blocks array — on an empty page, append === prepend. New blocks
array (one paragraph header + N bulleted_list_items):

```json
[
  {"object":"block","type":"paragraph","paragraph":{
    "rich_text":[{"type":"text","text":{"content":"Pass Note Guidance"},
                  "annotations":{"bold":true}}]}},
  {"object":"block","type":"bulleted_list_item","bulleted_list_item":{
    "rich_text":[{"type":"text","text":{"content":"<bullet 1 verbatim>"}}]}},
  ... one bulleted_list_item per bullet ...
]
```

**Anchor genuinely unresolvable (first-block markdown empty/whitespace AND page non-empty,
which is rare):** append at bottom via the empty-page fallback and surface the placement:
`⚠️ Could not reliably prepend; section appended at bottom of [Opp Name] instead. Reorder
manually if you want it at the top.`

**Case B — Existing section found (append bullets):**

Append the new bullets to the END of the existing section's bullet list. Two valid
approaches; prefer (i):

(i) **MCP `update_content` anchored on the last existing bullet:**
   ```
   notion-update-page
     command: update_content
     page_id: <opp_page_id>
     content_updates: [{
       old_str: "- <last existing bullet text verbatim>",
       new_str: "- <last existing bullet text verbatim>\n- <new bullet 1>\n- <new bullet 2>\n..."
     }]
   ```
   This preserves the existing header style (Form A or Form B).

(ii) **Direct REST `PATCH /v1/blocks/{section_header_block_id}/children`** with the new
    bulleted_list_item blocks as `children`, using the `after` parameter set to the last
    existing bullet's block ID. This appends the new bullets immediately after the existing
    ones, in-section, with no risk of string-matching collisions.

Idempotency: before writing, check whether each new bullet's text already appears in the
existing section (exact substring match, case-sensitive). Skip bullets that already exist
to handle re-runs of the same input. If ALL new bullets are duplicates, report
`⚠️ All bullets already present in [Opp Name]'s Pass Note Guidance — no change made.`

**Case C — Existing section found but section is empty (header only, no bullets):**

Same as Case B but anchor on the section header text instead of the last bullet:
```
old_str: "**Pass Note Guidance**\n\n"  (or "## Pass Note Guidance\n\n" for Form B)
new_str: "**Pass Note Guidance**\n\n- <bullet 1>\n- <bullet 2>\n...\n\n"
```

### Step 5: Verify the Write

Re-fetch the Opp page (cheap — same notion-fetch as Step 3, no transcript) and confirm:
- The section header exists
- All intended bullets are present (substring match against the rendered markdown)

Per `feedback_skill_self_report_diverges_from_actual_write.md`, do NOT trust the write
self-report alone — verify by re-reading. If verification fails, retry once. If it still
fails, surface the failure clearly to Tom rather than reporting success.

### Step 6: Report Back

**Case A success:**
```
✅ Pass Note Guidance added to [Opp Name] ([Fund]):
- <bullet 1>
- <bullet 2>
...
Opp: <Notion URL>
```

**Case B success (appended to existing):**
```
✅ Appended [N] bullet(s) to existing Pass Note Guidance on [Opp Name] ([Fund]):
- <new bullet 1>
- <new bullet 2>
(Existing bullets preserved: [count])
Opp: <Notion URL>
```

**Duplicate skip:**
```
⚠️ All bullets already present in [Opp Name]'s Pass Note Guidance — no change made.
```

**Verification failure:**
```
❌ Write attempted but verification failed for [Opp Name]. Re-check manually:
<Notion URL>
```

---

## Key Rules

- **Never edit Tom's words.** Transcribe verbatim; preserve typos, parentheticals, scare
  quotes, dash style. Pass-note-drafter handles voice/polish downstream.
- **Never create an Opportunity.** If the named company has no Opp entry, stop and tell
  Tom to run `add-to-crm` first.
- **One section per Opp.** Detect existing sections under all four label variants; append
  bullets rather than creating a second section with a different label.
- **Section goes at the TOP** of the page body for new sections. This makes it visible to
  Tom (and to pass-note-drafter on a quick body scan) without scrolling past historical
  call notes.
- **No permission prompts.** Per `feedback_first_pass_no_permission_prompts.md` and
  `feedback_no_permission_for_user_initiated_analysis.md`, Tom-invoked end-to-end skills
  run without asking. The only allowed pause is genuine target ambiguity (Step 1) or
  terminal-status mismatch (Step 2.5).
- **Verify after writing.** Re-fetch and substring-check; never trust the self-report
  alone.
- **No body duplication.** Do NOT add the guidance to any other field (Description, a
  comment, the Diligence Materials property, etc.). Page body only.
- **Don't escape special characters.** Per `feedback_no_escaped_tildes.md`, bare `~$3-4B`,
  bare `$100M`, no leading backslashes. Especially relevant for Tom's voice notes where
  numbers and ranges show up.
- **Use en dashes, not em dashes** in any prose YOU add (Step 6 reports). Tom's transcribed
  content stays verbatim regardless of dash style.

## Disambiguation From Adjacent Skills

- **`pass-note-drafter`** consumes the section this skill writes — it is the downstream
  reader, not a writer. Pass-note-drafter is triggered by Status = `Pass Note Pending`
  (Mode B webhook) or scheduled sweep (Mode A); it never writes to the guidance section.
- **`decision-retro`** captures Tom's retrospective on a decision *after* it's been made
  (post-Committed or post-Pass). Pass note guidance is *before* the pass — pre-decision
  steering. If Tom says "log my retro on X" or uses past tense, route to `decision-retro`
  instead.
- **"Log my thoughts as a note"** (per `feedback_log_thoughts_as_notes_db_entry.md`)
  creates a new Notes DB entry — that's for general diligence thoughts, not pass-note-
  specific steering. If Tom's framing is clearly broader than "how to write the pass note",
  prefer the Notes DB path.
- **`add-conversation-to-notion`** is for archiving the current Claude conversation; not
  relevant here.
