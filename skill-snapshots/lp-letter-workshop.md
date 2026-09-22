---
name: lp-letter-workshop
description: |-
  Quarterly LP letter pipeline in three gated phases: (1) Context Pack — assemble everything the letter draws on (prior letters, memos, CRM funnel + pass reasons, diligence dossiers, SOI diff + Company Updates evidence, research intake, people met, LPAC bridge, word-bank vocabulary — full inventory in the skill body) into one reviewable artifact; (2) Foundation — a comprehensive pre-drafting take Tom reacts to; (3) Drafting — a [WIP] Google Doc matching historical letter conventions exactly, iterated turn by turn. Supports mid-quarter starts and post-quarter "incorporate the latest" delta refreshes at every phase. Trigger on "start the [Q3] letter", "LP letter workshop", "let's work on the LP letter", "build the letter context pack", "letter foundation", "draft the Q[N] letter", "refresh the letter pack", "incorporate the latest into the letter". NOT fund-update-drafter (one-off LP email replies) and NOT log-investor-letter-to-notion (external firms' letters). Manual-only.

---

# LP Letter Workshop

Three phases, each gated on Tom. Workspace: `~/.claude/data/lp_letter_workshop/<QUARTER>/`
(quarter format `2026-Q3`). Never advance a phase without Tom's reaction to the prior one;
never send, share, or finalize the letter — Tom does.

## Resolve the quarter

From Tom's ask, else default: the letter covers the most recently *relevant* quarter — before
quarter-end that's the current quarter (quarter-to-date pack, expect a later refresh); after
quarter-end it's the just-closed quarter. Confirm the quarter in the first reply. If a prior
quarter has no letter (e.g. Q2 2026 — LPAC deck only), the pack's LPAC-bridge section carries
it and the foundation proposes how the letter handles the gap.

## Phase 1 — Context Pack

Read `references/context-pack.md` now and execute it in full:
1. `scripts/gather_local.py <QUARTER>` (`--refresh` on re-runs) — local sources: SOI + pins,
   decision ledger + pass notes, retro nuggets, word bank, corpus inventory, LPAC deck text,
   preflight flags.
2. API pulls (parallel subagents): Opportunities funnel, deep-dive dossiers, Company Updates,
   Notes-DB research intake, People/meetings/calendar.
3. Assemble `context-pack.md` (11 fixed sections, every claim cited), send the ONE completion
   alert (shape in the reference), stop for Tom's review + annotations.

## Phase 2 — Foundation

Gate: Tom has reviewed the pack. Read `references/foundation.md` now and execute it in full:
comprehensive, not curated — thinking-evolution ledger, every supportable through-line,
callback inventory, evidence bank, open loops, fund-updates inputs, vocabulary. Write
`foundation.md`, present, stop. Annotate Tom's reactions back into the file as `[TOM]` marks.

## Phase 3 — Drafting

Gate: Tom has reacted to the foundation. Read `references/drafting.md` now and execute it in
full: `[WIP] Inverted Capital I: Q<N> <YYYY> Letter` in the Drive LP Letters folder, formatting
mirrored from the most recent finalized letter, STYLE.md voice, memo-workshop editing harness,
numbers only from the pinned/pack data. Iterate turn by turn.

## Delta refreshes ("incorporate the latest")

Any phase, any time — typically after quarter-close on a mid-quarter start. Re-run Phase 1
with `--refresh` per `references/context-pack.md` → "Refresh / delta runs": prior run is
archived, a dated `## Delta since <as_of>` section leads the pack, and downstream artifacts
(foundation, draft) are updated only where the delta touches them, with Tom told what moved.
