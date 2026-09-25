---
name: dash-deal-detect
description: >
  Headless inbound-mail router for Tom's Dash inbox (tom@dashfund.co) — the Dash counterpart to
  Inverted's Gmail-webhook stack, which Dash lacks (no Gmail API/Pub/Sub). `dash-deal-detect.sh`
  rides the read-only watch-dash.sh watcher and enqueues this job with newly-ledgered candidate
  messages; it fetches each from the local Apple Mail store (dash_mail.py) and routes by CRM
  match + Status into three lanes: (1) NEW DEAL (no CRM hit + deal signal) → TEXTS Tom a 🆕
  Opportunity card, his 👍 adds it (sms-listener, Fund=Inverted 1️⃣ — new deals are never Dash, then Slack), his 👎 creates it as
  Pass (DNM); (2) FOLLOW-UP docs on a pipeline/Committed Opp → materials-handler Dash-lane
  (silent auto-file to Deal Docs/Diligence); (3) PORTFOLIO UPDATE / board material on an
  Active-Portfolio Opp → investor-update Dash-lane (Company Updates DB). Alert-first for new deals,
  silent for follow-ups/updates — mirrors Inverted exactly. Webhook/queue-only — never triggered
  manually. Default to SILENCE; most Dash mail is none of these.
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
                  "subject": "Outmarket - Series B - Dash Fund", "folder": "Inbox",
                  "existing_opps": [ { "id": "3be00bef-…", "name": "Outmarket (Series B FO)", "status": "Committed" },
                                     { "id": "128bec53-…", "name": "Outmarket", "status": "Active Portfolio" } ] } ] }
```

Each message also carries **`existing_opps`** — the CRM rows whose `Contact` contains the sender
email, resolved IN CODE by the rider as `[{id, name, status}]` (`[]` = no match; `null` = the
lookup failed, fall back to your own Gate A query). **Non-empty `existing_opps` = the company is
already Tom's. It can NEVER become a 🆕 card** — route it through Step 0's follow-up / update
lanes by the matched Status (Outmarket, 2026-09-22: Tom — "outmarket is an existing portfolio
company so shouldn't trigger add to crm / new opp"). Multi-row matches (portfolio primary + FO
cards) follow the multi-card routing rule below.

`dash-deal-detect.sh` already dropped Tom's own sends and obvious non-humans; still treat each
message on its merits. Read the FULL email before judging — `dash_mail.py get <rowid>` returns
`{from,to,cc,subject,date,body}`; `dash_mail.py attachments <rowid> <dir>` extracts any deck for
a local read (`pdftotext <path> - | head -80`). Never send an attachment anywhere; read-only.

## Step 0 — Route each message: FOLLOW-UP vs NEW DEAL

> **Code gate upstream (2026-09-22) — cold follow-ups never reach this skill.**
> `dash-deal-detect.sh` drops a cold pitcher's repeat emails in code, per
> `~/.claude/skills/shared-references/cold-followup-gate.md` (§ "The rule"; Dash column of
> § "Lane mechanics"). Dropped rows are logged as `cold-followup-skip` in `watch-dash.log` and
> ledgered in `.cold-followups`. Two carve-outs
> (who passes, how rebrands match) are defined in `~/.claude/skills/shared-references/revive-gate.md` § "Who reaches this gate" — a
> `Pass (Met)` founder always arrives, and an alias hit arrives with `matched_via`. Otherwise a
> repeat sender that DOES arrive here failed at least one of the gate's drop conditions.

Before the new-deal bar, check whether the message is a **follow-up to an existing deal** — the
Dash counterpart to Inverted's `materials-detect.js`. Tom's rule (2026-09-17): follow-ups must
auto-file on both funds. For each message, resolve the sender to an EXISTING Opp. **Blurb capture:** when the sender resolves to an existing Opp and the message carries a founder / intro-er blurb, also run `~/.claude/skills/shared-references/blurb-capture.md` (`lane: dash`). This applies on every route, silent ones included.

```bash
ntn api -X POST /v1/data_sources/fab5ada3-5ea1-44b0-8eb7-3f1120aadda6/query \
  --data '{"filter":{"property":"Contact","rich_text":{"contains":"<sender email>"}},"page_size":5}'
