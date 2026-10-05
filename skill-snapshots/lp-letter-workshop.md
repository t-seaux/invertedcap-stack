---
name: lp-letter-workshop
description: |-
  Quarterly LP letter pipeline in three gated phases: (1) Context Pack — assemble everything the letter draws on (prior letters, memos, CRM funnel + pass reasons, diligence dossiers, SOI diff + Company Updates evidence, research intake, people met, LPAC bridge, word-bank vocabulary — full inventory in the skill body) into one reviewable artifact; (2) Foundation — a comprehensive pre-drafting take Tom reacts to; (3) Drafting — a [WIP] Google Doc matching historical letter conventions exactly, iterated turn by turn. Supports mid-quarter starts and post-quarter "incorporate the latest" delta refreshes at every phase. Trigger on "start the [Q3] letter", "LP letter workshop", "let's work on the LP letter", "build the letter context pack", "letter foundation", "draft the Q[N] letter", "refresh the letter pack", "incorporate the latest into the letter", and the finalize triggers "finalize LP letter", "finalize the letter", "push final version", "letter is final", and the redacted-send triggers "redaction is complete", "redaction done", "redacted version is done", "redacted letter is final". NOT fund-update-drafter (one-off LP email replies) and NOT log-investor-letter-to-notion (external firms' letters). Manual-only.

---

# LP Letter Workshop

Three phases, each gated on Tom. Workspace: `~/.claude/data/lp_letter_workshop/<QUARTER>/`
(quarter format `2026-Q3`). Never advance a phase without Tom's reaction to the prior one;
never send or share the letter — Tom does. Finalize runs only on Tom's explicit trigger (see
**Finalize** below), and even then it only DRAFTS the LP email.

## Resolve the quarter

From Tom's ask, else default: the letter covers the most recently *relevant* quarter — before
quarter-end that's the current quarter (quarter-to-date pack, expect a later refresh); after
quarter-end it's the just-closed quarter. Confirm the quarter in the first reply. If a prior
quarter has no letter (e.g. Q2 2026 — LPAC deck only), the pack's LPAC-bridge section carries
it and the foundation proposes how the letter handles the gap.

## Portfolio guard (gate, Tom 2026-10-04)

`python3 ~/.claude/skills/soi-portfolio-event/portfolio_guard.py --quarter <YYYY-Qn>` compares real-world
developments (Company Updates + Portfolio Notes: exits, winddowns, raises / cash jumps) against the portal SOI,
the quarter pin, and the letter Doc (every company with a round / exit booked in the quarter must be named).
It runs inside `gather_local.py` (the "portfolio guard" line in `preflight.md`) and must be re-run before
Finalize. Exit 0 = clean; 1 = ⚠ flags → STOP and tell Tom (list each flag) before drafting or finalizing;
2 = could not run → tell Tom, never treat as clean. Read-only: it never edits lp-portal, and a flag is fixed
by Tom's normal path (Opp edit → soi-portfolio-event → his confirm). Letters already SENT are never edited
without Tom's explicit permission and guidance. False positives → `soi-portfolio-event/references/guard_dismissed.json`
(keyed by update title, with a reason). The same guard runs daily in the 17:50 digest (🛡️ section).

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

## Finalize ("finalize LP letter", "push final version")

**Step 0: re-run the portfolio guard (`--quarter <YYYY-Qn>`).** Any ⚠ → stop and show Tom before anything is renamed, exported or drafted.

