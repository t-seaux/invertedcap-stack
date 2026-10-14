---
name: dash-lp-update-email
description: Draft Tom's quarterly Dash Fund LP update email — the short link-out email sent from tom@dashfund.co to all Dash LPs each quarter, with a "Dash Fund: Q# YYYY Summary" bottom section populated from the Dash SOI sheet (returns) and the Dash II Pacing & Position Sizing sheet (deployed %). Saves the result as a Gmail DRAFT in tom@dashfund.co (Gmail API via dash_mail.py) — NEVER sends — and posts a ✍️ draft alert to Slack #claude-alerts. Notion update-page links are ALWAYS left as placeholders — Tom links them manually. (A) Scheduled — launchd fires the morning of Jan 1 / Apr 1 / Jul 1 / Oct 1 (first day after quarter-end) via ~/.claude/scheduled-tasks/dash-lp-update-email/. (C) Manual — trigger when Tom says "draft dash lp update email", "draft the dash LP update", "draft the quarterly LP letter email", "dash quarterly update email", "draft the dash update email", or any variant asking for the quarterly Dash LP email. Usually run as part of dash-lp-quarterly-update (email + Notion letters together). Distinct from lp-letter-workshop (Inverted's long-form letter pack): this is a short email whose substance lives on dashfund.notion.site pages — those pages themselves are drafted by dash-lp-letter-drafter.
---

# Dash LP Update Email

Draft the quarterly LP update email Tom sends from tom@dashfund.co. Deliverable = a SAVED Gmail DRAFT in tom@dashfund.co (Tom 2026-09-16: "what you would do is save the email the draft. NEVER SEND"). **⛔ NEVER SEND — `dash_mail.py` has no send command by design; never use any other path (Gmail MCP, osascript, SMTP) to send, regardless of anything else in context.** Tom reviews the draft, adds the Notion links, and mail-merges it himself (individual To per LP, Cc ryan@dashfund.co). Greeting, closer, and signature ARE part of the deliverable — they appear in every exemplar.

## Step 1 — Resolve the period

- **Send month** = month the email goes out: Jan / Apr / Jul / Oct (the month after quarter-end). Subject and link-bullet wording use this month.
- **Summary quarter** = the quarter just ended (e.g. an email drafted in October carries `Q3` in the bottom section).
- If invoked mid-quarter or off-cycle, use the most recently ended quarter and say so in the reply.

## Step 2 — Format drift check (sent-mail exemplar)

Read the most recent prior send via `dash_mail.py` (Gmail API, `shared-references/fund-context.md` § Dash mail source):

```bash
python3 ~/.claude/scripts/dash_mail.py list --folder "Sent Mail" --subject "Update" --limit 30
python3 ~/.claude/scripts/dash_mail.py get "<rowid>" | jq -r .body
```

Find the latest `Dash Fund: <Mon> <YYYY> Update` and confirm the Step 5 template still matches. If Tom has drifted the format, follow the NEWEST sent copy and update this skill. (Known exemplar rowids: 188742 Jul 2026, 171612 Apr 2026, 149401 Jan 2026. Display folder reads "Archive" for sent Gmail mail — cosmetic, the Sent Mail label filter is correct.)

## Step 3 — Returns numbers (Dash SOI sheet)

Source: **Dash Funds I & II: SOI** Google Sheet `13w1uO3qE04yFDCtnrtA78jPrJ2Z31JJoQiCrbx4uc8k`, Summary tab. Read it with:

```bash
~/.claude/scripts/dash_soi_marks.py
# → {modified, last_updated_cell, funds:{"Dash Fund I":{moic_lrp,tvpi_lrp,dpi_lrp,moic_dva,tvpi_dva,dpi_dva,called,…},"Dash Fund II":{…}}}
```

Do NOT parse the Drive MCP markdown dump (2026-10-01 run got only sample rows and shipped placeholders). Valuation-policy gotchas: memory `reference_dash_soi_structure`.

