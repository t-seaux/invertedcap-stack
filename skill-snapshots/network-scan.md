---
name: network-scan
description: Query the People DB and cached LinkedIn network to surface relevant contacts by expertise, background, or relationship — to facilitate intros, find angels, identify advisors, or answer open-ended network questions.
triggers:
  - /network
  - who in my network
  - anyone in my network
  - people in my network
  - find potential angels
  - find angels for
  - should I reconnect
  - catch up with
  - network contacts
  - who should I reach out to
---

# /network — Network Intelligence Query

Query Tom's cached LinkedIn profile database to answer questions about his network. Surfaces relevant people with ranked reasoning.

## When to trigger

- Explicit: `/network <query>`
- Natural language: "who in my network has fintech experience", "find potential angels for Tuor", "who should I reconnect with given my ideal founder profile", "is there anyone I know who's worked at Stripe"

---

## Step 1 — Extract search intent and route

Parse the query to identify:
- **Target type**: angels, operators, advisors, founders, specific expertise, reconnect candidates, etc.
- **Context**: if a portfolio company is mentioned, note its sector and stage
- **Natural language intent**: rephrase as a full descriptive sentence for semantic search

**Then pick a primitive based on whether the discriminating signal is on the COMPANY or the PERSON:**

- **Company-trait queries** → `csearch` (company-side). Use when the filter is about the company someone worked at: revenue trajectory, sector, stage, funding, momentum, "worked at a 10x'er / hypergrowth / specific company", "operators from fintech infra", "anyone who's been at Stripe / Plaid / Ramp". The `csearch` primitive vector-searches the Companies DB, joins the pre-resolved `profile_companies` table, and returns each match with full tenure dates already extracted.

  **For revenue-trajectory queries** ("10x", "hypergrowth", "company that scaled ARR fast"): run `csearch` for recall, then intersect with the Deal Digest cache for factual revenue grounding — this is the default pipeline, not an optional step. See Step 2a below for the exact sequence.

- **Person-trait queries** → `vsearch` (person-side). Use when the filter lives in the person's bio/headline: expertise, role pattern, founder archetype, angel activity, "deep technical founders", "biotech investors", "operators with consumer marketing chops".

When in doubt, run both and merge; they hit different parts of the index.

- **Qualifying / fit-list queries** ("who could I intro to X for <a specific need>", "lenders for a receivables line",
  "who has run collections at a lender", "suggest VCs for <founder>") → **`scan.py judge` first** (Step 4 – the same
  judge + list as the portco-ask text path). It reads EVERY person's full work history, not a keyword hit or the first
  1,200 chars. 2a/2b are the recall net.

**Past intros are a core input on EVERY network scan – fit is scored BLIND, history only labels and folds**
(Tom 2026-10-11: past intros validate the relationship, but "if we keep just doubling down on folks I've already made
intros, then it makes it hard for us to discover new intros to make"; for customer asks there may be little history –
"that's okay! It's just one of the factors"). So:
- **Fit is scored without history.** The judge never sees who Tom intro'd; history is attached after scoring.
- **One row per VC FIRM only** (`intro_history.is_fund()` / `FUND_TYPES = ("venture capital",)` – Tom 2026-10-11:
  "given that theyre not vcs… list them out as their own entry"). At a VC firm, someone Tom has intro'd there (kept by
  the judge or not) is the row – best outcome if several – carrying the firm's best fit score; colleagues fold into
  "+N more at <Firm>: names"; no intro'd contact → the firm's best fit. Credit funds, PE, banks, fintechs (a lending
  fintech is never a fund), operating companies and unclassified firms never fold: every kept person is their own row
  on their own fit, and an intro'd contact there who wasn't kept is not added. Stealth / self-employed / blank never group.
- **Sections only on investor scans, and only when someone has Made intros** (Tom 2026-10-14: elsewhere "that doesn't really mean anything" – talent / network = one fit-ordered list under ⭐ / 👀): "🔁 Intro'd Before" first, then "🆕 Never Intro'd", fit order inside each.
  No history at all → one plain list, exactly the blind fit order.
- **Intro'd Before** ties break on outcome = successful intros only: intros taken, then share taken. No intro-history
  line in the output (Tom: "remove the whole history of intros") – the section header says it.
