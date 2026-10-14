---
name: dash-lp-letter-drafter
description: Draft Tom's quarterly Dash Fund I and Dash Fund II Notion LP letters (the "Dash Fund I/II: <Mon YYYY> Update" pages linked from the quarterly LP email), using the newest published letters on dashfund.notion.site as the template and the Dash SOI + Dash II Pacing sheets as the only number sources. Tom duplicates last quarter's pair in the dashfund workspace, renames them `[WIP] … <Mon YYYY> Update` and unpublishes them. The skill then patches the copies block by block via the Dash API token. Ping Tom when a new pair of duplicates is needed. The Inverted workspace is never used. NEVER publishes. Trigger on "draft the dash letters", "draft the dash LP letters", "draft the Q3 dash letter", "dash fund I / II update page", or any request for the Notion letter itself (not the email). Usually run as part of dash-lp-quarterly-update (email + letters + tie-out). Pairs with dash-lp-update-email, which drafts the short email that links to these pages. Manual-only.
---

# Dash LP Letter Drafter

Two letters per quarter, published the month after quarter-end (Jan / Apr / Jul / Oct). **Fund II and Fund I are shaped differently. Never merge their templates.**

## Step 1 — Read the newest published pair (format drift check)

Index: `https://dashfund.notion.site/Dash-Fund-LP-Letters-6e935211705748a5812519e13bc9375b`. The index is in another workspace, so subpages 404 on `app.notion.com/p/<id>`. Fetch them as `https://dashfund.notion.site/<id>` with notion-fetch. The Fund I page overflows the tool limit: parse the saved file with python and strip the `prod-files-secure` image URLs. The newest pair is the template. If Tom has changed the structure, follow the newest pair and update this skill.

## Step 2 — Numbers (sources only, never memory)

- **Performance:** `~/.claude/scripts/dash_soi_marks.py` → MOIC / TVPI / DPI for LRP and DVA, one decimal. Use the staleness, integrity and DPI rules from `dash-lp-update-email` Step 3; every `issues[]` entry goes in the draft-notes callout.
- **Valuation Adjustments table: shift left (Tom, 2026-10-01).** In the duplicated page, the prior letter's right-hand (current-quarter) column becomes the new left column, copied as published in that letter. The new right column is the reported quarter, read from the SOI **`Valuation Framework` tab** (live counts per status: Fund I = col F, Fund II = col H, rows Hold 🏁 … Exited 📍). Delta = right − left (`+n` / `-n` / `-`). Total = sum of each column. Cross-check against the fund-tab col G status counts; any mismatch goes in the reply as a ⚠. Q3 2026 matched: Fund I 11/3/1/3/1/10, Fund II 14/7/1/2/2/3.
- **Exits are SOI subrows dated in the quarter, incl. stock-for-stock M&A** (`<Acquirer> Stock (M&A)`: no cash, so no DPI line; block = `**Co (N.Nx MOIC).** Co was acquired by X in a stock transaction that closed <Month D, YYYY>. We received X common stock in exchange for our position.` Hansa, Oct 2026). Never put confidential deal terms (clawbacks, per-share price) in the public letter.
- **Quarter activity:** subrows (col I round label, col J date inside the quarter). Follow-on = a priced round, SAFE or secondary that Dash bought into. Exit or distribution = `Distributions` / `Secondary` / `Escrow`. For round size, check the company's Company Updates row (founder announcement) and its `(… FO)` Opp. Lead investors come from SOI col Q.
- **Exits cross-check (Tom 2026-10-04, Hansa miss):** before drafting, run `python3 ~/.claude/skills/dash-lp-quarterly-update/exit_scan.py "<Mon YYYY>"`. Exit 1 = an M&A / exit sits in Company Updates or Notes with no SOI booking → stop and ⚠ Tom ("book it in the SOI first?"). Never add an exit to the letter without the SOI row. tie_out.py re-runs it as a gate.
- **⛔ Sent letters are NEVER edited without Tom's explicit permission AND guidance (Tom 2026-10-04).** Once the LP email is sent, the published pages are frozen. A missed item (e.g. an unbooked exit) → tell Tom what's missing and propose options (patch live / carry into next letter), then WAIT. Edit only after he says to and how. Hansa (Oct 2026) was patched live only because Tom said "Update Oct letter now". A carry-forward, if he picks it, goes in `corrections.json` `carry_forward["<Mon YYYY>"]` and is added verbatim.
- **Fund II pacing:** Dash II Pacing sheet `1XEuT9of2QQHfWVGzWIIkn6jfG7-H4S3Fis-1hudIi2U` (it links to the SOI live). Use `Total Pacing (%)` and the first / follow-on split from the **60/40** scenario column, as whole percents.