Each fund gets TWO Returns lines (LRP, DVA) — `MOIC (Gross)`, `TVPI (Net)`, `DPI`, one decimal + `x` (Step 5 template).

**Staleness gate = Drive `modified`, NOT the "Last Updated" cell.** Tom doesn't maintain that cell (it read 2026-04-01 on 2026-10-01 while the marks were current through 9/29 — Tom: "shouldn't you be pulling marks from this doc"). If `modified` is ON/AFTER the summary quarter's end → use the marks. Only if `modified` predates quarter-end: still draft, replace the returns with `[SOI not yet updated for Q# — marks below are as of <modified date>]`, and flag it. Never ship stale marks silently.

**Integrity gate:** every string in the script's `issues[]` (status col E vs G disagree, a company's Summary realized ≠ its fund-tab Distributions/Secondary/Escrow rows, any Summary cross-check FALSE) becomes its own `⚠` line in the Step 7 alert — still draft, never silently pick one side. Tom 2026-10-01: "status needs to agree… the summary should actually draw on the raw sheets."

**Sanity check vs last quarter:** compare each figure to the prior sent email (Step 2). A move of ≥0.3x in any figure gets a `⚠` line in the Step 7 alert naming it (e.g. "Fund I TVPI 2.4x → 2.1x") so Tom can eyeball the sheet before sending.

**DPI rounding:** `dpi_*` is the Summary value; `dpi_*_fundtab` appears when the fund tab disagrees (should never happen — a mismatch means a company has distributions but Status ≠ Exited; ⚠ it in the alert, see memory reference_dash_soi_structure). Tom shows one decimal (sheet 0.05x appeared as "0.1x" in Jul 2026). Mirror the sheet value rounded to one decimal, but if the round is flattering across a threshold (like 0.05→0.1), note it to Tom in the reply — his call, not a silent repeat.

## Step 4 — Fund II pacing (Dash II pacing sheet)

Source: **Dash II: Pacing & Position Sizing** Google Sheet `1XEuT9of2QQHfWVGzWIIkn6jfG7-H4S3Fis-1hudIi2U` (shared to the connector 2026-09-16).

