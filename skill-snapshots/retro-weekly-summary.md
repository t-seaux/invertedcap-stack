---
name: retro-weekly-summary
description: >-
  Weekly roll-up of Tom's decision retros: selects the past 7 days of completed retros from the decision-retro
  queue, re-fetches each Opp's current Status, synthesizes emerging themes, writes each crystallized theme into
  founder-taste/CASEBOOK.md (never RUBRIC.md), flags rubric-gap themes as Suggested Refinements, and posts one
  summary to #decision-retros. Runs Fridays 4pm ET (scheduled wrapper = transport only); a session runs the same
  steps on "weekly retro summary", "roll up this week's retros", "retro roll-up", "what came out of this week's
  retros", "run the retro summary".
---

# Retro Weekly Summary

The ONE spec for the weekly retro roll-up. The scheduled wrapper (`~/.claude/scheduled-tasks/retro-weekly-summary/`)
only sets the schedule, the unattended guard and `DRY_RUN`; a session runs these same steps and pastes the same
rendered summary. Promoted from a wrapper-only procedure 2026-10-11 (drift audit: sessions could not run it, the
Casebook helper's default path did not exist, entries were written as `##` sections, and rubric names had drifted).

**Model tier:** `sonnet` (synthesis against a fixed rubric; the scheduled run passes it explicitly).

Reference specs (read for mapping context — never echo their contents in the summary):
- `~/.claude/skills/decision-retro/SKILL.md` — how retros are captured (queue + `## Retro (YYYY-MM-DD)` blocks)
- `~/.claude/skills/founder-taste/CASEBOOK.md` — the ONLY write target
- `~/.claude/skills/founder-taste/RUBRIC.md` — §5 Six Signals, read-only here
- `~/.claude/skills/founder-taste/DECISION_RETROS.md` — thematic master log of nuggets

## Writes

- **CASEBOOK.md only, via `scripts/append_casebook_entry.py`** — the one write path (scheduled and session). It inserts
  the entry as `### <Theme>` at the end of the target `##` subject section, demotes a `#`/`##` entry title, refuses a
  duplicate title (exit 3 → drop that theme, note it), and refuses to touch RUBRIC.md.
- **RUBRIC.md never.** Framework refinements are Tom's manual workflow; rubric-gap themes are only *flagged*
  (Suggested Refinements). Recipe edits (neg1-sourcing A–D) are human-gated the same way.
- `DRY_RUN=1` → no Casebook write, no post: `append_casebook_entry.py --dry-run` + the rendered summary go to the
  wrapper's `logs/<date>-preview.md` (session: shown in chat).

## Steps

1. **Load the queue** `~/.claude/skills/decision-retro/queue.json`. Missing/empty → nothing to summarize (scheduled:
   one audit-log line, exit 0).
2. **Select** `items[]` with `status == "completed"` and `completed_at` within the last 7 days (midnight-to-midnight
   ET, ending today). `skipped` / `no_retro` / `prompted` / `post_failed` are hard-excluded — a "skip" reply never
   appears as text, theme or mapping.
3. **Re-fetch current Status** (the queue's `decision` is frozen at prompt time): `scope == "opp"` → `notion-fetch`
   the `opp_url`, read `Status`; `scope == "neg1"` → the historic -1 entry's recorded status (the -1 Scanner DB is
   deleted — use the queue's value). Fetch failure → fall back to the queue `decision` and log a WARN. The renderer
   refuses legacy bare `Pass` / `Passed`.
4. **Gather** each item's most recent `## Retro (YYYY-MM-DD)` block from the page (reuse the Step 3 fetch). Discover
   the Casebook's subject sections = its `##` headers (Founder / Market / Business Model today; adapt to whatever
   exists).
5. **Synthesize themes** — 0–N patterns crystallized enough for the Casebook: specific enough to apply, ≥1 retro this
   week, not already in the Casebook. Each: name (one line), subject (a discovered section), pattern (1–2 sentences),
   evidence (retros + one-line context), optional anchor = one of RUBRIC.md §5's six signals:
   **Non-Linearity, Earned Reps, Intellectual Honesty + Rigor, Anticipation, Intentionality, Range** (the renderer
   rejects anything else).
6. **Suggested Refinements** — themes that map to none of the six signals (still written to the Casebook). Plus at most
   one **Sourcing implications** sentence when the week's pass/draft reasons cluster on a sourcing-shaped attribute
   (count + shared attribute + the recipe tweak); query
   `SELECT label, decision, why FROM decisions WHERE date >= date('now','-7 day') AND why IS NOT NULL` on
   `~/.claude/data/decision_ledger.db`.
7. **Write each theme** to a temp file and run
   `python3 ~/.claude/skills/retro-weekly-summary/scripts/append_casebook_entry.py --section "<subject>" --entry-file <f>`.
   Entry shape (matches the Casebook):
   ```markdown
   ### <Theme name>

   **Surfaced**: weekly retro YYYY-MM-DD.

   **Pattern**: <1–2 sentences>

   **Evidence**:
   - <Company/Person> (<current Status>): <one-line context>

   **Anchors**: <Signal> (<signal / anti-signal>) – <why it is a strong example>
   ```
8. **Render** — the ONE renderer: write the payload and run
   `python3 ~/.claude/skills/retro-weekly-summary/scripts/render_summary.py --input <payload.json>` (payload schema in
   the script docstring). Never hand-format. Brevity is enforced by what you put in: every bullet is a headline —
   theme pattern ≤ 2 sentences, refinements and sourcing ONE sentence; detail belongs in the Casebook entry.
9. **Post** (scheduled, or a session asked to post): body file → `WEBHOOK_URL="$(cat ~/.claude/skills/decision-retro/.webhook_url)" BODY_FILE=<f> python3 ~/.claude/skills/send-alert/md_to_blocks.py` — top-level message in `#decision-retros`. A session also pastes the same rendered text in chat.
10. **Audit** (scheduled): one block in the wrapper's `audit-log/YYYY-MM-DD.log` — retros summarized by status,
    themes + target sections, refinements flagged, post response code, any duplicate-theme refusals.

## Cadence

`decision-retro-scan` (daily 9am) prompts, `decision-retro-listener` (daily 6pm) processes replies, this runs Friday
4pm — a reply Tom sends Friday morning lands in next week's summary (acceptable).
