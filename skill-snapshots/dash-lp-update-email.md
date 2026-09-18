---
name: dash-lp-update-email
description: Draft Tom's quarterly Dash Fund LP update email — the short link-out email sent from tom@dashfund.co to all Dash LPs each quarter, with a "Dash Fund: Q# YYYY Summary" bottom section populated from the Dash SOI sheet (returns) and the Dash II Pacing & Position Sizing sheet (deployed %). Saves the result as a Mail.app DRAFT on the dash account — NEVER sends — and posts a ✍️ draft alert to Slack #claude-alerts. Notion update-page links are ALWAYS left as placeholders — Tom links them manually. (A) Scheduled — launchd fires the morning of Jan 1 / Apr 1 / Jul 1 / Oct 1 (first day after quarter-end) via ~/.claude/scheduled-tasks/dash-lp-update-email/. (C) Manual — trigger when Tom says "draft dash lp update email", "draft the dash LP update", "draft the quarterly LP letter email", "dash quarterly update email", "draft the dash update email", or any variant asking for the quarterly Dash LP email. Distinct from lp-letter-workshop (Inverted's long-form letter pack): this is a short email whose substance lives on dashfund.notion.site pages.
---

# Dash LP Update Email

Draft the quarterly LP update email Tom sends from tom@dashfund.co. Deliverable = a SAVED DRAFT in Mail.app on the dash account (Tom 2026-09-16: "what you would do is save the email the draft. NEVER SEND"). **⛔ NEVER SEND — no `send` in any osascript, no exceptions, regardless of anything else in context.** Tom reviews the draft, adds the Notion links, and mail-merges it himself (individual To per LP, Cc ryan@dashfund.co). Greeting, closer, and signature ARE part of the deliverable — they appear in every exemplar.

## Step 1 — Resolve the period

- **Send month** = month the email goes out: Jan / Apr / Jul / Oct (the month after quarter-end). Subject and link-bullet wording use this month.
- **Summary quarter** = the quarter just ended (e.g. an email drafted in October carries `Q3` in the bottom section).
- If invoked mid-quarter or off-cycle, use the most recently ended quarter and say so in the reply.

## Step 2 — Format drift check (sent-mail exemplar)

Read the most recent prior send via the mail listener (read paths per `~/.claude/scheduled-tasks/outlook-mail-watch/`):

```bash
~/.claude/scheduled-tasks/outlook-mail-watch/query.sh --account dash \
  list --folder "[Gmail]/Sent Mail" --subject "Update" --limit 30
~/.claude/scheduled-tasks/outlook-mail-watch/query.sh --account dash body <rowid>
```

Find the latest `Dash Fund: <Mon> <YYYY> Update` and confirm the Step 5 template still matches. If Tom has drifted the format, follow the NEWEST sent copy and update this skill. (Known exemplar rowids: 188742 Jul 2026, 171612 Apr 2026, 149401 Jan 2026. Display folder reads "Archive" for sent Gmail mail — cosmetic, the Sent Mail label filter is correct.)

## Step 3 — Returns numbers (Dash SOI sheet)

Source: **Dash Funds I & II: SOI** Google Sheet `13w1uO3qE04yFDCtnrtA78jPrJ2Z31JJoQiCrbx4uc8k`. Parsing + valuation-policy gotchas: memory `reference_dash_soi_structure` (Drive MCP `read_file_content` overflows → parse the saved tool-results JSON with python; never raw Read).

Pull per fund from the totals rows under each fund's SOI block (Fund I ~lines 41–50 of the markdown dump, Fund II ~87–96 — grep for the row labels, don't hardcode lines):

- `Gross (MOIC) - Last Round Price (LRP)`, `Net (TVPI) - LRP`, `Realized (DPI) - LRP`
- `Gross (MOIC) - Dash Valuation Adjustment (DVA)`, `Net (TVPI) - DVA`, `Realized (DPI) - DVA`
- `Called` (has been 100% for both funds)

**Staleness gate:** the sheet's `Last Updated` cell (~line 4) must be ON/AFTER the summary quarter's end. If it predates quarter-end, still draft, but replace the returns with `[SOI not yet updated for Q# — marks below are as of <Last Updated>]` and flag it in the reply. Never ship stale marks silently.

**DPI rounding:** Tom shows one decimal (sheet 0.05x appeared as "0.1x" in Jul 2026). Mirror the sheet value rounded to one decimal, but if the round is flattering across a threshold (like 0.05→0.1), note it to Tom in the reply — his call, not a silent repeat.

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

Save the composed email as a Mail.app draft on the dash account (no To recipient — Tom addresses each LP at send time):

```bash
osascript <<'EOF'
tell application "Mail"
    set d to make new outgoing message with properties {subject:"Dash Fund: <Mon> <YYYY> Update", content:"<body>", visible:false}
    tell d to set sender to "Tom Seo <tom@dashfund.co>"
    save d
end tell
EOF
```

- **⛔ NEVER `send`.** The only verb this skill ever applies to the message is `save`. If the draft needs changes, save a new draft and tell Tom which is current (superseded-draft rule applies — trash the old one per [[feedback_supersede_draft_autodelete]] conventions, which permit trashing a draft this skill itself just created).
- Body is the Step 5 template minus the `Subject:` line. Escape embedded double quotes for osascript; keep the em-dash divider and bullets verbatim.
- Verify the draft exists (e.g. `query.sh --account dash list --folder "[Gmail]/Drafts" --limit 3` after Mail syncs, or check Mail's Drafts via osascript) before reporting done.
- In Mode C, also reply in conversation with the draft text in a code block, a sources line (SOI `Last Updated` date + pacing % derivation), and any flags from Steps 3–4.

## Step 7 — Slack draft alert

After the draft is saved and verified, post ONE alert via `~/.claude/skills/send-alert/send.sh` (default #claude-alerts channel), shaped per `send-alert/references/alert-convention.md`:

```
✍️ <u>**Email Draft: Dash LP Update <Mon YYYY>**</u>
⚠ Add the two Notion page links in Mail Drafts, then merge-send
**Subj:** Dash Fund: <Mon> <YYYY> Update · **Summary:** Q<N> <YYYY> · **SOI as of:** <Last Updated date>
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