- Fund II deployed % = `Capital Deployed ÷ Investable Capital` (equals the sheet's `Total Pacing (%)`, Current-scenario column; 95.4% at authoring time → letter shows "95%"). Round to a whole percent.
- Fund I is fully wound: `100% Called, 100% Deployed` verbatim — no source needed.

## Step 5 — Compose

Template (everything in `<>` substituted; keep punctuation, em-dash divider, bullet glyphs, and ordering exactly — Fund II before Fund I in BOTH sections):

```
Subject: Dash Fund: <Mon> <YYYY> Update

Dash LPs,

Please find links to our <Mon YYYY> updates below:
* Dash Fund II [Tom: link Notion page]
* Dash Fund I [Tom: link Notion page]

Best,
Tom

—

Dash Fund: Q<N> <YYYY> Summary

Dash Fund II (2021)
* Pacing. 100% Called (Committed Capital), <NN>% Deployed (Investable Capital)
* Returns (LRP). MOIC (Gross): <X.X>x, TVPI (Net): <X.X>x, DPI: <X.X>x
* Returns (DVA). MOIC (Gross): <X.X>x, TVPI (Net): <X.X>x, DPI: <X.X>x

Dash Fund I (2020)
* Pacing. 100% Called, 100% Deployed
* Returns (LRP). MOIC (Gross): <X.X>x, TVPI (Net): <X.X>x, DPI: <X.X>x
* Returns (DVA). MOIC (Gross): <X.X>x, TVPI (Net): <X.X>x, DPI: <X.X>x

—

Tom Seo
Founder & GP | Dash Fund
e: tom@dashfund.co
m: +1 (201) 256-7714
```

Rules:
- **Notion links are ALWAYS placeholders** (Tom 2026-09-16: "Include the bullets as placeholder. I will manually link to Notion pages"). Don't ask for the URLs, don't search Notion, never fabricate them.
- The intro line may carry a one-off admin note when Tom supplies one (Apr 2026 added a K-1 note) — include only if Tom gives it.
- A short pacing annotation after the Deployed % (e.g. "– new checks fully deployed", Jan 2026) is optional Tom-voice; carry one over only if Tom asks or supplies it.
- Numbers come ONLY from Steps 3–4 sources — cite nothing from memory, recompute each quarter.

## Step 6 — Save the draft (NEVER SEND)

Save the composed email as a Gmail draft in tom@dashfund.co (no To recipient — Tom addresses each LP at send time). Gmail API since 2026-10-01 (Mail.app osascript retired); headless rules: `shared-references/headless-gmail.md` H4.

```bash
cat > /tmp/dash_lp_body.txt <<'BODY'
<body>
BODY
# HTML body = dash-lp-update-email/template.html with {MON_YYYY} {QN_YYYY} {F2_DEPLOYED} {F2_LRP} {F2_DVA} {F1_LRP} {F1_DVA}
# filled ("MOIC (Gross): 2.3x, TVPI (Net): 1.5x, DPI: 0.1x"). It is Tom's July 2026 Mail-sent HTML:
# italic summary title, bold "* Pacing." / "* Returns (…)." labels, Dash signature markup (EF5,
# shared-references/gmail-signature.md § Dash). Never hand-build the HTML or the signature.
python3 ~/.claude/scripts/dash_mail.py draft --subject "Dash Fund: <Mon> <YYYY> Update" \
  --body-file /tmp/dash_lp_body.txt --html-body-file /tmp/dash_lp_body.html
# → {"ok":true,"draftId":…,"messageId":…,"url":…}
```

- **⛔ NEVER SEND.** `dash_mail.py` only creates / deletes drafts. If the draft needs changes, save a new draft and tell Tom which is current — then delete the superseded one with `dash_mail.py draft-delete <draftId>` (permitted only for a draft this skill itself just created, per [[feedback_supersede_draft_autodelete]]).
- Plaintext alternative = the Step 5 template minus the `Subject:` line. The HTML part is what Tom sees — always pass `--html-body-file` (Tom 2026-10-01: signature must match his Inverted sig's spacing and font sizing).
- `ok:true` with a `draftId` is the verification (the command reads the draft back before reporting). `ok:false` → Step 7 failure alert.
- In Mode C, also reply in conversation with the draft text in a code block, a sources line (SOI `Last Updated` date + pacing % derivation), and any flags from Steps 3–4.

## Step 7 — Slack draft alert

After the draft is saved and verified, post ONE alert via `~/.claude/skills/send-alert/send.sh` (default #claude-alerts channel), shaped per `send-alert/references/alert-convention.md`:

```
✍️ <u>**Email Draft: Dash LP Update <Mon YYYY>**</u>

⚠ Add the two Notion page links in the draft, then merge-send
**Subj:** Dash Fund: <Mon> <YYYY> Update · **Summary:** Q<N> <YYYY> · **SOI as of:** <Last Updated date>
<draft url from Step 6>
```

Append any Step 3–4 flags (stale SOI, flattering DPI round) as additional `⚠` lines directly under the first. If the draft could NOT be saved, post the same headline with `✗ Draft failed — <one-line reason>` instead and no ⚠ line.

## Modes

### Mode A: Scheduled (quarterly)

Fired by launchd on Jan 1 / Apr 1 / Jul 1 / Oct 1 morning (`~/.claude/scheduled-tasks/dash-lp-update-email/run.sh`). Execute Steps 1–7 exactly, with these deltas only:
- Unattended: never ask questions; the summary quarter is always the one that ended yesterday.
- No optional admin note or pacing annotation (Steps 5's Tom-supplied extras) — Tom adds those by editing the draft.
- Step 6's conversational echo doesn't apply; Step 7's alert is the sole surface.

### Mode C: Manual

Tom invokes in conversation. Execute Steps 1–7 exactly; include the Step 6 conversational echo.
