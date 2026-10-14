---
name: dash-lp-quarterly-update
description: End-to-end quarterly Dash LP update. Drafts BOTH the LP email (dash-lp-update-email) AND the two Notion letters (dash-lp-letter-drafter), then ties every number across SOI ↔ email ↔ letters ↔ SOI images before reporting. Human-in-the-loop: Tom duplicates last quarter's two Notion pages (renamed "[WIP] … <Mon YYYY> Update", unpublished). The skill checks for them and pings Tom if they're missing. NEVER sends the email, NEVER publishes the pages. (A) Scheduled: the quarterly Jan/Apr/Jul/Oct 1 job (scheduled-tasks/dash-lp-update-email) runs this skill's Mode A. (C) Manual: "draft the dash LP update", "do the dash quarterly update", "draft the dash email and letters", "dash LP update: email + notion", "dupes are ready" / "made the dupes" (resume after Tom duplicates the pages); "I updated marks" / "refresh the dash numbers" (Refresh mode: re-tie letters + email after SOI mark changes). "Dash LP letters have been finalized" / "dash letters are final" / "finalize the dash update" (Finalize mode: link the published pages into the email and draft the BCC batches). For only the email use dash-lp-update-email; for only the letters use dash-lp-letter-drafter.
---

# Dash LP Quarterly Update (email + Notion letters)

Orchestrator. It owns sequencing, the duplicate handshake and the tie-out. It re-implements nothing. Every step's mechanics live in the two component skills; read each one in full when its phase starts:
- `~/.claude/skills/dash-lp-update-email/SKILL.md`: the email (Gmail DRAFT in tom@dashfund.co, never sent).
- `~/.claude/skills/dash-lp-letter-drafter/SKILL.md`: the two Notion letters (patched in place in the dashfund workspace, never published).

## Step 0: Resolve the period
Summary quarter = the quarter just ended. Send month = the month after it (Jan/Apr/Jul/Oct). Letter titles: `Dash Fund II: <Mon YYYY> Update`, `Dash Fund I: <Mon YYYY> Update`.

