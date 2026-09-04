---
name: log-pass-note-guidance
description: >
  Log Tom's free-form Pass Note Guidance — scratch notes or transcribed voice notes that
  give a general read on how he wants the pass note drafted — to the body of an Opportunity
  in the Notion Opportunities DB. The guidance lands as a `⛔` callout directly beneath the
  `📚` Company Overview callout, formatted so `pass-note-drafter` Step 3f picks it up as
  authorial intent when Tom
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
- **Target field:** page body — a `⛔` **callout** block placed directly beneath the `📚`
  Company Overview callout, whose first line is the dated italic label
  `Pass Note Guidance (<date>)` and whose children are the guidance bullets.
  (Pass-note-drafter Step 3f detects the callout shape, plus the older
  heading/bolded-paragraph variants for back-compat.)

## Section Format (canonical — must match what pass-note-drafter recognizes)

The section MUST be written so pass-note-drafter Step 3f detects it. It deliberately
mirrors the `log-company-blurb` 📚 callout — same dated italic label line, same transparent
fill — so the two read as siblings on the page.

**Canonical form — `⛔` callout (use this for every new section):**
```
<callout icon="⛔">
	*Pass Note Guidance (<mention-date start="YYYY-MM-DD"/>)*
	- {bullet 1}
	- {bullet 2}
	- ...
</callout>
```
Children (label + bullets) are indented one tab inside the `<callout>` per Notion-flavored
Markdown.

- **Label is italic only** — `*…*`, never bold. This matches the 📚 blurb exactly.
- **No `color` attribute** — the callout fill stays transparent, same as the blurb. Do not
  set `gray_bg`.
- **Icon** is always the no-entry emoji `⛔`.
- **Date** is a `<mention-date>` for the day the guidance was last updated — see the
  last-updated rule below.

**Date = last updated, not a history (Tom, 2026-09-03).** The parenthetical carries a
single date: when the guidance was last touched. On any material revision or addition, bump
it to today. Never stack multiple dated entries or keep a running log of past guidance
inside the callout — the drafter should read one current intent, not an archive.

**Bullets, one per conceptual point (Tom, 2026-09-03).** Keep the separate ideas separate —
don't fuse his read into a single paragraph. Each bullet is a cleaned-up sentence, not a
transcript fragment.

