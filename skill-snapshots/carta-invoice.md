---
name: carta-invoice
description: "Job-queue only (never user-triggered). Carta fund-admin invoice landed in tom@dashfund.co → stage it for the local Reminders watcher. Normally finishes in the code pregate with no model session; this body runs only if the pregate failed."
---

# carta-invoice

Enqueued by gmail-webhook `carta-invoice.js` (Dash push) with `args.message_id`. The processor
pregate runs `stage.py <message_id>`, which drops the id in
`~/.claude/scheduled-tasks/office-cleaning-expense/carta_invoice_pending/` and kickstarts
`com.invertedcap.lupe-paid-watch`. Its rider `scripts/carta_invoice_watch.py` (this skill since 2026-10-07; state + pending dir stay in
scheduled-tasks) parses the invoice PDF and
creates `[DF] Pay Carta fund admin invoice (INV…)` (Work list, due = invoice due date, per-fund
split in notes), then texts Tom.

**If you are reading this, the pregate failed.** Do not create the reminder yourself (the job
processor has no Reminders permission). Re-run `python3 ~/.claude/skills/carta-invoice/stage.py
<message_id>` once and report its error. If it still fails, post a 🛠️ alert via the send-alert
skill: `Watcher Failed: Carta Fund Admin Invoice`, with the error and the message id.