- No lookalike / "similar to" layer (Tom: redundant with fit ranking), no spreading / fatigue rule ("dont need to
  spread intros around").
Never hand-build a shortlist: every fit list goes through `scan.py render` (same output as texts).

---

## Step 2c — Person cards (whole-network qualifying pass)

A card per profile in `~/.claude/scripts/person_cards.db`: name, current role + firm type, EVERY role (title, firm, firm
type, dates, one concrete line on what they did, firm overview), expertise tags, who they're a credible intro for, a
3–5 sentence arc. Roles come from `network_firms.py` (every role, grouped headings included) with its firm type /
what-the-firm-does; `oneshot` adds one concrete line per numbered role + the arc, and a code gate checks every role
got one, each echoed firm matches, no invented role, and the name matches. Thin profiles (<2 roles and <3k chars, or a
logged-out LinkedIn scrape) are never carded – `thin` lists them for re-enrichment.

```bash
S=~/.claude/skills/network-scan/scripts/person_cards.py
python3 $S judge --ask "<the ask, verbatim>" --context "<what the portfolio company does>" --out /tmp/judge.json \
    [--exclude-firm-types VC,angel] [--exclude-firm "<the portco>"] [--precision-model opus]
python3 $S stats                      # ok / thin / low_fidelity / failed / missing / stale_hash + a measured build ETA
python3 $S build --missing-only       # card new profiles (unchanged raw_text is hash-skipped; re-enriched ones refresh)
python3 $S thin --out /tmp/thin.txt   # profiles too thin to qualify → re-enrich with enrich-file
python3 $S card <linkedin_url>        # one stored card (verbatim review)
```

`judge` = two passes: a cheap-tier recall screen over every compact card (score ≥4), then a precision re-score of
the survivors on their FULL cards (keeps ≥6, a why grounded in one role). BLIND to intro history: neither the prompts
nor the card text mention it; intro'd people with no card (no cache profile, or a thin one) are judged from their
People row (`intro_only`). After scoring, code adds `section` (new | intro_before) and `intro_history` (lists,
took / passed / asked, `marker`), merges the same person's duplicate cache rows, folds FUNDS to one row each (`firm_key`,
`fund`, `firm_type`, `folded`, `flipped_to_intro`; at a fund, intro'd contacts not kept get `card_status: intro-contact` and
a why built from the best colleague's; non-funds never fold) and orders rows fit → outcome; coverage counts `kept_people` vs `kept_firm_rows` + `folded_people` + `fund_rows`. Before judging it chains `intro_history.py ensure-fresh` (a resync only when the index is
>24 h old, ~40 s, read-only) – `--no-intro-sync` skips it; a failed sync is reported in `intro_history_error`. Its coverage JSON says how many were read,
passed, kept, too thin, missing, and any `chunks_failed` (those people are listed in `unjudged` – say so, never
imply they were read). Tiers are env-overridable family aliases: `PC_BUILD_MODEL` (sonnet), `PC_SCREEN_MODEL`
(haiku), `PC_PRECISION_MODEL` (sonnet). Before you return anyone, `network_cache.py dump <url>` and confirm the
current role – a card can lag a job change.

### Intro history (`intro_history.py`)

Local index of who Tom has intro'd: People DB `Intros (Made)` + the Opportunity relations `✉️ Intros (Made)`,
`👓 Intros (Qualified)`, `🚫 Intros (Declined / NR)` (that last one exists only on the Opp). Notion is READ-ONLY here.
Outcome = successful intros only (Tom: "not need to track beyond that"): took (Made) vs passed (Declined / NR). Opp
Status is Tom's own deal outcome and is NOT read as an intro outcome.

```bash
I=~/.claude/skills/network-scan/scripts/intro_history.py
python3 $I top --category Investor --limit 30   # most-intro'd investors
python3 $I person <li-or-name>                  # one person: made / lined up / declined-NR, with the Opp names
python3 $I stats                                # coverage, cache matches, age, top 10
python3 $I unmatched                            # intro'd people with no network-cache profile (enrich them)
python3 $I sync                                 # full resync (resumable); ensure-fresh = only if >24 h old
```

---

## Step 2a — Company-side search (preferred for company-trait queries)

```bash
python3 ~/.claude/scripts/network_cache.py csearch "<natural language query>" --no-vc --per-company 8
```

Returns JSON: array of company groups, each shaped as
`{notion_url, name, category, distance, momentum, overview, total_funding, employee_count, people: [{person, url, role, employer, dates, is_current}]}`.

Each `people[].dates` is the verbatim tenure tail extracted from the LinkedIn profile (e.g. `Jan 2020 - Jan 2021 • 1 year`). No further raw-text reads needed — Step 4's reasoning works directly on this output.