**Legacy forms (accept when they already exist — append into them, don't convert):** a
`⛔` callout with a plain undated `**Pass Note Guidance**` label, a bolded paragraph
`**Pass Note Guidance**` followed by sibling bullets, or a `## Pass Note Guidance` /
`### Pass Note Guidance` heading. If a legacy section already exists on an Opp, append into
it in place rather than creating a second callout.

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

**Paraphrase rule (Tom's explicit preference, 2026-07-21; reinforced 2026-09-03):** Tom
WANTS your paraphrase — he drops guidance unstructured via voice, so clean it into clear
bullets that capture *his read*. Do not transcribe verbatim: rewrite run-on dictation into
finished sentences, drop filler and transcription artifacts, and fuse the repeated
restatements he makes while thinking out loud into one clear claim. Preserve his **meaning,
emphasis, and any specific pass reason** exactly — do not soften his verdict, invent
substance he didn't imply, or over-editorialize with your own analysis. When in doubt about
whether a point is his read or your inference, keep it to his read. This is the one skill
where paraphrasing is correct: the downstream `pass-note-drafter` treats this as authorial
intent, so it must faithfully represent what Tom thinks, just more legibly than raw voice.

**Structuring:** one bullet per conceptual point, in the order his logic moves — what he
credits, then the turn, then the pass reason. Keep distinct ideas in distinct bullets rather
than fusing them into a paragraph; each bullet should stand on its own as a finished
sentence. Lead with what excited him / the strengths; put the actual pass reason (if he names
one) in its own bullet so the drafter can lift it as the spine. Don't pad to hit a bullet
count — if he gave one clear thought, one bullet is fine.

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

Walk the block children and capture two things:

1. **The `📚` Company Overview callout** (written by `log-company-blurb`) — this is the
   placement anchor. Note its full rendered markdown.
2. **An existing Pass Note Guidance section** (case-insensitive match on the label variants
   above). The detector should match:
   - A `callout` block whose first child is the italic dated label — current shape
   - A `callout` block whose first child is the plain bolded label (older shape)
   - A `paragraph` block whose entire content is bolded text equal to the label (legacy)
   - A `heading_2` or `heading_3` block whose plain text equals the label (legacy)

If a section exists, note which shape it is (dated-italic callout vs. legacy) and the
rendered markdown of its last bullet — that's the append anchor.

### Step 4: Write the Section

**Case A — No existing section (most common, first invocation):**

Insert the `⛔` callout **directly beneath the `📚` Company Overview callout**. Use
`notion-update-page` with `command: update_content`, anchored on the blurb callout's full
rendered markdown:

```
notion-update-page
  command: update_content
  page_id: <opp_page_id>
  content_updates: [{
    old_str: "<📚 blurb callout markdown verbatim>",
    new_str: "<📚 blurb callout markdown verbatim>\n<callout icon=\"⛔\">\n\t*Pass Note Guidance (<mention-date start=\"YYYY-MM-DD\"/>)*\n\t- <bullet 1>\n\t- <bullet 2>\n</callout>"
  }]
```
Children inside the callout are indented one tab. Reproduce the blurb markdown exactly as
`notion-fetch` returned it (including its `<mention-date>` tag and indentation) or the match
will fail.

**No `📚` blurb on the page:** fall back to prepending at the top — `insert_content` with
`position: {"type":"start"}` and the callout markdown as `content`. Same for an empty page.

**Case B — Existing dated-italic callout (extend or revise):**

If Tom is adding a genuinely new point, append bullets inside the existing callout, anchored
on its last bullet. If he's restating or correcting an existing point, rewrite that bullet
in place rather than stacking near-duplicates — the drafter reads the whole section as one
intent, so contradictory leftovers are worse than a clean overwrite.

**Always bump the label's date to today** on any material change, replacing the old date.
The parenthetical is a last-updated stamp, not a history — never append a second dated line
or keep the prior date alongside the new one.

**Case C — Existing legacy section:**

Append new bullets in place — do NOT convert it to the dated-italic shape. Anchor on the
last existing bullet's rendered markdown and match its indentation exactly (a callout's
bullets carry a leading tab; legacy sibling bullets do not).

Idempotency (all cases): before writing, check whether the new content already appears in
the existing section (substring match) to handle re-runs. If it's all already there, report
`⚠️ Guidance already present in [Opp Name]'s Pass Note Guidance — no change made.`

### Step 5: Verify the Write

Re-fetch the Opp page (cheap — same notion-fetch as Step 3, no transcript) and confirm:
- The `⛔` callout exists, with an italic (not bold) dated label and no `color` attribute
- It sits directly beneath the `📚` Company Overview callout (Case A)
- All intended bullets are present (substring match against the rendered markdown)
- Exactly one date appears in the label, and it's today's (on any write that changed content)

Per `feedback_skill_self_report_diverges_from_actual_write.md`, do NOT trust the write
self-report alone — verify by re-reading. If verification fails, retry once. If it still
fails, surface the failure clearly to Tom rather than reporting success.

### Step 6: Report Back

**Case A success:**
```
✅ Pass Note Guidance added to [Opp Name] ([Fund]), under the Company Overview callout:
- <bullet 1>
- <bullet 2>
...
Opp: <Notion URL>
```

**Case B/C success (extended or revised an existing section):**
```
✅ [Extended | Revised] Pass Note Guidance on [Opp Name] ([Fund]) — date bumped to [today]:
- <new or revised bullet 1>
- <new or revised bullet 2>
(Existing bullets preserved: [count])
Opp: <Notion URL>
```

**Duplicate skip:**
```
⚠️ Guidance already present in [Opp Name]'s Pass Note Guidance — no change made.
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
- **One bullet per conceptual point.** Don't fuse his separate ideas into a paragraph, and
  don't split one idea across bullets.
- **Match the blurb's chrome exactly.** Italic-only dated label (never bold), no `color`
  attribute so the fill stays transparent — the ⛔ callout should read as a sibling of the
  📚 Company Overview callout, not as a different artifact.
- **The date is a last-updated stamp, not a history.** One date in the parens, bumped to
  today on every material change. Never accumulate dated guidance entries.
- **Never create an Opportunity.** If the named company has no Opp entry, stop and tell
  Tom to run `add-to-crm` first.
- **One section per Opp.** Detect existing sections under the callout and legacy label
  shapes; extend or revise the existing section rather than creating a second one (never
  convert a legacy section to the dated-italic shape — append in place).
- **Section goes directly beneath the `📚` Company Overview callout** — company context
  first, then Tom's read. Only prepend to the very top when no blurb callout exists.
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