One-off notes (e.g. a capital call heads-up) go on the ACTIVE LP drafts only, never Relationship
LPs or friends. Cap-call heads-up — ONLY when Tom asks (e.g. "give LPs a heads up on a capital call in the LP letter");
this letter email is the sole place LPs hear about a call from Tom – inverted-capital-call never emails LPs.
PRESERVE THIS LANGUAGE verbatim whenever Tom asks for a heads up on an upcoming
capital call (Tom 2026-10-02; fill only <when>, <nth>, <x>, <total>):
"And an early heads up – <when> we plan to issue our <nth> capital call for <x>% of your commitment
amount. This will bring cumulative capital called to <total>%. Vector will reach out with a formal
notice with details soon."
(Q3 2026: when = "later this month", nth = "fourth", x = 15, total = 65. From lp-portal
fund_inputs.json: nth/x = next capital_call_schedule entry; total = capital_called_pct (= sum of
Completed calls) + x. Never guess.)
Generate it, don't hand-fill: `python3 ~/.claude/skills/inverted-capital-call/scripts/cap_call.py --pct <x>
--issue-date "<date>" --when "<when>" --paragraph-only` prints the sentence (prefix "And ") with nth/total
computed and tied to the portal. Same sentence as the standalone LP heads-up in inverted-capital-call.

Explicit trigger only. Run `scripts/finalize_letter.py <QUARTER> --dry-run`, sanity-check the
plan + BCC list (LP (Active) Master Emails from the 🌥️ LP Directory, Tom excluded), then run it
for real. It is resumable (state in `<QUARTER>/finalize.json`) and does, in order: drop `[WIP] `
from the Doc title → PDF to `[PARTNERS]` (no ~/Downloads copy) as `Inverted Capital I_ Q<N> <YYYY>
Letter.pdf` → Doc copy into `[EXTERNAL] Inverted Capital LP Letters` as `[WIP] [EXTERNAL] …`
(redaction flow starts there; the mechanical redaction — CONFIDENTIAL header cleared, Fund Updates → one `[REDACTED]` line, Disclaimers untouched — is applied automatically, `--redact-external` re-runs just that step; essay redactions are worked through live with Tom) → LP Directory: uncheck every box on the sent-letter checkbox,
THEN rename it `Sent Q<N> Letter?` → Gmail DRAFT (BCC-only, PDF attached, signature via
gmail-create-draft.py; never sent) → a second set of drafts to LP (Relationship) with the
FULL letter (Tom 2026-10-02: non-redacted for relationships, going forward): subject `<title> -
friends of the firm`, the "Friends," note with NO archive line (Tom 2026-10-02: "i dont
want them to have access" — no portal, no letters-website link) → ONE summary text to Tom
(LP draft, PDF, EXTERNAL copy ready for redaction, rename, checkbox reset). The existing
✍️ Email Draft alert fires on its own for the draft; don't suppress it.

When Tom sends, gmail-webhook `lp-letter-sent.js` (live push + 30-min reconciler) ticks `Sent
Q<N> Letter?` for every LP (Active) row on the send and confirms in Slack: all active LPs, or
"X of Y" + who's missing (batched sends re-confirm the running total). Then update the
handoff; the portal letter pin stays offer-only per `references/drafting.md` → Finalization.

### Essay redaction (Tom highlights, I black out)

After finalize, Tom sends screenshots of what to hide in the `[WIP] [EXTERNAL]` copy. Use
`scripts/redact_blackout.py <QUARTER> <cmd>` — never a highlight laid over live text (it stays
copyable in the PDF). It replaces the text with a width-matched x/i fill, black on black, so
every line stays where it was in the unredacted letter; each command re-checks line positions
against the pre-blackout baseline and says so if anything moved. Commands: `span "<first words>"
"<last words>"` (passage; its footnotes are blacked too, the superscripts kept), `inline
"<unique context>" "<target>"` (a phrase, body or footnote), `cells "<table caption>" <rows>
<cols>` (also squares multi-line cells into rectangles), `extend` (Tom: blacked passages run to
the end of the line — run after any span), `restore` (undo a phrase), `verify --terms "a|b"`
(layout + leak scan + links + header/comments/suggestions — run before "redaction is
complete"). Tom also edits text directly (e.g. deleting "PF"); that's fine. Do what he
highlights; flag what can be back-solved but don't push (Tom 2026-10-02: "i dont mind people
doing the division").

### Redacted send ("redaction is complete", "redacted version is done")

Explicit trigger only, after the LP finalize. `scripts/finalize_letter.py <QUARTER> --redacted
--dry-run`, check, then run: drop `[WIP] ` from the `[EXTERNAL]` Doc → PDF into `[EXTERNAL]` as
`Inverted Capital I_ Q<N> <YYYY> Letter_Redacted.pdf` → Gmail DRAFT `<title> - friends of the
firm` (same "Friends," note, no archive line) BCC Tom's "Friends (Redacted)" list (Sheet
`1DxQaO3z…`, Worksheet tab, column B under that column-A label; re-read every run; the
"Close Friends (Full)" block below it gets the FULL letter at finalize step 5b instead) → ONE
summary text. LP (Relationship) is NOT on this send (they got the full letter at finalize). The
tracker is never reset here.

Send confirmations (gmail-webhook `lp-letter-sent.js`, Slack 📨): "Sent: Q<N> LP Letter – Active
LPs" / "… LP Letter – Relationship LPs" (box-ticks) / "Sent: Q<N> Redacted Letter – Friends"
(count only). The two friends-of-the-firm sends share a subject; recipients tell them apart. ONE alert per
email group, not per batch: each batch send only updates state; the alert fires on the send that
leaves no batch of that group in Drafts (script lock serializes rapid-fire sends). Stalled >15 min after the last batch with
batches still in Drafts → one ⚠ alert (reapStalledLetterSends: one-off trigger per batch, 30-min reconciler backup).
Gmail caps drafts at 50 recipients, so every list is split into balanced batches.
Reminder: once every LP (Active) and LP (Relationship) address is ticked, `scripts/reminder_watch.py`
(rider on office-cleaning-expense/lupe_watch.sh, which holds the Reminders grant) completes "[IC]
Send LP letters" (Work list, every 3 months on the 1st) and texts Tom. Publishing to
invertedcap.com/letters is not part of these flows.

## Delta refreshes ("incorporate the latest")

Any phase, any time — typically after quarter-close on a mid-quarter start. Re-run Phase 1
with `--refresh` per `references/context-pack.md` → "Refresh / delta runs": prior run is
archived, a dated `## Delta since <as_of>` section leads the pack, and downstream artifacts
(foundation, draft) are updated only where the delta touches them, with Tom told what moved.
