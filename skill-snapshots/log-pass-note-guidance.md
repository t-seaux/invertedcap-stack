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
- **Target field:** page body — a `⛔` **callout** block at the top of the body, whose
  first line is the bolded label `Pass Note Guidance` and whose children are the guidance
  bullets. (Pass-note-drafter Step 3f detects the callout shape, plus the older
  heading/bolded-paragraph variants for back-compat.)

## Section Format (canonical — must match what pass-note-drafter recognizes)

The section MUST be written so pass-note-drafter Step 3f detects it.

**Canonical form — `⛔` callout (use this for every new section):**
```
<callout icon="⛔" color="gray_bg">
	**Pass Note Guidance**
	- {bullet 1}
	- {bullet 2}
	- ...
</callout>
```
Children (label + bullets) are indented one tab inside the `<callout>` per Notion-flavored
Markdown. Icon is always the no-entry emoji `⛔`; color is always `gray_bg` (matches the
Notion UI's default gray callout — without an explicit color the API renders it
transparent, which looks off).

**Legacy forms (accept when they already exist — append into them, don't convert):** a
bolded paragraph `**Pass Note Guidance**` followed by sibling bullets, or a
`## Pass Note Guidance` / `### Pass Note Guidance` heading. If a legacy section already
exists on an Opp, append into it in place rather than creating a second callout.

Casing/punctuation variants Tom uses interchangeably: `pass note guidance`,
`Pass-Note Guidance`, `Pass note guidance`. Treat all as the same section — do not create
a second section with a slightly different label.

---

## Execution Workflow

### Step 1: Parse Tom's Input

Extract two things from Tom's message:

1. **Target Opportunity name** — the company / Opp Tom is referencing. May be shorthand
   (e.g., "the Lex one", "Acme") or explicit (e.g., "the Acme Seed Opp").
2. **Guidance content** — the actual scratch/voice thoughts. May arrive as:
   - Inline bulleted text in the message
   - A pasted screenshot of bullets (read the image)
   - A voice-style run-on paragraph (Tom's default — he dictates unstructured)

**Paraphrase rule (Tom's explicit preference, 2026-07-21):** Tom WANTS your paraphrase —
he drops guidance unstructured via voice, so clean it into clear, structured bullets that
capture *his read*. Rewrite run-on dictation into self-contained thoughts; fix transcription
artifacts and filler; organize into a tight bullet per idea. Preserve his **meaning,
emphasis, and any specific pass reason** exactly — do not soften his verdict, invent
substance he didn't imply, or over-editorialize with your own analysis. When in doubt about
whether a point is his read or your inference, keep it to his read. This is the one skill
where paraphrasing is correct: the downstream `pass-note-drafter` treats this as authorial
intent, so it must faithfully represent what Tom thinks, just more legibly than raw voice.

**Structuring:** split into natural thought boundaries, one self-contained idea per bullet.
Lead with what excited him / the strengths; put the actual pass reason (if he names one) in
its own bullet so the drafter can lift it as the spine. Don't pad to hit a bullet count —
if he gave one clear thought, one bullet is fine.

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
(case-insensitive match on the label variants above). The detector should match:

- A `callout` block whose first child is the bolded label (canonical current shape)
- A `paragraph` block whose entire content is bolded text equal to the label (legacy)
- A `heading_2` or `heading_3` block whose plain text equals the label (legacy)

Capture: (a) whether a section exists, (b) if so, which shape (callout vs. legacy), and
the rendered markdown of its last bullet (the append anchor).

### Step 4: Write the Section

**Case A — No existing section (most common, first invocation):**

Prepend a new `⛔` callout to the TOP of the page body. Notion's REST API has no native
prepend, so use MCP `notion-update-page` with `command: update_content`, anchored on the
first child block's rendered markdown:

```
notion-update-page
  command: update_content
  page_id: <opp_page_id>
  content_updates: [{
    old_str: "<first child markdown verbatim>",
    new_str: "<callout icon=\"⛔\" color=\"gray_bg\">\n\t**Pass Note Guidance**\n\t- <bullet 1>\n\t- <bullet 2>\n</callout>\n<first child markdown verbatim>"
  }]
```
Children inside the callout are indented one tab. Fetch the first child via `notion-fetch`;
the first non-empty block's markdown is the anchor. If that markdown is short/common enough
to risk a non-unique match (a bare `---`, a one-word heading), extend the anchor to include
the next block so the substitution is unambiguous.

**Simplest alternative (equally valid):** `command: insert_content` with
`position: {"type":"start"}` and the callout markdown as `content` — this prepends without
needing an anchor. Prefer it when the first-block anchor is awkward.

**Page is empty (no children):** `insert_content` with `position: {"type":"start"}` (append
=== prepend on an empty page) and the callout markdown as `content`.

**Case B — Existing section found (append bullets):**

Append the new bullets into the existing section. Anchor on the last existing bullet's
rendered markdown (works for both callout and legacy shapes — the new bullets inherit the
same indentation/nesting):
```
notion-update-page
  command: update_content
  page_id: <opp_page_id>
  content_updates: [{
    old_str: "\t- <last existing bullet verbatim>"   (callout: leading tab; legacy: no tab)
    new_str: "\t- <last existing bullet verbatim>\n\t- <new bullet 1>\n\t- <new bullet 2>"
  }]
```
Match the existing indentation exactly — a callout's bullets carry a leading tab, legacy
sibling bullets do not. Do NOT convert a legacy section to a callout; append in place.

Idempotency: before writing, check whether each new bullet's text already appears in the
existing section (substring match). Skip duplicates to handle re-runs. If ALL new bullets
are duplicates, report
`⚠️ All bullets already present in [Opp Name]'s Pass Note Guidance — no change made.`

**Case C — Existing section found but empty (label only, no bullets):**

Anchor on the label line and append the bullets after it, matching the shape's indentation
(callout: tab-indented bullets under the label; legacy: sibling bullets).

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

- **Paraphrase, don't transcribe.** Clean Tom's unstructured voice/scratch into tight,
  legible bullets that capture his read (his explicit preference, 2026-07-21). Preserve his
  meaning, emphasis, and pass reason exactly; don't soften his verdict or add your own
  analysis. Keep it to his read when unsure whether a point is his or your inference.
- **Never create an Opportunity.** If the named company has no Opp entry, stop and tell
  Tom to run `add-to-crm` first.
- **One section per Opp.** Detect existing sections under the callout and legacy label
  shapes; append into the existing section rather than creating a second one (never convert
  a legacy section to a callout — append in place).
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
- **Use en dashes, not em dashes** throughout — both the guidance bullets (now your
  paraphrase) and the Step 6 report. Per `feedback_writing_mechanics`.

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
