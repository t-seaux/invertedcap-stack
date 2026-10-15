---
name: portco-ask
description: |-
  Portfolio founder asks Tom for people in a category ("do you know any lenders…", an update's "Asks:" list) → a 🙋 Portco Ask card texted to Tom; 👍 runs the matching network / talent / investor scan and nests the results under the card; Tom replies with numbers to draft intro asks. Rules: shared-references/portco-asks.md. Job-only – Mode email (gmail-webhook intro-agent category-only hit from a portfolio founder), Mode execute (sms-listener 👍 via apply_portco_ask.py). The text lane runs inside deal-text-scanner; investor updates hand off from investor-update Step 4.7. Covers named-person intro asks too (type named → intro-outreach-drafter Mode C, every lane) – from portfolio AND pipeline founders (pipeline: named only, card shows the Opp's Status); Mode email-offer = a founder approving a list Tom proposed (Pattern 4 → register-list + offer-drafts). Category asks from pipeline founders are out of scope.
---

# portco-ask

> **Headless Gmail:** every Gmail read in this skill follows `shared-references/headless-gmail.md`.

Rules – what counts, types, keys, card flow, edge cases: **`shared-references/portco-asks.md`** (read it first).
Mechanics: `portco_ask.py` in this folder (docstring has every exit code). This file is transport only.

## Mode email – args `{mode:"email", messageId, threadId, from, subject, date, body, opp_id, opp_name, categories, mailbox?}`

`mailbox:"dash"` = the ask came to tom@dashfund.co (Dash IS a lane, Tom 2026-10-14): any extra read goes through
`dash_mail.py` (headless-gmail H4), never the Gmail MCP; put `mailbox:"dash"` in the propose payload. Otherwise identical.

Enqueued by gmail-webhook `intro-agent` (or intro-agent's reconcile sweep, its Step 1.6) when the classifier says
intro ask AND the mail isn't an investor update (those come via investor-update Step 4.7): from a PORTFOLIO founder a
category OR specific named people (type `named`, spec § 2); from a PIPELINE founder (any Opp not intro-closed) named
people only. `opp_status` in the args is a hint – use `context`'s live `status`.

1. Read `body` (the webhook passes ≤4000 chars of the founder's NEW text, quoted history stripped). Judge it against
   spec § 1 – the webhook classifier is cheap and generous; silence is fine.
2. `portco_ask.py context --opp <opp_id>` (what the company does now) and `portco_ask.py prior --opp <opp_id>`
   (reuse keys) – spec § 3.
3. Build the payload – `lane:"email-adhoc"`, `company` = the Opp title, `opp_status` = `context`'s `status` (required;
   a pipeline Opp takes `named` asks only and its card shows the stage – code refuses the rest), `founder` = sender's name (`role` if not a
   founder), `source_ref` = messageId, `gmail_id` = messageId, `mailbox` (from args, if set), `source_text` = body – and pipe it to
   `portco_ask.py propose`. exit 2/3 → fix the payload (the errors say what) and rerun; never loosen a verbatim.
4. No alert of your own – the card is the alert. Nothing to card → end silently.

## Mode email-offer – args `{mode:"email-offer", messageId, threadId, from, subject, date, body, offer_message_id, offer_text, opp_id, opp_name, opp_status}`