## Step 1: Numbers gate (once, shared by both outputs)
1. `dash_soi_marks.py`: `issues[]` must be empty. Any issue → still draft, but ⚠ it in the final alert and say which side is unreliable. SOI TVPI is already ADJUSTED (net cash add-back pulled live from the private sheet "Dash Funds: Unearned Management Fees & Adjusted TVPI"); quote it as-is, labelled `TVPI (Net)` everywhere.
2. **Quarter-end cash is ESTIMATED at draft time, by design** (Tom 2026-10-01: the letters and email go out BEFORE the quarter-end money-market (MMF) statements, which land around the 15th). Never wait or block on them. **Preferred source (Tom 2026-10-01): Tom's screenshot on the quarter's first day of (a) the Citi Deposit Account Summary using the START-OF-DAY columns (= quarter-end close; the "current" columns already include the new quarter's fee transfer) and (b) the money-market fund holdings view.** That's actual quarter-end cash, so no estimate is needed. Ask for it at Step 1 (manual), or include the ask in the duplicate-request text (scheduled). Fallback estimate = the latest MMF statement balance (month-end before quarter-end, from the Citi statements Tom drops in ~/Downloads or forwards), minus/plus SOI subrows dated after that statement (checks out, distributions in), plus the LP checking balances at quarter-end close (Tom's Citi "Deposit Account Summary" snapshot; exclude "MC" management-co accounts). Write it into the private sheet's cash anchors (B14/B15 = amount, E14/E15 = quarter-end date) with the derivation in col C. Ask Tom (manual mode) for the checking snapshot if it isn't in Downloads.
   **True-up (~15th, when Tom sends the MMF statements; Work-list reminder):** replace the anchors with actual quarter-end cash. Re-read SOI TVPI. If any one-decimal figure differs from what was sent, tell Tom and note it in the next quarter's draft notes (the sent letters are not edited after the fact). The SOI updates live either way.
3. Payables: ask Tom (manual mode) whether anything is owed at quarter-end. In scheduled mode, assume none and ⚠ it.
4. Record the canonical figures (one decimal + `x`): per fund MOIC/TVPI/DPI × LRP/DVA, plus Dash II deployed %. These figures are THE numbers for every downstream artifact.

## Step 2: Email
Run dash-lp-update-email Steps 1–6 with the Step 1 figures. Notion links stay placeholders (its rule). Supersede any earlier draft for the same month (create new → verify → delete old with `dash_mail.py draft-delete`).

## Step 3: Duplicate handshake (human in the loop)
Look under "Dash Fund LP Letters" (`6e935211705748a5812519e13bc9375b`, via the Dash API token per dash-lp-letter-drafter Step 4) for two child pages whose titles contain `<Mon YYYY> Update` and `[WIP`.
- **Both exist:** go to Step 4.
- **Missing:** ask Tom and stop the letter phase. The email is already drafted.
  - Manual mode: ask in the conversation.
  - Scheduled mode: text Tom 1:1 (`~/.claude/skills/sms-listener/send_imessage.sh "+12012567714"`, body on stdin):
    ```
    ✍️ Dash Letters: <Mon YYYY>

    ⚠ Duplicate the <prev Mon YYYY> Dash II + Dash I pages, rename to "[WIP] … <Mon YYYY> Update", unpublish both
    Then tell Claude "dupes ready" to finish the letters
    Email draft already saved
    ```
    Shape checked against `send-alert/references/alert-convention.md`: one domain emoji (✍️ draft, consistent every run), Title Case `Headline: Subject`, blank line after the headline, action line first, plain text, 4 body lines, no links.
- **Exist but still published** (Tom sees the banner, or he says so): ask him to unpublish before any edit. Drafts must never be public.

## Step 4: Letters
Run dash-lp-letter-drafter in full on the WIP pages, with the Step 1 figures. That covers performance, the shift-left valuation table from the Valuation Framework tab, closed-deal-only follow-ons/exits/writeoffs from SOI dates, SOI summary images via `soi_summary_png.py`, logo verification contact sheet, new page icons and the Dash I ledger rules.

## Step 5: Tie-out gate (mandatory before reporting)
Run `python3 ~/.claude/skills/dash-lp-quarterly-update/tie_out.py "<Mon YYYY>"`. **Exit 0 is required before any report.** On ✗, fix the artifact that's wrong (never the SOI marks), re-run, and repeat until it's clean. It checks (Tom 2026-10-01 spot checks):
1. **As-of dates:** each letter's Performance line = quarter-end (`MM/DD/YY`); Dash II Pacing "As of <Month> <day>".
2. **Fund figures:** MOIC / TVPI / DPI × LRP / DVA identical across SOI Summary, both letters and the email draft. Dash II deployed / first-check / follow-on % = pacing sheet (60/40 Base).
3. **Valuation table:** right column = SOI `Valuation Framework` tab; left column = prior letter's right column; deltas `-` or `±n`; totals; header quarters.
4. **Labels:** `TVPI (Net)` everywhere.
5. **History untouched:** Dash I Exits and Follow-On entries carried from the prior letter are identical in text, links and logo image (new quarter groups only ADDED at the top). Dash II "New Investments" line is verbatim. Both "Portfolio (Final)" sections, including the image, are unchanged. The only allowed edits are Tom-approved fixes listed in `corrections.json` (e.g. the Caplight Series A link → caplight.com).
7. **Logos + history (`review_checks.py`, run by tie_out.py; Tom 2026-10-02):** every NEW company block's
   logo is compared with the CRM Opp icon (a contact sheet is always written to
   `~/.claude/data/dash_lp_update/<Mon-YYYY>/logo_contact_sheet.png`; rebrands make old entries
   differ by design, so history entries are judged against the prior letter instead). History is
   TERMINAL unless Tom explicitly approves a change, recorded in `corrections.json` (`link_fixes`,
   `logo_fixes`).
8. **Exits booked (`exit_scan.py`; Tom 2026-10-04, Hansa miss):** every Company Updates row / Portfolio Note from quarter start → today with M&A / exit language must have SOI evidence (fund-tab `Exited`, or a Distributions / Secondary / Escrow / `<Acquirer> Stock (M&A)` subrow dated in the quarter). A hit = book it in the SOI first (stock-for-stock pattern in memory `reference_dash_soi_structure`), then the letter's 🚪 Exits block. False positives (acquirer-side, declined offers) → `corrections.json` `exit_scan_dismissed`, keyed by update TITLE with a reason. Also run it at Step 1, before anything is drafted.
6. **Dash II follow-on blocks:** count and $ total = SOI rounds dated in the quarter. Per block: round label, follow-on $, invested-to-date, lead (SOI col Q), MOIC = SOI. CRM card `<Co> (<Round> FO)` (or `(<Round>)` at entry): Inv @ Round, Total Invested, Round Details size and Fund = Dash 2. Letter logo visually matches the CRM icon.

## Refresh mode: Tom changed marks (any time before send)
Trigger: "I updated marks", "refresh the dash numbers", "re-tie the letters", or the same by text. Tom may re-toggle a company's Valuation Framework status in the SOI, which moves DVA values (and the Valuation Framework counts):
1. Re-read the SOI (`dash_soi_marks.py`: issues must be empty).
2. Patch each letter's MOIC / TVPI / DPI bullets and the valuation table's RIGHT column and deltas / totals (left column never changes). Re-render and swap both SOI images.
3. Email: if any figure changed, save a new draft and delete the superseded one.
4. Run `tie_out.py` until clean, then reply with old → new for every figure that moved.

## Step 6: Report (ONE completion)
Manual mode: reply with the email draft link, both WIP page links, the tie-out table (✓/✗) and open flags. Scheduled mode: one Slack alert via send-alert (✍️ shape from dash-lp-update-email Step 7), adding the WIP page links or "⚠ waiting on Tom's duplicates". Tom then reviews, links the Notion pages in the email, publishes the pages (removing "[WIP]") and sends.

## Finalize mode ("Dash LP letters have been finalized")
Tom publishes both pages himself; finalize drops "[WIP]" from both titles. Then run
`python3 ~/.claude/skills/dash-lp-quarterly-update/finalize.py "<Mon YYYY>" --dry-run`, check, and run it
for real. It (1) verifies both "Dash Fund II / I: <Mon YYYY> Update" pages are public, then drops "[WIP] " from their titles,
(2) swaps each `Dash Fund II/I [Tom: link Notion page]` placeholder in Tom's template draft for the
name linked to the public URL and refuses if any `[Tom …]` bracket or `%%` remains, (3) BCCs every
address in the "Dash Fund LPs" database (multi-address fields split on `;`/`,`, deduped) in balanced
batches of ≤ 50, Cc ryan@dashfund.co, one `dash_mail.py draft` per batch (drafts only), (4) texts Tom
ONE summary text + ONE ✍️ Slack draft alert for all batches ("Batches: 1-N"). The template draft stays (deleting
needs Tom's OK). Mirrors lp-letter-workshop's finalize. On send, gmail-webhook `dash-lp-update-sent`
(Dash live handler) posts ONE 📨 "Sent: <Mon YYYY> Dash LP Update" once no batch is left in Drafts
(no tracker); a send stalled >15 min after the last batch, with batches still in Drafts, gets one ⚠ alert.

## Modes
- **A (scheduled, quarter-start morning):** Steps 0–3 always. Steps 4–6 only if the duplicates already exist. Otherwise text the duplicate request and include it in the Slack alert. Never ask questions; ⚠ assumptions instead.
- **C (manual):** all steps. On "dupes ready", resume at Step 3 (the numbers gate re-runs quickly; the email is only redrafted if figures changed).
