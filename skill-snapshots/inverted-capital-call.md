---
name: inverted-capital-call
description: Set up a capital call for an Inverted fund (today Inverted Capital I; capital calls are tracked PER FUND via funds.json) – drafts ONE email, to Vector AIS (fund admin), asking them to issue the call; never an email to LPs. Call history is read from the LP portal (read-only) and must tie before anything is drafted. Never sends. (A) Automatic – when Vector's capital call notice lands in Tom's personal (LP) inbox, texts a 👍 card to mark the call Completed in the LP portal; nothing changes until Tom 👍s. (C) Manual – "set up a capital call", "ask Vector to set up a capital call", "run the capital call workflow", "call X% for Inverted", "next cap call". An LP heads-up is NOT this skill – it only goes inside the quarterly LP letter email when Tom asks ("give LPs a heads up on a capital call in the LP letter" → lp-letter-workshop, which uses this skill's --paragraph-only sentence). Inverted only; Dash distributions are dash-distribution.
---

# inverted-capital-call

**Capital calls are per fund (Tom, 2026-10-03).** `funds.json` maps each Inverted fund's legal name (exactly as
Vector writes it, e.g. `Inverted Capital I, LP`) to its own LP portal inputs, engine, publish script, portal URL
and Vector address. Every step resolves the fund first and never mixes funds:
- `cap_call.py --fund "<name>"` (default Inverted Capital I, LP) – an unknown fund stops with `UNKNOWN_FUND`.
- The detector reads the fund from the notice subject (and requires the body to name the same fund). An Inverted
  fund without an entry gets a heads-up text and nothing is staged. **Dash notices are ignored: Tom is done with
  Dash capital calls.**
- Cards, reminders and the 👍 apply are keyed by (fund, call #) and name the fund.

**Adding a new fund:** Tom has no LP portal for any fund but Inverted Capital I yet (2026-10-03). A new Inverted
fund first needs its own portal built, a separate project when Tom closes one. Only then does it get a `funds.json`
entry (portal inputs with `capital_call_schedule` + `capital_called_pct`, a `call-complete` engine, a publish script).
Until then its notices only produce an FYI text.

Vector AIS (`invertedcap@vectorais.com`) issues the formal notices to LPs on Valence ([[vector-ais-fund-admin]]). Tom's job is the ask to Vector. Gmail access follows `shared-references/headless-gmail.md`.

⛔ **Never address an email to LPs from this skill** (Tom, 2026-10-03). The only LP-facing piece is the heads-up sentence in the quarterly LP letter email, added only when Tom asks for it there.

## Step 1 – Inputs

From Tom: **% to call**, plus timing if he names one (default "end of next week"). Check LP holiday cut-offs (MASIC flags Eid every year).

## Step 2 – Draft the Vector email

```bash
cd ~/.claude/skills/inverted-capital-call && python3 scripts/cap_call.py \
  --pct <N> --notices-by "<timing>" --draft      # required: Tom's timing ask (e.g. "end of next week"); ask if unsaid
```

- **History tie-out:** the script sums `capital_call_schedule` Completed rows in `~/code/lp-portal/fund_inputs.json` and stops (`CALL_HISTORY_MISMATCH`) unless the total equals `capital_called_pct`. If it stops, tell Tom. Don't patch the portal: it's read-only ([[lp-portal-read-only]]).
- **Over-call guard:** cumulative > 100% stops the run.
- **Subject:** `Inverted Capital I - Capital Call #<n> (<x>%)`, e.g. `Inverted Capital I - Capital Call #4 (15%)` (Tom, 2026-10-03).
- **Body:** Tom's own template (2026-10-03, `references/prior-emails.md` ★), verbatim. Only %, cumulative and timing are filled; the GP cashless reminder is part of it. The GP commit is 70% cashless, so the GP notice calls 30% in cash. Vector got this wrong in Sep 2025 and had to reissue.

## Step 3 – Report

Show Tom the preview, the call math (call #, this %, cumulative after, uncalled after) and the draft link. Tom sends.

When Vector publishes the notices, Mode A picks them up from Tom's personal inbox and asks to update the LP portal. No reminder needed.

## LP-letter heads-up sentence (used by lp-letter-workshop only)

`python3 scripts/cap_call.py --pct <x> --issue-date "<date>" --when "<when>" --paragraph-only` prints Tom's verbatim sentence with the call number and cumulative % computed from the same tied history:

> An early heads up – later this month we plan to issue our fourth capital call for 15% of your commitment amount. This will bring cumulative capital called to 65%. Vector will reach out with a formal notice with details soon.

## Mode A – notice lands → 👍 card → LP portal update (Tom, 2026-10-03)

Tom is an LP in Inverted Capital I, so Vector's notice ("Inverted Capital I, LP Capital Call due …", from
`no-reply@valence.vectorais.com`) reaches `thomas.seo@outlook.com`.

1. **Detect:** `scripts/capcall_detect.py` rides outlook-mail-watch's `watch.sh` on every tick (no model, no new
   launchd job). It skips "additional commitment" catch-up notices, reads the % and due date from the notice
   body, and takes the next open row of `capital_call_schedule`. If the notice states a call number ("Capital Call #4",
   "fourth capital call") that isn't that row, it texts a ⚠ mismatch and stages nothing; an unnumbered notice (Vector's
   2026 template) stages with "Call # inferred from the portal schedule" on the card. One card per notice and per call #
   (`runs/capcall-seen.json` + staged-file check). If it can't read the notice, it texts Tom and stages nothing.
2. **Ask:** it texts Tom `💸 Capital Call #N Issued` with the change: the call's month goes from the planned
   estimate to the month the notice went out, the status to Completed, and capital called from before → after
   (if the % differs from plan, it also lists how the future estimates rebalance to keep the schedule at 100%, so the portal's call bar redraws in proportion to what was actually called). The card ends `👍 to update the LP portal`.
3. **Apply only on 👍:** sms-listener `references/capital-call-confirm.md` runs `refresh_inputs.py call-complete`
   (dry-run, then apply), then `run.sh` to publish, then replies ✅. This is the one sanctioned write to the
   LP portal from this skill ([[lp-portal-read-only]]: Tom-gated, per change).
4. **Nudge:** the LP portal's daily 7am `run.sh` runs `scripts/capcall_remind.py`. A card unanswered for 24h+ gets one
   ⏰ reply under it per day until Tom reacts. Reminder only: the daily run never decides a call went out.

## Follow-ons Vector typically handles (no action here)

New-LP catch-up calls after a close (Jun 2026: "called 50% to date so that'd be the catch up %"), management fee true-ups, GP notice revisions.

## Simulation harness (required – shared-references/automation-harness.md)

`python3 tests/sim_capcall.py`: 20 end-to-end scenarios in a sandbox (on/off-plan, rebalance cascade, over-100%,
catch-up / Dash / unknown fund / I vs II vs I-A, unreadable notice, spoofed sender, duplicates, reminder + 👍 on
reminder, double 👍, decline, edited numbers, stale card, publish failure, next call). Runs automatically on any
commit touching this skill or the sms-listener branch. Add a scenario for every new production failure.