Enqueued by gmail-webhook `intro-agent` (and intro-agent's reconcile sweep, Pattern 4) when an open-Opp founder replies
in a thread where TOM proposed a list of people (`offer_text` = Tom's proposal, `body` = the founder's new text).
Spec § 4b "Tom proposed the list himself" – the same code the text lane runs, so the outcome is identical.

1. Read `body` against `offer_text`. Not an answer to Tom's list (thanks, scheduling, a question only) → end silently.
2. `portco_ask.py context --opp <opp_id>` (live Status; intro-closed → end silently).
3. `portco_ask.py register-list --opp <opp_id> --company "<opp_name>" --founder "<sender name>" --lane email
   --source-ref <offer_message_id> --offer-text "<offer_text>"` with ONE row per person Tom proposed (`name`, `company`,
   `role` if stated, `linkedin` if linked) on stdin. Names the founder adds that Tom never proposed → a second call with
   `--append` (they become the un-offered rows).
4. Map the founder's verdicts to row numbers: "yes" / "strong fit" / "go ahead" → picked; "skip" / "not a fit" → not
   picked; "tell me more about X" / "not sure" → not picked (never a yes). Blurb: `pending` ONLY when the founder said
   one is coming and it isn't in this email; otherwise `none`.
5. `portco_ask.py offer-drafts --opp <opp_id> --results list-<offer_message_id> --pick <picked rows> --offered <every
   row Tom proposed> --who "<founder>" --blurb pending|none --source-ref <messageId> --source-text "<body>" --offer-text
   "<offer_text>"` – it drafts now (Mode draft job), HOLDS for the blurb, or cards the un-offered picks. No other alert.
   Never write 👓 Qualified yourself – the drafter does when it drafts.

## 👍 on a results text → Google Sheet

`portco_ask.py sheet --results <handle>` (sms-listener routes the tapback; Tom 2026-10-14). Chains into investor-crm:
its `from_scan` row rules + `crm_sheet.py push` (locked template). Investor scans land in the company's ONE Investor CRM
(read back first – Tom's cuts/edits kept, only new people added); talent / network scans get a sheet per ask target.
Strong fits = main section, 👀 6–7s + held = "Weaker fits (for review)".

## Mode execute – args `{mode:"execute", card_handle, company, opp_id, founder, lane, source_ref, gmail_id, ask{key, verbatim, type, target, query, executor}}`

Enqueued by `sms-listener/apply_portco_ask.py` after Tom's 👍. One job = one ask.

0. `python3 ~/.claude/skills/portco-ask/portco_ask.py context --opp <opp_id>` – judge fit against what the company
   does now (latest update over a stale Description). Description stale vs the latest update →
   `opp_description.py offer` (spec `shared-references/opp-description-drift.md`), then carry on.
1. Run the executor's SEARCH steps only, READ-ONLY – no Notion writes, no candidate-store / cache writes, no drafts,
   no Slack posts, whatever that skill's later steps say:
   - `network-scan` → **the same scan Tom gets asking in a session, never a single top-k pass** (Tom 2026-10-11: "scan
     the network cache as if I asked you to do it in a Claude session"; the Palla run did one `vsearch` top-40 of
     5,897 profiles):
     a. **Judge every person card** – `portco_ask.py judge --card <card_handle> --ask "<ask.verbatim>" --context
        "<what the company does now, from step 0>" --exclude-firm "<company>" [--exclude-firm-types VC,angel unless
        the ask is for investors]`. Every profile's work-history card (all roles, what they did, firm context) is read
        against the ask: a cheap recall screen, then a precision re-score of the survivors. `out_file` = the kept
        people with a score + a why grounded in one role. `results` refuses without it once person cards exist.
        `--precision-model opus` for a high-stakes ask. **Past intros: scored blind, shown split** (Tom 2026-10-11): the judge scores
        fit without history; intro'd people with no card are judged from their People row (no LinkedIn → `url`
        `notion:<id>`; send them with no `linkedin`, or drop them by `name`). `results` does the rest in code – ONE
        ROW PER VC FIRM (venture capital only – Tom's intro'd contact there first, even if the judge didn't
        keep them; colleagues fold into "+N more at <Fund>"; credit funds, PE, banks, companies and unclassified firms never fold – each fit is its own row), on INVESTOR scans only (talent / network = one fit-ordered list – Tom 2026-10-14) "🔁 Intro'd Before" then "🆕 Never Intro'd" only when someone has Made intros (else one plain
        list), numbered straight through, fit then intros taken / share, no intro-history line, every ≥8 fit sent in one go (no cap – Tom 2026-10-12; long lists split across texts); the 6–7s fill the same text up to 40 rows total, broken out as "⭐ Strong Fits (n)" then "👀 Also Worth a Look (m)" (Tom 2026-10-14), only the rest wait behind "more". Send `people` in any order; one per FUND is enough (folded fund colleagues
        count as covered for the ≥8 rule) – at a bank or company send every ≥8 fit, never write markers, don't mention past intros in `why`. Drop an intro'd contact
        who left the fund (`dropped` by name) so code doesn't make them the fund's row.
     b. **Recall safety net** – `portco_ask.py sweep --card <card_handle> --terms "<≥3 keywords: the category, its
        synonyms, sub-types, the titles such people hold>" --query "<ask.query>" --query "<angle 2>" [--query …]`
        (angles = the ask as written, the provider/firm side, the person/title side), plus network-scan Step 2a
        `csearch` (`--no-vc` unless the ask is for investors) and Step 2b `vsearch` per angle; FTS `query` for any
        firm/name that surfaces. Anyone these surface that the judge did NOT keep: read their card
        (`person_cards.py card <url>`) and keep them only if the card shows a real fit.
     c. **Verify every keeper scored ≥8** – `network_cache.py dump <url>` and read the FULL profile: confirm the
        CURRENT role (a card can lag a job change), rewrite `why` from what the profile says. EVERY judge fit scored
        ≥8 goes in `people` – or in `dropped: [{linkedin, why}]` with a concrete reason (left the firm, at the
        portfolio company, a VC). Never trim the list for length: `results` refuses (exit 2) if any ≥8 fit is
        missing, and stages the 6–7s itself (in the text up to 40 rows, the rest as "Reply more") (Tom 2026-10-11 – the job had cut 140 fits to 36).
     Drop anyone at the portfolio company itself and anyone at a VC firm unless the ask is for investors.
   - `talent-scan` / `coinvestor-recommender` → the SAME a–c as network-scan with `judge --kind talent` (ask = the
     role profile from `ask.query`, talent-scan Steps 1–2; no JD needed) or `judge --kind investor` (ask = the round /
     next-round profile, Mode B for `company`; never `--exclude-firm-types VC`; its curated sources are the recall
     net, intro history is attached after scoring – blind). Same `results` (Talent / Investor Scan headline); Tom
     2026-10-11 "every kind of network scan … same structure / format".
   - `network-scan` for an `at-company` ask → `csearch` scoped to that one company (current employees first, then
     alumni who'd still know the team). Judge + sweep first too (judge ask = "current employees or recent alumni of X";
     sweep terms = the company's name variants + domain; angles = "current employees of X", "alumni of X").
   - `intro-outreach-drafter` (a `named` ask) → NO scan and NO results message: run that skill's Mode C for the named
     people on this Opp, purpose = the founder's ask, then send the done-beat exactly as Mode draft step 2.
2. Keep EVERY person Tom could credibly intro the founder to – no cap, best first; drop only weak fits. Rank by fit to the ask (scored blind – Tom 2026-10-11; intro history only labels the results, never ranks them). Each
   gets `why` = one line grounded in their profile (current role / tenure / what they run). Never invent a role.
3. Send (`results` = `portco_ask.compose` – the SAME composition a Claude-session scan prints via network-scan
   `scan.py render`; change the list format there, never in one surface – Tom 2026-10-11 "same brain"):
   ```bash
   python3 ~/.claude/skills/portco-ask/portco_ask.py results --card <card_handle> <<'J'
   {"company": "...", "opp_id": "...", "founder": "...", "target": "<ask.target>", "executor": "<ask.executor>",
    "people": [{"name": "...", "role": "...", "company": "...", "linkedin": "https://www.linkedin.com/in/...", "why": "..."}]}
   J
   ```
   exit 2 → fix the listed fields and rerun. Zero fits → `people: []` (it says so and invites a different angle).
4. Done. Tom's numbered reply on the results is handled by sms-listener (confirm-routing § Portco ask results).

## Mode draft – args `{mode:"draft", results_handle, reply_to?, company, opp_id, founder, target, picked_by?, pick_text?, offer_text?, blurb_text?, blurb_source?, vehicle?, forward_of?, forward_subject?, people:[{name, role, company, linkedin}]}`

Enqueued by sms-listener when Tom answers the results with numbers or names, by a 👍 on a `🙋 Draft Intro Asks`
card, by `offer-drafts` directly (people Tom already offered, no blurb coming), or by `blurb-landed` releasing a
hold (`reply_to` = the `🙋 Holding Intro Drafts` text, `blurb_text` set).

1. **`vehicle: "forward"`** (the founder wrote `forward_of` to be forwarded whole – EF6 "decide the vehicle", EF8b):
   for each of `people`, a real forward of that email – resolve their email exactly as intro-outreach-drafter does
   (People DB, identity match; a work address at a past employer = flag it in `missing`/`why`, never guess a new one),
   write ONLY the cover note in the `portco-ask-forward` voice (writing-style/portco-ask-forward/STYLE.md), then
   `gmail-create-draft.py --to <email> --forward-of <forward_of> --subject "Fwd: <forward_subject>" --skill
   portco-ask-forward --html-body-file … --snapshot-text-file … --no-alert`. The founder's email rides below
   untouched – no About block, nothing lifted or re-voiced. Then stage them in 👓 Qualified as the drafter does.
   **Otherwise** run `intro-outreach-drafter` Mode C for `people` on this Opp, purpose = `target` (the founder's ask).
   Drafts only. **No usable email for a person** (both vehicles) → `shared-references/intro-recipient-rules.md` R3: the LinkedIn DM is drafted in Beeper
   (a forward → cover note + the founder's blurb through his closing line, `--founder`; never the forwarded email) and goes
   in the done-beat's `dms` list, not `missing`. `blurb_text` (when present) is the founder's blurb – it goes into each draft's About block VERBATIM,
   perspective included (`I'm Charlie, …` stays first person – EF6, Tom 2026-10-06); without one, the drafter pulls
   the 📚 callout as usual.
   `pick_text` is the founder's opt-in, `offer_text` Tom's own offer – honor their color ("don't know her too well but can try" → a
   warmer, lower-pressure ask; "Edy" is how Tom addresses Edouard).
2. Done-beat, nested under `reply_to` when present, else the results (or the card, for a `named` ask):
   ```bash
   python3 ~/.claude/skills/portco-ask/portco_ask.py drafted --reply-to <reply_to or results_handle> <<'J'
   {"company": "...", "opp_id": "...", "drafts": [{"name": "...", "to": "...", "subject": "..."}],
    "dms": [{"name": "...", "status": "drafted|no-chat"}],
    "missing": [{"name": "...", "why": "..."}]}
   J
   ```
   Every picked person appears once: in `drafts` or in `missing` with the reason. exit 2 → fix and rerun.

## Mode email-blurb – args `{mode:"email-blurb", messageId, from, subject, body, opp_id, opp_name}`

Enqueued by gmail-webhook `intro-agent` when a PORTFOLIO founder's email carries a forwardable blurb (its classifier's
`forwardable_blurb`). Confirm it IS one (spec `blurb-capture.md` § "When it fires" #2; a one-liner in a scheduling
note is not), then **decide the vehicle** (email-formatting.md EF6, Tom 2026-10-06):
- `blurb` – a description of the company handed over to reuse (`here's a quick blurb`, an About paragraph);
- `forward` – the email itself is written to be forwarded: it names who it's for and makes the ask
  ("Love to chat with Sarah if she's open to it…", subject "Collections at Upstart?"), or says "feel free to forward this".
Then `portco_ask.py blurb-landed --opp <opp_id> --opp-name "<opp_name>" --lane email --ref <messageId>
--text "<the blurb paragraph(s), verbatim from body>" [--vehicle forward --subject "<subject>" --note "<whole body>"]`.
It captures the blurb (📚 callout) and releases the hold – for `forward`, only the people the note names (the
rest wait for their own note; a forward note naming no one goes to everyone held). Not a blurb → end silently.
