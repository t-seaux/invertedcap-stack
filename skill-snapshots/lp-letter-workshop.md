---
name: lp-letter-workshop
description: |-
  Finalize + distribute the quarterly Inverted LP letter (Tom drafts the letter himself; the Context Pack / Foundation / Drafting phases were retired 2026-10-06). Finalize: rename the [WIP] Doc, PDF to [PARTNERS], [EXTERNAL] copy with mechanical redaction, LP Directory checkbox reset, BCC'd Gmail DRAFT to Active LPs (optional capital-call heads-up). Essay redaction: black out the passages Tom highlights in the [EXTERNAL] copy. Redacted send: drafts the ONE redacted version to every friends-of-the-firm list. Trigger on the finalize triggers "finalize LP letter", "finalize the letter", "push final version", "letter is final", and the redacted-send triggers "redaction is complete", "redaction done", "redacted version is done", "redacted letter is final". NOT fund-update-drafter (one-off LP email replies) and NOT log-investor-letter-to-notion (external firms' letters). Manual-only.

---

# LP Letter Workshop

Finalize and distribution only — Tom writes the letter himself in the `[WIP] Inverted Capital I:
Q<N> <YYYY> Letter` Doc (the context-pack / foundation / drafting phases were retired 2026-10-06).
Workspace: `~/.claude/data/lp_letter_workshop/<QUARTER>/` (quarter format `2026-Q3`). Never send
or share the letter — Tom does. Every step runs only on Tom's explicit trigger and only DRAFTS emails.

Quarter: from Tom's ask, else the just-closed quarter. Confirm it in the first reply.

## Portfolio guard (gate, Tom 2026-10-04)

`python3 ~/.claude/skills/soi-portfolio-event/portfolio_guard.py --quarter <YYYY-Qn>` compares real-world
developments (Company Updates + Portfolio Notes: exits, winddowns, raises / cash jumps) against the portal SOI,
the quarter pin, and the letter Doc (every company with a round / exit booked in the quarter must be named).
Run it before Finalize. Exit 0 = clean; 1 = ⚠ flags → STOP and tell Tom (list each flag) before finalizing;
2 = could not run → tell Tom, never treat as clean. Read-only: it never edits lp-portal, and a flag is fixed
by Tom's normal path (Opp edit → soi-portfolio-event → his confirm). Letters already SENT are never edited
without Tom's explicit permission and guidance. False positives → `soi-portfolio-event/references/guard_dismissed.json`
(keyed by update title, with a reason). The same guard runs daily in the 17:50 digest (🛡️ section).

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
(the redacted version starts there; the mechanical redaction — CONFIDENTIAL header cleared, Fund
Updates and every section under it → one `[REDACTED]` line, Disclaimers untouched — is applied
automatically, `--redact-external` re-runs just that step; Tom then highlights essay passages to
black out) → LP Directory: uncheck every box on the sent-letter checkbox,
THEN rename it `Sent Q<N> Letter?` → Gmail DRAFT (BCC-only, PDF attached, signature via
gmail-create-draft.py; never sent) → ONE summary text to Tom (LP draft, PDF, EXTERNAL copy ready
for highlights, rename, checkbox reset). No friends-of-the-firm drafts at finalize — they wait
for the redacted send. ⛔ Friends of the firm NEVER see Fund Updates or any section under it
except Disclaimers (Tom 2026-10-05): the full `[PARTNERS]` PDF is Active-LP-only, code-enforced
in `make_draft` (tests/test_friends_guard.py). The existing
✍️ Email Draft alert fires on its own for the draft; don't suppress it.

When Tom sends, gmail-webhook `lp-letter-sent.js` (live push + 30-min reconciler) ticks `Sent
Q<N> Letter?` for every LP (Active) row on the send and confirms in Slack: all active LPs, or
"X of Y" + who's missing (batched sends re-confirm the running total). Then update the
handoff; the portal letter pin stays offer-only per `references/finalize.md` → Finalization.

### Essay redaction (Tom highlights, I black out)

After finalize, Tom sends screenshots of what to hide (he may highlight in the PARTNER Doc —
apply to the `[EXTERNAL]` copy). Use `scripts/redact_blackout.py <QUARTER> <cmd>` — never a
highlight laid over live text (it stays copyable in the PDF). It replaces the text with a
width-matched x/i fill, black on black, so every line stays where it was; each command re-checks
line positions against the pre-blackout baseline. Commands: `span "<first words>" "<last words>"`
(passage; its footnotes are blacked too), `inline "<unique context>" "<target>"` (a phrase, body or
footnote), `cells "<table caption>" <rows> <cols> [<lines>]` (rows 0-based with row 0 = header —
COUNT the rows, tables differ; optional `<lines>` blacks only those lines inside each cell, e.g.
the round label under a post-money), `extend` (run after any span), `refs` (fill footnote markers left inside blacked text; verify flags them), `restore` / `uncells` (undo), `verify --terms
"a|b"` (layout + leak scan — run before "redaction is complete"; --terms is REQUIRED: every company name/figure blacked this quarter, no built-in list). `span`'s end phrase must be unique after the start (it stops and names the count otherwise) and the resolved span is printed — read it before applying. When Tom says "all other
references to those fields", sweep every table, prose sentence and footnote carrying the same
figure/label (verify --terms with each value). Q3 2026 precedent (archived first pass in
`[EXTERNAL]/Archive`; final, Tom 2026-10-05): ONLY the portfolio traction passages + their
footnotes (markers filled). KEPT after review: all Dash/Inverted returner figures — entry price
+ ownership (tables + footnote 4), post-money, round labels (Series F (PF), Series B), "PF
post-money". Tom's "all other references" sweep was then walked back field by field. Undo with `uncells` (cells, from the
partner Doc) / `restore` (body or footnote).

### Redacted send ("redaction is complete", "redacted version is done")

Explicit trigger only, after the LP finalize + Tom's highlights. `scripts/finalize_letter.py
<QUARTER> --redacted --dry-run`, check, then run: drop `[WIP] ` from the `[EXTERNAL]` Doc → PDF
into `[EXTERNAL]` as `Inverted Capital I_ Q<N> <YYYY> Letter_Redacted.pdf` → Gmail DRAFT(s)
`<title> - friends of the firm` ("Friends," note, NO archive line — Tom 2026-10-02: "i dont want
them to have access") BCC EVERY friends list: LP (Relationship) + the sheet's "Close Friends
(Full)" and "Friends (Redacted)" blocks (Sheet `1DxQaO3z…`, Worksheet tab, column B under each
column-A label; re-read every run), Active LPs excluded → ONE summary text. ONE redacted version
for everyone (Tom 2026-10-05: "i dont want to create confusion"). One-off adds later →
`--close-friends` or a draft attaching `finalize_redacted.json` pdf_id. The tracker is never
reset here.

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