## Step 3 — Compose

**Fund II** is written fresh each quarter, and only that quarter's activity goes in:
archive callout → `Dash II LPs,` → 💼 Portfolio Performance (`As of MM/DD/YY:`) → 🔧 Valuation Adjustments table → `### Highlights` → 💸 New Investments ("No new investments; Fund II is fully deployed on new checks.") → 📈 Follow-On Rounds (`We invested $Xk in N follow-on round(s) in Q#.` + one logo|text column per round: `[**Co**](url)** Round ($Xk follow-on, $Ym invested to date).** $Zm round; round led by L. N.Nx MOIC.`) → 🚪 Exits **only if any happened that quarter** → SOI summary link (`gid=476672787`) + screenshot → 🏃‍♂️ Pacing → `### *Dash Fund II: Portfolio (Final)*` image.

**Fund I** is a cumulative ledger, so copy the prior letter **verbatim** and only apply deltas:
fixed intro sentence (a–d) → Performance → table → SOI link (`gid=2075368769`) + screenshot → 🏁 Exits ("Dash I has N exits to date.", newest quarter first) → 📈 Follow-On Rounds (newest `*Q# YYYY*` group **prepended**; each block lists the full round chain `Coinvested with … → Series X led by …`) → ❌ Writeoffs (append; a company moving to Winddown becomes `(Expected: Full Writeoff)` and is flagged for Tom to confirm, and an Expected entry flips to `(Realized: …)` once it closes) → `### *Dash Fund I Portfolio (Final)*`.

Both letters:
- Put a yellow `✍️ DRAFT NOTES — delete before publishing` callout at the top, listing every change vs the prior letter, every SOI integrity issue, and every judgment call.
- Logos and screenshots can't be carried across workspaces (signed URLs expire). Use `🖼️` placeholders and red `[Tom: …]` spans.
- Write the archive link as the absolute dashfund.notion.site URL.
- Closed deals only. Anything not closed (signed TS, expected close) stays out of the letter body and is noted in the draft-notes callout at most.

## Follow-on rounds: Fund II vs Fund I (they differ)

