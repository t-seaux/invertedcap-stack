---
name: dash-deal-detect
description: >
  Headless deal-flow detector for Tom's Dash inbox (tom@dashfund.co). The Dash mail lane has no
  Gmail API/Pub/Sub, so `dash-deal-detect.sh` (riding the read-only watch-dash.sh watcher)
  enqueues this job with newly-ledgered candidate messages. For each, it fetches the email from
  the local Apple Mail store (dash_mail.py), applies Tom's high deal-flow bar, dedups against the
  CRM, and for a genuine deal TEXTS Tom a 🆕 Opportunity card — never a direct CRM write. Tom's
  👍 (handled by sms-listener) does the add, stamping Fund=Dash 2️⃣, then posts the Slack new-Opp
  alert. Alert-first, review-gated (Tom, 2026-09-17). Webhook/queue-only — never triggered
  manually. Default to SILENCE; most Dash mail is not a deal.
---

# Dash Deal Detect — 🆕 CRM proposal cards from the Dash inbox

The Dash-email counterpart to [[deal-text-scanner]]'s deal lane. Same downstream contract
(🆕 text card → Tom's 👍 → `sms-listener` writes the CRM row), different source (Dash email via
the local Apple Mail store, not iMessage / not the Gmail API). Fund model:
`/Users/tomseo/.claude/skills/shared-references/fund-context.md`.

**Always runs headless** (`claude --print`, via claude-job-queue). No Gmail MCP, no interactive
Notion. Use `~/.claude/scripts/dash_mail.py` for mail, `ntn api` for the CRM, and
`~/.claude/skills/sms-listener/send_imessage.sh` for the text.

## Input

```json
{ "mode": "scan", "fund": "Dash 2️⃣",
  "messages": [ { "rowid": 202459, "from": "vishal@outmarket.ai",
                  "from_name": "Vishal Sankhla",
                  "subject": "Outmarket - Series B - Dash Fund", "folder": "Inbox" } ] }
```

`dash-deal-detect.sh` already dropped Tom's own sends and obvious non-humans; still treat each
message on its merits. Read the FULL email before judging — `dash_mail.py get <rowid>` returns
`{from,to,cc,subject,date,body}`; `dash_mail.py attachments <rowid> <dir>` extracts any deck for
a local read (`pdftotext <path> - | head -80`). Never send an attachment anywhere; read-only.

## Step 0 — Route each message: FOLLOW-UP vs NEW DEAL

Before the new-deal bar, check whether the message is a **follow-up to an existing deal** — the
Dash counterpart to Inverted's `materials-detect.js`. Tom's rule (2026-09-17): follow-ups must
auto-file on both funds. For each message, resolve the sender to an EXISTING Opp:

```bash
ntn api -X POST /v1/data_sources/fab5ada3-5ea1-44b0-8eb7-3f1120aadda6/query \
  --data '{"filter":{"property":"Contact","rich_text":{"contains":"<sender email>"}},"page_size":5}'
```

