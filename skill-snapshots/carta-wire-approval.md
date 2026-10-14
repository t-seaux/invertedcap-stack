---
name: carta-wire-approval
description: "Job-queue only (never user-triggered). Carta's 'Approve $X wire transfer by <date>' / 'payment is ready for release' / 'Distribution Update: $X received' email (Dash distribution) landed in tom@dashfund.co → stage it for the local Reminders watcher. Normally finishes in the code pregate with no model session; this body runs only if the pregate failed."
---

# carta-wire-approval

Enqueued by gmail-webhook `carta-wire.js` (Dash push) with `args.message_id`. The processor
pregate runs `stage.py <message_id>`, which drops the id in
`~/.claude/scheduled-tasks/office-cleaning-expense/carta_wire_pending/` and kickstarts
`com.invertedcap.lupe-paid-watch`. Its rider `scripts/carta_wire_watch.py` (this skill; state + pending dir stay in
scheduled-tasks) parses the email and keeps TWO Work reminders per distribution (Tom 2026-10-07), created together
with one text:
- `[DF] Approve Dash AMM wires in Citi (…)` – the bulk Automated Money Movement wires to Carta's pass-through
  account. Each `investorservices@carta.com` "Distribution Update: $X received" email ticks its fund + amount line;
  when every one is in, the watcher checks the reminder off and texts Tom ✅. Unmatched receipts are skipped silently.
- `[DF] Release Carta direct LP wires in Citi (…)` – wires to LPs not on AMM. No receipt email exists; Tom checks it off.

**If you are reading this, the pregate failed.** Do not create the reminder yourself (the job
processor has no Reminders permission). Re-run `python3 ~/.claude/skills/carta-wire-approval/stage.py
<message_id>` once and report its error. If it still fails, post a 🛠️ alert via the send-alert
skill: `Watcher Failed: Carta Wire Approval`, with the error and the message id.
