---
name: dash-distribution
description: Ask Carta (Dash fund admin) to set up an LP distribution for Dash Fund II / II-A after exit proceeds or an escrow release land. Finds the payment notices in tom@dashfund.co, runs the II ↔ II-A rebalancing calc (Tom's Vantager/Teal workbook, live formulas), and saves a Gmail draft to dashfund@carta.com with the calc xlsx attached. Never sends. (C) Manual — "run distribution workflow", "run the distribution workflow for Dash", "ask Carta to set up a distribution", "set up a Dash distribution", "distribute the [company] proceeds / escrow", "we got escrow from [company], distribute it". Dash II / II-A only. Inverted (Vector) and Dash Fund I are out of scope.
---

# dash-distribution

Dash Fund II and II-A are parallel funds and always invest together ([[reference_dash_ii_parallel_funds_naming]]). Checks were sized on the **original** commitment split. A **$300k GP commit later moved from II-A to II**, so proceeds belong on the **adjusted** split. Payers (acquirers, SRS Acquiom) wire on the original split, so every distribution needs a small cash transfer between the funds before LPs are paid. Carta (`dashfund@carta.com`) is the fund admin ([[carta-logins-personal-vs-dash]]).

Mail: `~/.claude/scripts/dash_mail.py` only (headless-gmail.md H4). Never the Gmail MCP (that's the Inverted account).

## Step 1 – Find the proceeds

Use Tom's message if it names the company and amounts. Otherwise search Dash mail for unrequested proceeds:

```bash
python3 ~/.claude/scripts/dash_mail.py search 'newer_than:60d (from:srsacquiom.com OR "payment update" OR escrow OR proceeds OR "wire" OR distribution) -from:tom@dashfund.co' --limit 30
```

- You need **one amount per fund**. SRS sends one notice per entity: "Dear Dash Fund II, LP … Your payment of X USD" plus a matching "Dash Fund II-A, LP" notice (consecutive Ref #s). Read both with `dash_mail.py get`.
- **Already requested?** Check `runs/ledger.jsonl` and `search 'to:dashfund@carta.com distribution newer_than:90d'`. If this payment is already in a request, stop and tell Tom.
- **Only one fund's notice found** → ask Tom. Never assume the other fund got $0.
- A Tom → Carta "FYI this is an escrow payment" forward is just a heads-up. It doesn't count as a distribution request.

## Step 2 – Calc + draft

```bash
cd ~/.claude/skills/dash-distribution && python3 scripts/dist_calc.py \
  --company "<Company>" --fund2 <II amount> --fund2a <II-A amount> \
  --event "an escrow payment from our <Company> exit" --draft
```

What the script does:
- Writes `runs/<date>-<company>/<Company> Distribution Calculation MM.DD.YY.xlsx`. The layout matches Tom's sheets: original vs adjusted split, then Received → % Transfer → $ Transfer → Adj. Distribution, all live formulas. A copy also lands in `~/Downloads`.
- Computes the transfer with Tom's formula, `(adjusted II % − original II %) × total` (≈1.4291% of total, II-A → II). This reproduces Teal Dec 2025 ($315.20), Teal Jan 2026 ($1,471.23) and Vantager Mar 2026 ($3,572.70) to the cent.
- Saves the Gmail draft to Carta with the xlsx attached and the Dash signature. The body is Tom's canonical template (`references/prior-emails.md` ★), third bullet's amounts bold ("Which leaves" unbold). To refresh an existing draft in place (no delete), add `--replace-draft <draftId>`.

Checks before reporting:
- `split_off: true` means the payer did NOT wire on the original split, so the formula's assumption breaks. `--draft` then REFUSES (exit 2) and prints the received split. Tell Tom; re-run with `--split-confirmed "<what Tom said>"` only after he says to draft anyway.
- `--event` is required: write what the notice says the payment is (escrow, holdback, final proceeds). The script never fills it in.
- **Attach the xlsx. Never send a Google Sheets link.** Carta's alias can't open Sheets (Dec 2025, Jessica asked for Box instead). The attachment worked in Mar 2026.
- **Never net the transfer against the year-end interco balance.** Carta (Jessica, 2026-03-02): clear the YE interco separately, for the paper trail. If Tom mentions interco, it's a separate line/transfer.
- `--note "<text>"` adds a paragraph, e.g. which bank accounts to distribute from (Dec 2025 asked SVB batch-wire template vs Citi). Add it only if Tom raises it.

## Step 3 – Report + ledger

Append one line to `runs/ledger.jsonl`: `{"date","company","fund2","fund2a","transfer","lp2","lp2a","source_msg_ids":[…],"draftId"}`.

Reply to Tom with the three numbers (wired split, transfer, LP amounts) and the draft link. Tom sends ([[feedback_never_ask_if_good_to_send]]). After sending, Tom moves the transfer cash himself. He may also enter the distribution request in Carta, though Carta has said they can set it up on the backend (Jan 2026).

## Step 4 – Sent check (never assume a draft is still pending)

Before reporting status, or whenever Tom asks about a distribution, check Dash Sent:
`python3 ~/.claude/scripts/dash_mail.py search 'in:sent to:dashfund@carta.com subject:"distribution – <Company>"' --limit 5`.
If it's there, mark the ledger line `status: "sent"` with `sent_message_id` / `sent_at` and report it as sent. The draft
link is dead once the email is sent. (Tom, 2026-10-03: the Teal note was sent and I should have found it.)

## Follow-ups Carta typically sends (no action from this skill)

Beneficiary address missing for some LPs → Tom asks Carta to contact the LPs directly ("carries more weight coming from fund admin").

## Changing the constants

`FUND2_COMMIT` 17,542,284 / `FUND2A_COMMIT` 3,450,216 / `GP_COMMIT_SHIFT` 300,000 in `scripts/dist_calc.py`, sourced from "Vantager Distribution Calculation.xlsx" (Tom → Carta 2026-03-02). Change them only on Tom's say-so and re-run the regression in `references/prior-emails.md`.