- **Hit on a non-portfolio-or-Committed Opp** (Status NOT `Active Portfolio`/`Exited` — Committed
  IS allowed) **AND the email carries a material signal** (an attachment — check
  `dash_mail.py attachments <rowid> <dir>` — a doc link, or ≥400 chars of substantive body):
  this is a **follow-up**, NOT a new deal. Do NOT text a 🆕 card. Instead enqueue a
  **materials-handler Dash-lane** job to auto-file the docs (no 👍 gate — filing to an existing
  card is additive and safe, exactly like Inverted's silent materials-handler Mode B):

  ```bash
  ~/.claude/scripts/enqueue-job.sh materials-handler \
    '{"mail_source":"dash-local","rowid":<rowid>,"oppId":"<opp page id>","oppName":"<company>","fund":"Dash 2️⃣"}' \
    "dash-materials-<rowid>" 900 dash-mail-watch scheduled-sweep
  ```

  **Multi-card routing (Outmarket case):** when the company has several Opp rows (portfolio
  primary + round FO cards), route transaction/diligence docs to the **active round card** — the
  `Committed` (or otherwise non-portfolio) FO card that owns the round's Deal Docs — never the
  `Active Portfolio` primary. Pick the Opp whose Status is Committed/pipeline over any portfolio
  row. Then record the rowid in `~/.claude/skills/dash-deal-detect/.materials-processed` (one
  rowid per line; grep before enqueuing so a re-tick never double-files — belt-and-suspenders on
  top of the queue idempotency key and `notion_files_property.py`'s URL dedup).

  This is exactly the pending **Anshu & Farhan W9** case: when their W9 emails land, the sender is
  on the Outmarket Opp, the Series B FO card is Committed, and this lane files each W9 to that
  card's Deal Docs automatically, with the code-enforced Slack materials alert — no 👍 needed.

- **No existing-Opp hit** → fall through to Step 1 (new-deal classification + 🆕 card).

## Step 1 — Classify (high bar, default SILENT)

A message is a **deal-flow candidate** only with a concrete investable signal, exactly the bar in
`~/.claude/skills/deal-text-scanner/references/deal-lane.md` §1:
- a founder/company + round details (raise, valuation, stage, deck, data room), OR
- an intro offer to a founder, OR a shared deck/LinkedIn accompanying a pitch/referral.

**NOT candidates (stay silent) — and the Dash inbox is full of these:** portfolio-company
investor updates / board decks (that's `investor-update`, not a new deal), fund-admin / LP /
audit / capital-call mail, existing-portfolio ops, transaction docs on a deal already in the CRM
(wire instructions, term sheets — that's `materials-handler`, and the company already has a
card), scheduling, newsletters. When unsure → silent. False negatives are fine (Tom sees his
inbox); a false 🆕 card erodes trust.

Read `deal-lane.md` in full for the classification bar and the card/staging contract — this skill
follows it verbatim; only the source fetch and the staged fund fields below differ.

## Step 2 — Dedup (BOTH gates, in order)

Per `deal-lane.md` §2, but query the CRM via `ntn` (headless):

**Gate A — the CRM itself.** A 🆕 card for a company/founder already in the pipeline is a defect.
Query the Opportunities data source for the PERSON and the company (the Dash inbox skews toward
companies Tom already holds — Outmarket, etc.):

```bash
export NOTION_API_TOKEN=$(cat ~/code/notion-backup/.notion-token.enc.txt | (command -v sops >/dev/null && sops -d /dev/stdin 2>/dev/null || cat))
ntn api -X POST /v1/data_sources/fab5ada3-5ea1-44b0-8eb7-3f1120aadda6/query \
  --data '{"filter":{"or":[{"property":"Contact","rich_text":{"contains":"<founder email>"}},{"property":"Name","title":{"contains":"<company or last name>"}}]},"page_size":5}'
```

ANY hit with a non-terminal Status → do NOT propose. If the mail carries genuinely new signal on
an EXISTING company (a new round kicking off), that's a follow-on, not a new card — hand off to
`add-follow-on-round` / `materials-handler`, don't fire a 🆕 card.

**Gate B — prior proposals.** `~/.claude/skills/dash-deal-detect/.proposed` (one line per prior
proposal: `YYYY-MM-DD <founder/company> via <referrer> rowid=<rowid>`). Grep first; skip if
already proposed. Append a line for each new proposal.

## Step 3 — Propose to Tom (TEXT, 1:1)

Exactly `deal-lane.md` §3 — ONE text via Sendblue to Tom (`+12012567714`), capturing the handle:

```bash
H=$(~/.claude/skills/sms-listener/send_imessage.sh "+12012567714" "<card>") && H=${H#ok }
```

Card shape is the canonical 🆕 Opportunity format (header `🆕 Opportunity: <subject>`, then a
blank line, then `* Source / * Stage / * HQ / * Description` bullets, closing
`👍 to Add to CRM. Respond to make changes.`). Subject/stage/HQ rules per `deal-lane.md` §3.
For a Dash deal the **Source** is usually the sender (the founder or the referrer who emailed
Tom). Resolve the founder's HQ via ContactOut on an in-thread LinkedIn if the mail doesn't state
it. Render in PLAIN TEXT (iMessage) per the alert convention's text lane — no Slack markup.

Then append the audit line binding the confirm — SAME file and shape `sms-listener` already reads
for iMessage deal cards, so no confirm-loop change is needed to FIND the proposal:

```bash
echo "[$(date '+%F %T')] sent_handle=$H notes=proposed add-to-crm <founder> via <referrer> (dash rowid=<rowid>)" \
  >> ~/.claude/skills/sms-listener/audit-log/$(date +%F).log
```

## Step 3b — STAGE the payload (this is what the 👍 consumes)

Write `~/.claude/skills/deal-text-scanner/staged/<sent_handle>.json` (the SAME staged dir
sms-listener loads on confirm) with everything add-to-crm needs, PLUS the Dash-lane fields that
tell the confirm handler to stamp Fund and post the Slack alert:

```json
{
  "opp_title": "…", "stage": "Seed 🌾", "round_details": "…", "hq": "San Francisco",
  "description": "…", "source": "<referrer/founder>", "source_context": "<what the email said>",
  "contact": "<founder email or N/A>", "website": "N/A", "icon": "…",
  "links": ["…"], "deck_drive_link": "<drive url or null>",

  "mail_source": "dash-local",
  "rowid": 202459,
  "fund": "Dash 2️⃣",

  "proposed_at": "<ISO8601>"
}
```

- If the deal email has a **deck attachment**, extract it (`dash_mail.py attachments <rowid> <dir>`)
  and upload to Drive at proposal time (Drive Upload Apps Script, `drive-upload.md`) so
  `deck_drive_link` is ready and the confirm is instant. No deck → `deck_drive_link: null`.
- `mail_source: "dash-local"` + `rowid` let the confirm-time create (or a fallback full
  add-to-crm run) fetch the email via `dash_mail.py` instead of the Gmail API.
- `fund: "Dash 2️⃣"` is the one field the confirm handler must NOT infer — it stamps this Fund on
  the new Opp (see `sms-listener` confirm §4, Dash-lane branch).

## Exit

No candidate is a deal → exit silently. No text, no log noise beyond the watcher's own line.
Never write to the CRM here — proposing is the whole job; the 👍 (sms-listener) does the add and
fires the Slack new-Opp alert.
