---
name: office-package
description: "Job-queue only (never user-triggered). The office front desk (7 Metrotech / Belltel Lofts) logged a delivery in BuildingLink → text Tom. Normally finishes in the code pregate with no model session; this body runs only if the pregate failed."
---

# office-package

Enqueued by gmail-webhook `office-package.js` when `notify@us1.buildinglink.com` emails
tom@invertedcap.com a front-desk event ("You have a USPS package.", "You have a FedEx delivery at
the front desk", "You have dry cleaning at the front desk", …). Args: `message_id`, `subject`, `body`.
The processor pregate runs `scripts/notify.py`, which parses carrier / time and texts
Tom's 1:1 in text-lane alert convention (`📦 Office Delivery: <Carrier>`, blank line, `Logged: <time> by <first name>`; the desk's bin code / tracking # is never sent – not actionable) (Tom 2026-10-07 asked for a TEXT, overriding the email→Slack default). One text per
email; `scheduled-tasks/office-package/sent.jsonl` dedups re-runs.

Not alerted: "picked up" confirmations and the "unclaimed Incoming Deliveries" reminder digest
(the webhook gate excludes both).

**If you are reading this, the pregate failed.** Re-run `python3 ~/.claude/skills/office-package/scripts/notify.py
--message-id <id> --subject <subject> --body <body>` once with the job's args and report its error.
If it still fails, post a 🛠️ alert via the send-alert skill: `Watcher Failed: Office Package Alert`,
with the error and the message id. Never text Tom a hand-written alert from here.
