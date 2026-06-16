---
name: writing-style
description: >-
  Umbrella router for Tom's writing-style stylebooks — the canonical voice + structural rules live in
  writing-style/<type>/STYLE.md; this entry point picks the right sub-stylebook. Three stylebooks: pass-note
  (founder pass notes; read by pass-note-drafter); outreach (cold founder outreach; read by founder-outreach);
  letters-and-memos (LP letters, investment memos, pre-mortems, first-pass-diligence prose, investor-update
  prose; read by pre-mortem, first-pass-diligence, investor-update, and ad-hoc long-form). Trigger whenever Tom
  asks to "draft", "write", "clean up", "edit", "polish", or "refine" any prose — infer the type from context.
  Also on "log to writing style", "log this letter", "log this memo", "log this checkpoint", "log this to
  letters-and-memos", "save this as a voice example", or any variant to log a draft into the right
  VOICE_EXAMPLES.md.
---

# Writing Style — Umbrella Router

## What this skill is

A thin router. The real voice rules live in three sub-stylebooks:

```
writing-style/
  pass-note/
    STYLE.md            ← canonical voice + scaffold + anti-patterns
    EDIT_PATTERNS.md    ← Canonical Principles + Recent Edits (auto-fed by draft-feedback)
    VOICE_EXAMPLES.md   ← from-scratch sends + voice analyses (auto-fed)
  outreach/
    STYLE.md
    EDIT_PATTERNS.md
    VOICE_EXAMPLES.md
  letters-and-memos/
    STYLE.md            ← canonical voice + structural conventions + anti-patterns
    VOICE_EXAMPLES.md   ← finished letters/memos with voice analyses (manually logged)
    (no EDIT_PATTERNS — workflow is iterative co-writing, not single-shot drafting)
```

Drafters (`pass-note-drafter`, `founder-outreach`, `pre-mortem`, `first-pass-diligence`, `investor-update`) read directly from their corresponding sub-stylebook. This umbrella skill exists for ad-hoc writing requests and the `log to writing style` trigger.

## Routing logic

When invoked, infer the destination sub-stylebook from context:

| If Tom is writing… | Read |
|---|---|
| Pass note to a founder | `writing-style/pass-note/{STYLE,EDIT_PATTERNS,VOICE_EXAMPLES}.md` |
| Cold founder outreach | `writing-style/outreach/{STYLE,EDIT_PATTERNS,VOICE_EXAMPLES}.md` |
| LP letter, investment memo, pre-mortem, first-pass diligence, investor update, or any long-form analytical prose | `writing-style/letters-and-memos/{STYLE,VOICE_EXAMPLES}.md` |

If genuinely ambiguous, ask Tom which type. Don't guess between long-form and short-form — they have very different conventions.

## "Log to writing style" trigger

When Tom says "log to writing style" or any variant during or after a co-writing session:

1. Identify the writing type from the current session context (typically `letters-and-memos` since that's the iterative-workflow stylebook; rarely pass-note or outreach mid-session since those use the draft-feedback pipeline).
2. Run the voice-analysis prompt from `~/.claude/skills/draft-feedback/processor.py` (`analyze_voice_via_claude`, lines 274-313) on the current draft state.
3. Append a dated entry to the corresponding `VOICE_EXAMPLES.md`, using the same format as `append_to_voice_examples` (processor.py:251-271).
4. Confirm to Tom: which file, what got appended.

Mid-session checkpoints are explicitly supported — multiple log calls in one drafting session = multiple timestamped entries.

## Canonical principles

All three stylebooks share a few principles worth naming once:

- **Specific quoted exemplars beat generic descriptors.** "Warm" / "thoughtful" / "candid" are useless without a quoted phrase that demonstrates them.
- **Patterns are observations, not commands.** Drafters apply them as priors, not rigid rules. Tom's voice is contextual — what's right for an LP letter isn't right for a pass note.
- **`EDIT_PATTERNS` is corrective signal; `VOICE_EXAMPLES` is positive ground truth.** They serve different cognitive cues and should not be merged.
- **Em dashes (`—`) only in signatures and as rhetorical pauses in long-form prose. En dashes (`–`) elsewhere.** This is a Tom invariant across all three stylebooks.

## What NOT to do

- Don't hand-edit `EDIT_PATTERNS.md` Recent Edits section — that's auto-fed by the `draft-feedback` processor. The Canonical Principles section above the divider IS hand-curated; promote patterns up from Recent Edits when they're durable.
- Don't write `VOICE_EXAMPLES.md` entries by hand for pass-note or outreach — those are auto-fed by the draft-feedback pipeline. For letters-and-memos, manual logging is the only path.
- Don't merge stylebooks. Each writing type's voice is calibrated to its register and audience. The three are deliberately separate.
