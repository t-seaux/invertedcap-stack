---
name: vector-fee-check
description: "Job-queue only (never user-triggered). Vector AIS emailed tom@invertedcap.com a Management Fee Calculation workbook (quarterly, or + Catch-up after a close) → check it against the LPA fee formula and the LP portal, post ONE #claude-alerts note. Normally finishes in the code pregate (fast.py); this body runs only if the pregate failed."
---

# vector-fee-check

Enqueued by gmail-webhook `vector-fee.js` when a `vectorais.com` sender emails tom@invertedcap.com an
`.xlsx` whose name contains "Management Fee" (e.g. `Inverted Capital I, LP Q4 '26 Management Fee
Calculation.xlsx`, `… Q3 '26 Management Fee Calculation + Catch-up.xlsx`). Args: `message_id`.

Vector sends one every quarter for Tom's review, plus a catch-up workbook after each subsequent close
(Tom 2026-10-14; only one correction ever, early 2025). `fast.py` (the processor pregate) pulls the
message, runs `scripts/check.py` per workbook and posts `📑 Vector Fee Workbook: <Quarter>`:
- **✓ ties**: fee base = LP portal; each quarter = base × 0.625% (×69/91 first quarter, 0.375% after
  the step-down); inception-to-date = LPA formula (`~/code/lp-portal/nav_rollforward.fee_catch_up`);
  GP cashless credits = 70% × GP commit × % called as of the email; cash to transfer foots.
- **⚠ N issues to raise with Vector before approving**: the lines that don't tie, with the expected figure.
- **New close**: fee base above the portal → the fee catch-up on the added commitment, printed as the
  `refresh_inputs.py late-close --fee-catch-up <x>` figure (see memory feedback_lp_late_close_upsize_workflow).

Read-only: never writes the LP portal, never replies to Vector. Dedup: `runs/seen.jsonl` (message id +
filename). Tom approves the transfer with Vector himself.

**If you are reading this, the pregate failed.** Re-run `python3 ~/.claude/skills/vector-fee-check/fast.py
'<job json>'` once and report its error. If it still fails, post a 🛠️ alert via send-alert:
`Watcher Failed: Vector Fee Check`, with the error and the message id. Never hand-compute the check.