```

The CRM hit's **Status routes the message** (same split as Inverted: portfolio → investor-update;
pipeline/Committed → materials-handler; no hit → new deal):

- **Hit on a PORTFOLIO Opp** (Portfolio set, `shared-references/opp-status-sets.md`) **AND
  the email is a portfolio update or board material** (a founder/CEO investor update, monthly/
  quarterly update, board deck/meeting, a Google Slides/Docs share of a board deck): this is a
  **portfolio update**, NOT a deal and NOT a Deal-Docs drop. Do NOT text a 🆕 card. Enqueue an
  **investor-update Dash-lane** job to log it to the Company Updates DB (no 👍 gate — same silent
  auto-log as Inverted's investor-update webhook):

  ```bash
  ~/.claude/scripts/enqueue-job.sh investor-update \
    '{"mail_source":"dash-local","rowid":<rowid>,"oppId":"<opp page id>","oppName":"<company>","fund":"Dash 2️⃣"}' \
    "dash-investor-update-<rowid>" 900 dash-mail-watch scheduled-sweep
  ```

  Record the rowid in `~/.claude/skills/dash-deal-detect/.updates-processed` (grep before
  enqueuing). investor-update's own artifact-idempotency (period-row check) is the real guard;
  this ledger just avoids re-enqueuing across ticks.

- **Hit on a TERMINAL Opp** (Terminal set, `shared-references/opp-status-sets.md`) → skip the
  materials lane below and go straight to Gate A's Terminal branch — the **Revive Gate**
  (`~/.claude/skills/shared-references/revive-gate.md`). It files the materials AND the update
  email itself, and sends the 🔁 card. Routing a passed founder's email here would file it
  silently with no card (Kismet, 2026-09-23).

- **Hit on a live pipeline or Committed Opp** (Status NOT terminal and NOT `Active Portfolio`/
  `Portfolio: Follow-On`/`Exited`) **AND the email carries a material signal** (an attachment —
  check `dash_mail.py attachments <rowid> <dir>` — a doc link, or ≥400 chars of substantive body):
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
card), scheduling, newsletters, **vendor / cold sales pitches** (someone selling Tom a product —
a Ramp card, SaaS, services; same rule as the Inverted gate's "cold sales"). When unsure → silent. False negatives are fine (Tom sees his
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

ANY hit → do NOT fire a new 🆕 card; the company is already his. Route by the hit's Status:
- **Non-terminal / live** → if the mail carries genuinely new signal on an EXISTING company (a
  new round kicking off), that's a follow-on, not a new card — hand off to `add-follow-on-round`
  / `materials-handler`, don't fire a 🆕 card.
- **Terminal** (Pass (Met), Pass (DNM), Lost, NR / Missed) → run the **Revive Gate**,
  `~/.claude/skills/shared-references/revive-gate.md` — enrichment, update-email PDF, rebrand, 🔁 card, staged payload and `target_status`
  all live there; do not restate them here. Dash-lane extras on the staged payload:
  `mail_source:"dash-local"`, `rowid`, `fund`. Pass `matched_via` through when present.

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
`👍 to Add to CRM. 🗑️ to archive. Respond to make changes.` — the `🗑️ to archive` clause
appears on every EMAIL-sourced card (Dash here, Inverted via add-to-crm Step 4T), never on
iMessage cards; sms-listener §4 archives via `dash_mail.py archive`). Subject/stage/HQ rules per `deal-lane.md` §3.

**Source = provenance, NOT the founder.** A founder emailing Tom's Dash inbox **directly** (no
referrer in the thread) is a **Direct inbound → `Source: Direct`** — the founder is never their
own source. Only when someone else **forwarded/introduced** the founder is Source that
**referrer's** name. Since Dash founder mail is usually the founder writing Tom directly, the
common case is `Source: Direct`. For a referral, resolve the referrer against the People DB and
NEVER auto-create a row for an unknown one — carry their email with a `(not in People DB)` marker
(per `add-to-crm` Test 1). This matches `add-to-crm`'s `sourceDirective`: `"Direct"` OR
`{email, name}`.

Resolve the founder's HQ via ContactOut on an in-thread LinkedIn if the mail doesn't state it.
Render in PLAIN TEXT (iMessage) per the alert convention's text lane — no Slack markup.

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
  "description": "…", "source": "Direct",   // "Direct" for a direct founder inbound; else the REFERRER's name
  "source_context": "<what the email said>",
  "contact": "<founder email or N/A>", "website": "N/A", "icon": "…",
  "links": ["…"], "deck_drive_link": "<drive url or null>",

  "mail_source": "dash-local",
  "rowid": 202459,
  "fund": "Inverted 1️⃣",

  "proposed_at": "<ISO8601>"
}
```

- If the deal email has a **deck attachment**, extract it (`dash_mail.py attachments <rowid> <dir>`)
  and upload to Drive at proposal time (Drive Upload Apps Script, `drive-upload.md`) so
  `deck_drive_link` is ready and the confirm is instant. No deck → `deck_drive_link: null`.
  **Name the uploaded file per the internal convention (materials-handler principle 10):**
  `[Company] - Deck MM.DD.YY.pdf`, date = the email's SENT date (e.g. `LookingPro - Deck 09.18.26.pdf`)
  — NEVER the founder's raw attachment name (`LookingPro Seed 2026.pdf` is a defect). Stage a
  `deck_label` field equal to that exact filename; the confirm step chips `deck_drive_link` with
  `deck_label`, so **the Notion chip label and the Drive filename are byte-identical.** (Tom, 2026-09-18:
  the chip and the Drive file must carry the same convention name — an ad-hoc deck name is a bug.)
- `mail_source: "dash-local"` + `rowid` let the confirm-time create (or a fallback full
  add-to-crm run) fetch the email via `dash_mail.py` instead of the Gmail API.
- `fund: "Inverted 1️⃣"` — NEW deals from the Dash inbox are filed under **Inverted**, never Dash
  (Tom, 2026-09-24: "regardless of whether the opportunity is sent to dash or inverted, in the Notion field make it inverted — I'm not investing in new deals out of dash anymore"). The confirm handler stamps it on the new Opp (see `sms-listener` confirm §4). This applies
  to NEW Opps only — materials/updates for existing Dash portfolio rows keep `"fund":"Dash 2️⃣"`
  (the jobs above), and a revive never touches an existing row's Fund.

## Exit

No candidate is a deal → exit silently. No text, no log noise beyond the watcher's own line.
Never write to the CRM here — proposing is the whole job; the 👍 (sms-listener) does the add and
fires the Slack new-Opp alert.