**Fund II (still writing checks): what Dash II put in this quarter.**
- Lead: `📈** Follow-On Rounds. **We invested $<sum>k in <N> follow-on round(s) in Q#.` The sum covers Dash II checks only, rounded to $10k (e.g. $540k, $550k). Use "1 follow-on round" (singular) when there's one.
- Include **every priced round a Dash II company closed that quarter, even with no Dash check**. Show that as `($0 follow-on, $750k invested to date)` (Spade Series B, Q1'26). Count those in N, but they add $0 to the sum.
- Per block: `[**Co**](site)** <Round> ($<check>k follow-on, $<total>m invested to date).** $<size>m round; round led by <Lead>[ with participation from <X>]. <N.N>x MOIC.` MOIC is Dash II's **position** MOIC after the round. Invested-to-date is the SOI parent cost, written $0.6m / $1.3m / $1.75m (use two decimals only when one would mislead).
- Cross-fund follow-ons get a footnote asterisk: `rounds*` in the lead, then `* Given that Dash 1 is fully deployed, we made our follow-on investment in <Co> via Dash 2.`, with any portfolio-count change underlined. A company in both funds shows up in both letters. **Caplight is held by both funds** (Tom, 2026-10-01). Dash I holds the Pre-Seed SAFE + Seed ($129k, 2.8x). Dash II holds the Series A follow-on ($150k, Jun '26), which took Dash II from 28 to 29 companies. Any future Caplight round therefore goes in **both** letters. In Dash II it's a normal block: its check, or `$0 follow-on` if Dash II sits it out. In Dash I it's a ledger entry with Dash I's position MOIC. Each fund's valuation table and MOIC use only its own tranches. Check for other dual-fund names by comparing company names across the two fund tabs.
- Optional look-ahead at the end of the lead line: `FYI we are expecting at least two follow-on rounds in Q2 (OatFi Series B, Suppli Series A) – closing in process.` Add it only when the CRM or SOI shows a live TS. In Notion this is the "Rounds in Progress – Live Term Sheet(s)" legend.
- No rounds that quarter: say so in one line. Never drop the section.

**Fund I (fully deployed, no new money): a running record of every priced round its companies have raised.**
- Lead (fixed): `**📈 Follow-On Rounds. **Please keep confidential as many of these rounds are unannounced.`
- Add a new `*Q# YYYY*` italic group **at the top**. Everything older stays word for word, newest first.
- Include every priced round a Dash I company closed, whatever fund (if any) put money in. Caplight's Series A was funded by Dash II but is still logged in Dash I.
- Per block: `[**Co**](site)** <Round> ($<round size>m, <N.N>x MOIC). **Coinvested with <seed co-investors> at <first round> → <Round> led by <Lead> (+ joined by <X>) → …`. That's the full round-by-round chain to date, extending the company's previous block by one step. MOIC is Dash I's position MOIC. Older entries say "MOIC on 1st Check". New entries drop that suffix, and the round size is left out when it isn't known.
- Exits (`🏁 Exits. Dash I has N exits to date.`, newest quarter first) carry the realized MOIC and DPI contributed ("This exit returned 0.3x in DPI."). A partial secondary counts as an exit: `**EvenUp Partial Secondary (20.2x MOIC). **We sold half of our position at the Series D valuation ($1b+). This exit returned 0.3x in DPI.`

**Sources for both: the SOI is the source of truth for which rounds happened (Tom, 2026-10-01).** A round is in the letter if and only if its fund-tab subrow's close date (col J) falls in the quarter. `$0`-cost subrows are rounds the fund sat out. Don't search Company Updates, the CRM or the news for rounds the SOI doesn't have, and don't ask Tom to confirm them. If the SOI doesn't have it, it isn't in the letter. For the text of each block, round size comes from the Company Updates / FO Opp, and leads come from SOI col Q. **CLOSED deals only (Tom, 2026-10-01).** A signed TS, a round mid-close or a pending secondary does not go in the letter body, not even as a placeholder. It goes in the quarter it closes. At most, mention it in the draft-notes callout. The one exception is Fund II's optional "FYI we are expecting…" look-ahead line, which Tom writes himself.

## Notion layout primitives (exact values, read from the published letters)

Reference renders are in `references/screenshots/` (`dash1-2026-07.png`, `dash2-2026-07.png`, `dash2-2026-04.png`). Re-capture them with `references/capture_letters.py`. dashfund.notion.site sits behind a Cloudflare bot check, so headless Chromium stalls on "Just a moment…". Run a headed browser positioned off-screen (`--window-position=-2400,0`), wait for `.notion-page-content`, then grow the viewport to the scroll height before the screenshot.

- **Page:** Notion **Mono** font, default (not full) width, gradient image cover, a new emoji icon each quarter (🚞 ☄️ 🖥️ 🪃 📖 …). The API can't set the font. Covers and icons can be set.
- **Callout:** 🗃️ `gray_bg` ("Archive of historical quarterly updates here.").
- **Valuation table:** `header-row` + `header-column`. Column widths are **58.99 / 221.99 / 138.00 / 138.00 / 138.00** px (#, ADJUSTMENT, prior Q, current Q, Delta). The Total row is `orange_bg` with bold totals. Emit the `<colgroup>` every time; omitting it renders equal-width columns.
- **Company blocks:** a 2-column layout with ratio **6.25 / 93.75**. The left column holds only the square logo image. The right column holds the linked bold name + bold round label, then plain detail. Dash I groups blocks under italic `*Q# YYYY*` lines. Writeoff blocks are bold text only.
- **Section leads:** emoji + bold label + period, e.g. `💼** Portfolio Performance. **`, inline with the text that follows.
- **SOI summary image:** never hand-screenshot. Run `references/soi_summary_png.py <outdir> Q<N>_<YYYY>`. It Drive-copies the SOI, **collapses the grouped company-detail columns (C–N) in the copy** so the label column sits right next to the cohort columns, as Tom screenshots it, exports `B9:S36` per fund tab as a PDF, crops it to PNG, and deletes the copy. The live sheet is never touched. An export without the collapse comes out about twice as wide, with a blank band in the middle.
- **Portfolio (Final) image:** unchanged since Fund II finished deploying. Carry it over.

## Step 4 — Duplicate + patch in the Dash workspace (NEVER publish)

The Dash workspace API token is the "Claude – Dash Letters" internal connection, stored encrypted at `~/.claude/secrets/dash-notion-token.enc`. Decrypt it with `SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -d --input-type binary --output-type binary …`. Use `curl` with `Notion-Version: 2026-03-11` and always quote URLs (zsh globs `?`). Don't use `ntn api` with this token: it hung.

The public API has **no page-duplicate endpoint**, and it can't set the Mono font or table column widths. That's why the July pages get duplicated by hand:
1. Tom duplicates last quarter's two pages in Notion (••• → Duplicate) and moves the copies into his **private "Drafts"** page. Never leave them under "Dash Fund LP Letters": that page is published at dashfund.notion.site, so a draft there is public. The Drafts page must have the connection added (••• → Connections).
2. The skill patches the copies **block by block** (`PATCH /v1/blocks/{id}` on rich_text). That keeps fonts, widths, logos and the cover. New company rows are a `column_list` with `width_ratio` 0.0625 / 0.9375 (the API exposes this) and a logo uploaded with `POST /v1/file_uploads`. The SOI summary image is swapped the same way. Rename the title to `<Mon YYYY> Update` and set a new emoji icon.
3. Fetch the result back and check it, then reply with both links and the flags. Tom publishes by moving the pages under Dash Fund LP Letters.



## Conventions added 2026-10-01 (Tom)
- **TVPI label = `TVPI (Net)` everywhere** (letters, email, SOI; Tom 2026-10-01). No "Net of Fees" or "Net of Fees & Carry" variants. The method is net of earned fees & carry (adjusted, see below), but the label stays `(Net)`.
- **TVPI is ADJUSTED, live in the SOI (Tom, 2026-10-01).** The SOI's TVPI cells (Fund I/II tabs S19/S25; Summary P44/P48/P90/P94) compute `1 + 0.8 × ((total value + add-back) ÷ called − 1)`, pulling the add-back by IMPORTRANGE from the private sheet "Dash Funds: Unearned Management Fees & Adjusted TVPI" (`1ri3J5pFXMkymdK2YdNLRunSdMUSaBZTHDBAZkQgNFm8`; shared only with tom@invertedcap.com + tom@dashfund.co). Add-back = **net cash** (all fund cash less amounts owed; Tom 2026-10-01: standard NAV includes cash, and future fees are not a liability). D14 for Dash II (II + II-A), D15 for Fund I. Unearned fees (col J) are for reference and for sizing exit holdbacks. Each quarter, confirm with Tom that nothing is owed at quarter-end (9/30/26: confirmed nothing owed). No fee schedule lives in the SOI, and no static numbers. Each quarter: set the private sheet's as-of date (B2), refresh cash on hand (quarter-end money-market statements + LP checking; exclude "MC" management-co accounts), then read TVPI straight from the SOI (`dash_soi_marks.py`). Letter, image and SOI then tie automatically. The render script reads D14/D15 live, because IMPORTRANGE doesn't resolve inside the temp copy. Fee terms: Dash II 2.5% on $17,542,284 from 7/23/2021; II-A 2.5% on $3,150,216 from 8/28/2021 (2025 audit); Fund I 0.375%/qtr mgmt + 0.25%/qtr platform admin = 2.5%/yr for 40 full quarters after Initial Closing (initial closing date TBD → F9). 10-year terms.
- **Dynamic inputs (2026-10-01):** the private sheet's as-of date (B2) auto-rolls to the latest quarter-end, and cash rolls forward from the anchor (B14/B15 at E14/E15) minus fees charged since. Re-anchor cash (new B and E values) after any follow-on check, distribution or exit, or when quarter-end money-market statements arrive. Tom sends the statements around the 15th after each quarter-end (Apple Reminders on the Work list through 7/15/2027; add the next four when the last one is due). Fund I's cash at the current fee rate runs out around 2028, so the add-back shrinks toward $0 unless distributions refill cash.
- **Valuation table delta:** `-` when unchanged, otherwise signed integers (`+1`, `-2`). Total row the same.
- **Page icon:** a new emoji every quarter, never one already used in the archive (list the archive icons first). Oct 2026: Dash II 🍁, Dash I 🎃.
- **Logos must be SQUARE icons, not wordmarks (Tom 2026-10-04).** Every logo renders at the same column width (~26px), so a wide wordmark (Hansa's "hansa" text) reads as tiny next to a filled square (Outmarket's ring). If the CRM icon is a wordmark, use the company site's apple-touch-icon / webclip (`<link rel="apple-touch-icon">`), else crop to the mark. Eyeball against the neighboring logo.
- **Logos must match the company.** Pull from the company's own site icon or the CRM Opp icon. Don't trust an old letter's logo: Jan and Apr 2026 had a wrong Outmarket logo (blue bag; the real one is the orange ring from outmarket.ai). After editing, build a contact sheet of every logo next to its text and eyeball it. Exited or written-off companies are low priority.
- **Writeoffs:** a company moving to Winddown 🏴‍☠️ is added as `<Co> (Expected: Full Writeoff)` (bold, with logo) after the last writeoff entry. It flips to `(Realized: …)` once closed.
- **Unpublish check:** the duplicate inherits site publishing. Confirm the WIP pages show no "live on dashfund.notion.site" banner before editing.

## History

- 2026-10-01: built; drafted the Oct 2026 (Q3) pair. Template = Jul 2026 letters.