Flags:
- `--no-vc` — drop VC firms from company hits (recommended for operator queries).
- `--companies N` — pre-filter pool size from company vsearch (default 40).
- `--per-company N` — cap people per company in output (0 = no cap).
- `--max-distance D` — embedding distance gate (default 1.06).

**Revenue-trajectory queries — hybrid pipeline (default for "10x", "hypergrowth", "scaled ARR fast"):**

Run these three steps in sequence; the whole pipeline completes in ~5–10 seconds:

1. `csearch` for recall — semantic search over Companies DB + profile_companies join (broad net, may include VC firms without `--no-vc` context):
```bash
python3 ~/.claude/scripts/network_cache.py csearch "<revenue/growth query>" --no-vc --per-company 8
```

2. Deal Digest intersection — ground the csearch hits against factual revenue trajectories. Load the cache and filter to companies with documented ≥5x ARR growth:
```bash
python3 -c "
import json, re
cache = json.load(open('/Users/tomseo/.claude/skills/shared-references/deal-digest-cache.json'))
# Print companies with explicit ARR multiples or hypergrowth signals
for c, entries in cache.items():
    text = ' '.join(e.get('raw','') for e in entries)
    if re.search(r'\b([5-9]\d*x|10x|[1-9]\d{1,}x|\\\$\d+M.*ARR.*\\\$\d+[BM]|\\\$\d+[BM].*ARR)', text, re.I):
        print(c)
" 2>/dev/null
```

3. Intersect: keep only csearch hits whose company name appears in the deal-digest filter output. Annotate each person with the deal-digest revenue data for that company.

Each match's `momentum` field in csearch output also includes recent digest mentions inline — use that as a quick signal before loading the full cache.

## Step 2b — Person-side search (preferred for person-trait queries)

```bash
python3 ~/.claude/scripts/network_cache.py vsearch "<natural language query>" --limit 40
```

Returns JSON: array of `{linkedin_url, name, distance, context_blob, raw_text_preview, companies}`. Lower `distance` = more semantically similar (good results are typically under 1.05).

If vsearch returns <5 results or you need to find someone by exact company/name, supplement with FTS:

```bash
python3 ~/.claude/scripts/network_cache.py query "<keywords>" --limit 20
```

---

## Step 3 — Check cache stats (first run only)

If Step 2 returns 0 results, run:

```bash
python3 ~/.claude/scripts/network_cache.py stats
```

If the cache is empty, tell Tom:
> "Cache is empty — run the enrichment job first: `/network enrich`. Then export LinkedIn connections at linkedin.com/settings/data-privacy/export-data and run: `python3 ~/.claude/scripts/network_cache.py enrich-file ~/path/to/Connections.csv`"

---

## Step 4 — Judge, verify, render: the SAME brain as the text path, for EVERY kind of scan

Tom 2026-10-11: "even when I ask you to do this through our cloud session, the output, the way you go about flagging
these people, the way you organize it should be the same" · "text and Claude sessions should refer to the same brain".
A session scan and a portco-ask text scan run ONE code path (`network-scan/scripts/scan.py` → `portco_ask.compose`),
so the method and the text are identical. Every KIND of scan is this one method + format (Tom 2026-10-11: "every
kind of network scan regardless of what we're scanning for should be in same structure / format"): `--kind talent`
(talent-scan – the ask is the role profile) and `--kind investor` (coinvestor-recommender – the ask is the round
profile) change only the headline label (Talent / Investor Scan) and add a fit hint to the judge context
(`portco_ask.KIND_HINTS`); `--kind network` (default) covers everything else, at-company asks included. For EVERY fit-list ask – "who in my network for X", "find lenders / VCs /
customers / advisors for Y", "suggest people to intro to Z", "potential angels for …", and relationship / exploratory
asks like "who should I reconnect with given <profile>" (the ask is the profile):

```bash
S=~/.claude/skills/network-scan/scripts/scan.py
python3 $S judge --ask "<Tom's ask, verbatim>" --context "<who the intros are for / what they do>" [--kind network|talent|investor] \
    [--exclude-firm "<the company itself>"] [--exclude-firm-types VC  # unless the ask is FOR investors]
# → {"scan": "<id>", "out_file": …, "coverage": …}   out_file.kept = the fits (score, why, section, folded …)
```

1. **Judge** (above): every person card read blind against the ask, intro history attached after (Step 2c).
2. **Recall backstop**: Step 2a `csearch` / 2b `vsearch` / `query` for the ask's angles; anyone they surface that the
   judge did NOT keep: read their card (`person_cards.py card <url>`) and add them only if it shows a real fit.
3. **Verify every keeper scored ≥8**: `network_cache.py dump <url>` – confirm the CURRENT role from the full profile
   (a card can lag a job change), rewrite `why` from what the profile says (by name, never he/she). Every judge fit
   ≥8 goes in `people` or in `dropped: [{linkedin|name, why}]` with a concrete reason (left the firm, at the company
   itself, a VC on a non-investor ask). One person per VC fund is enough – colleagues fold under the fund's row.
4. **Render** and paste the output VERBATIM:

```bash
python3 $S render --scan <id> --target "<short plural target, e.g. lenders for a receivables line>" \
    [--for "<company or person the intros are for>"] [--founder "<person – their first name goes in the intro line>"] <<'J'
{"people": [{"name": "...", "role": "...", "company": "...", "linkedin": "https://www.linkedin.com/in/...", "why": "..."}],
 "dropped": [{"linkedin": "...", "why": "..."}]}
J
```

   (`render` takes the judge's kind; `--kind` that differs is refused.) It prints exactly the text Tom would get by
   text: "✅ <for / target>: <Network|Talent|Investor> Scan Completed", "N <target> you could
   intro <first name> to:" (or "N <target> in your network:" with no --founder), 🔁 Intro'd Before / 🆕 Never Intro'd (investor scans only)
   sections, bold numbered rows, one row per VC fund ("+N more at <Fund>"), the 🔎 coverage footer and the 💬 CTA;
   every ≥8 fit is sent in one go (no cap; long lists split across texts), then the 6–7s up to 40 rows total, in two sections "⭐ Strong Fits (n)" / "👀 Also Worth a Look (m)"; only the rest held for "more" (Tom 2026-10-14). Exit 2 = refused (a ≥8 fit neither sent nor dropped,
   a bad row) – fix the payload and rerun, never trim. Output the text as printed: no re-formatting, no markdown, no
   commentary inside the list; a short note AFTER it is fine. There is no --stage: replying "more" / numbers works on a
   texted list (it's keyed by the message handle) – offer to send it as a text if Tom wants that.

**Not a fit list – answer directly, no render** (the only exceptions): a single-person / single-fact lookup ("did X work
at Stripe?", "what does Y do now?"), counts and coverage questions ("how many fintech founders do I know?",
`network_cache.py stats`), and the enrichment commands below.

Judging notes the cards can't carry: **Founder framework ask** – Tom's founder heuristics (hypergrowth operator,
domain depth, archetype); read `founder-taste/RUBRIC.md` and put the rubric in `--context`. **Revenue-trajectory
asks** – intersect with the Deal Digest (Step 2a) before you verify.

---

## Enrichment commands (when Tom runs /network enrich or asks to populate the cache)

### From People DB (Notion)

Query People DB (`1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`) for all rows where the `LI` property is populated. Extract each LinkedIn URL and write to a temp file, then enrich:

```bash
python3 ~/.claude/scripts/network_cache.py enrich-file /tmp/people_db_urls.txt
```

### From LinkedIn connections CSV

Tom exports from: linkedin.com → Me → Settings & Privacy → Data Privacy → Get a copy of your data → Connections. The export CSV has a `Profile URL` column.

```bash
python3 ~/.claude/scripts/network_cache.py enrich-file ~/path/to/Connections.csv
```

### From ContactOut cache (free, no Exa credits)

```bash
python3 ~/.claude/scripts/network_cache.py enrich-contactout
```

Run this first — it's instant and imports anyone already enriched by neg1-enricher.

### Link profiles to company cache

After enriching, run this to build/refresh the person→company join table:

```bash
python3 ~/.claude/scripts/network_cache.py resolve-companies --missing-only
```

This matches each profile's employers (via LinkedIn company URL slugs) against Tom's Companies DB in SQLite. Results appear as `portfolio_companies` in every `query` / `vsearch` result.

### Check coverage

```bash
python3 ~/.claude/scripts/network_cache.py stats
```

---

## EXA_API_KEY setup

The script reads `EXA_API_KEY` from environment or `~/.claude/scripts/network.env`. If the key isn't set, write it:

```bash
echo "EXA_API_KEY=your_key_here" >> ~/.claude/scripts/network.env
```
