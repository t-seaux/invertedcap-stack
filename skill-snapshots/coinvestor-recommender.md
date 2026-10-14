---
name: coinvestor-recommender
description: |-
  Produce investor candidates to fill out a round Tom is leading — friendly follower checks + likely next-round leads who might write a smaller check now (plus angels when the deal warrants) — as an Investor Scan: the network-scan judge over every person card (blind fit) and the shared scan layout, identical on the text lane and in a session; evidence sources and persistence live in the skill body. Trigger: "recommend coinvestors for [company]", "who should I bring into [company]", "who could co-invest in [deal]", "who should come into the cap table", "build out the syndicate for [company]", "find coinvestors for [deal]", "who could fill out the round", "help [founder] fill the rest of the round", "suggest investors to bring in", or Tom describes a round structure (check size, round size, stage) and asks who in his network fits. ALSO Mode B — Next-Round Leads: who should LEAD a portfolio company's NEXT round (seed / Series A); trigger on "who should lead [company]'s next round", "next-round investors for [company]", "who leads [company]'s seed / Series A", "who do we take [company] to for their raise", "recommend next-round leads for [company]".

---

# Investor Recommender

> **Headless Gmail:** every Gmail read/write in this skill follows `shared-references/headless-gmail.md` — reads via `admin_run.py` when the Gmail MCP isn't attached (its absence ≠ Gmail down); a run that can't finish ends with a `JOB_FAILED:` line.

This skill runs in **two modes.** Detect which from Tom's ask; **default to Mode A.**

- **Mode A — Coinvestors (default):** who fills the **pre-seed round Tom is leading now** — friendly
  follower checks + next-round funds who might opportunistically come in early. Triggers: "recommend
  coinvestors for X", "who should I bring into X", "fill / build out the round".
- **Mode B — Next-Round Leads:** who should **LEAD a portfolio company's NEXT round** (seed / Series A)
  — connecting the founder to the right next-stage lead. Triggers: "who should lead X's next round",
  "next-round investors for X", "who leads X's seed / Series A", "who do we take X to for their raise".
  See **## Mode B — Next-Round Leads** near the bottom for its deltas.

**Almost everything is shared.** The output (an Investor Scan – below), the judge (Step 4.5), the fit model, Step 0
loop, the comprehensive sweep (Step 5), the People DB freshness sweep (Step 5.5), identity / current-firm checks
(Step 6), cache-reachability, and the persist-on-finalize loop (Step 8) are **identical across modes.** Steps 0–9
below are written for Mode A; **Mode B changes only the judge's ask (who LEADS the next round), the Step 1 framing,
the primary sources, the category buckets, and the corpus files — all specified in ## Mode B.** Read that section first when in Mode B,
then run the shared steps with its overrides in mind.

---

## The deliverable (both modes): an Investor Scan

An investor scan is a network scan (Tom 2026-10-11: "every kind of network scan regardless of what we're scanning for
should be in same structure / format" · "text and Claude sessions should refer to the same brain"; he chose
"Network-scan layout for all"). ONE method and ONE output for Mode A and Mode B, in a session and on the portco-ask
text lane (`investor` ask → `portco_ask.py judge --kind investor` / `results`):

1. **Judge** – `scan.py judge --kind investor` reads EVERY person card blind against the round profile (Step 4.5).
2. **Recall backstop** – the curated sources (Steps 0–5.5) and the cache sweep add anyone the judge missed, only on a
   real fit read from their card.
3. **Verify** every ≥8 keeper's CURRENT firm (Step 6).
4. **Render** – `scan.py render --kind investor` → "✅ <Co>: Investor Scan Completed", numbered
   `N. Name (Role @ Firm). Why. link ↗` rows, one row per VC firm ("+N more at <Fund>"), 🔁 Intro'd Before /
   🆕 Never Intro'd only when someone has intro history, every ≥8 fit sent in one go (no cap), 6–7s held, 🔎 footer,
   💬 CTA. Paste it verbatim (`network-scan/SKILL.md` Step 4).

No category headers, tiers, or flat/alphabetical by-firm lists in the scan output. The categories (Mode A: **Follower
Checks** / **Next-Round Investors – who may opportunistically invest at Pre-Seed** / **Angels**) are Tom's calls AFTER
the scan – he assigns them when he locks the list (Step 8), and they head the founder email (Step 9).

## The core principle — fit is judged blind; behavior is evidence, never a score

Fit is scored without relationship or intro history (Tom 2026-10-11: past intros validate the relationship, but
doubling down on them "makes it hard for us to discover new intros to make"). The judge never sees who Tom intro'd;
`intro_history` is attached after scoring and only sections, folds and tie-breaks the rows. So every source below lands
in exactly one of two places:

| Evidence | Examples | Where it goes |
|---|---|---|
| **Fit** – would this investor write THIS check? | stage they write at, sector / sub-problem thesis, geo, check size, leads vs follows, led or joined a comparable round, vertical-fund mandates | the judge's `--ask` / `--context` (the round profile), the cards, and the verified `why` ("Led Oun's pre-seed; $250-500k checks") |
| **Relationship / outcome** – how Tom knows them, what happened before | Qualified / Made / Declined-NR relations, Drive-CRM outcomes, `RELATIONSHIP STRENGTH`, friends-of-the-firm Bcc, Gmail cadence, co-investment graph | **never a score**: intro history is attached by code; a prior-round outcome is stated as a fact at the end of the `why` ("Passed on Fair's pre-seed, Aug 2026: pre-seed non-core") or is the reason in `dropped` |

The curated sources are still the best RECALL – they find names the cache can't (Steps 0–5.5) – and Tom reads the
relationship evidence when he locks the list (Step 8).

---

## The fit model — three axes: sector, stage, geo

Fit is not one thing. A candidate qualifies on **any one** of three axes, any combination, or all
three — **matching a single axis is enough to surface a name.** None is mandatory; none is a gate on
its own.

- **Sector** — works the deal's sub-problems / thesis (a specialist). Sector fit *sorts between the
  two categories* (thesis earns the Next-Round slot) but **never gates inclusion.**
- **Stage** — writes at the deal's stage (a pre-seed / seed check-writer), regardless of sector. A
  **generalist-at-stage is fully valid** (Step 5.5).
- **Geo** — invests in the deal's geography (a London / EMEA deal wants UK / EU investors), regardless
  of sector or thesis. On AgentBay this axis returned the tightest cache matches and prior history
  surfaced none of them.

Relationship is **orthogonal** to all three and never moves a score (above); cache membership already means
reachable (Step 5). The comprehensive
sweep (Step 5) runs a query per axis precisely so no single-axis fit is missed.

---

## Step 0 — Read the past rounds first (this is the feedback loop)

Every prior finalized round is a training example. Read both halves before any search — skipping this
means re-deriving a roster Tom already curated and re-pitching names that already said no.

**The two halves — Notion holds the ground truth, the ledger holds the interpretation:**

**1. Notion is the live outcome log — query it, never trust a copy.** The intro/outreach pipeline
maintains these relations automatically as outreach lands, so they're always current:
- `👓 Intros (Qualified)` on an Opportunity = the **locked, founder-approved roster** for that round.
- `✉️ Intros (Made)` = who got **connected to the founder** (a yes / engaged).
- `🚫 Intros (Declined / NR)` = who **passed or didn't respond**.

Query these on the current company's Opp and on the closest comparable companies' Opps – as RECALL (every name must be
judged) and as history for Tom; never as a score.

**2. `references/finalization-log.md` is the interpretation layer** — the things a flat relation can't
hold: which **category** each name went in, **why** Tom cut someone, hard **mandate lines** ("Rapha —
no consumer"), whether an absence was **founder-excluded vs Tom-cut vs not-considered**, and the
**cross-deal lesson** ("at homeownership pre-seed, angels convert and funds don't"). Read the block
whose deal shape best matches the current one (stage + sector + sub-problems, not category label).

**Pair them to seed this run:**
- **Same company, new round** → the `Qualified` roster is your working draft; only search the delta.
- **Comparable shape** → those names are recall: make sure each is judged (on the list or in `dropped` with the
  reason). Their live Notion outcome is history, attached after scoring – a `Declined / NR` name is stated as such in
  its `why`, never re-pitched as fresh; a name the ledger records as cut or mandate-excluded goes in `dropped` with the
  reason (a mandate line is fit – put standing mandates in the `--context` too).

**Not every logged round is equal-signal — weight by posture, which the ledger records:**
- **Tom-led rounds with a genuine gap** (Fair, Oun, Rengo, AgentBay) are the highest-signal training
  examples — the roster and outcomes reflect *Tom's own* coinvestor picks.
- **Follows, not leads** (Signal7 behind Fika, Factir behind Outsiders) — the intro relations map the
  *lead's* syndicate, not Tom's picks. Low-signal; weight down.
- **Fully-subscribed rounds Tom rallied for the founder** (Quiet Software, and Tuor's tiny gap) — the
  intros are operator-angel / design-partner support, not round-filling. Read them for the
  angels-and-operators pattern, not for follower-fund behavior.

The ledger + Notion supersede the KB's older "Prior-Round Outcomes" / "Category Calibration" prose for
anything they cover; those blocks remain for names the loop hasn't reached yet.

---

## Step 1 — Establish the real gap

Do NOT take round size from the Opportunity's `Round Details` field alone — it goes stale. Read both:

1. The Notion Opportunity row (`Fund`, `Stage`, `Inv @ Round`, `OS% @ Round`, `Round Details`, `Description`, HQ)
2. The **finalized diligence doc** in `✍️ Notes` — grep for `committed`, `allocation`, `other commitments`, `LOI`, `angel`. Actual round composition lives here, nowhere else.

Build the capital stack explicitly:

| Line | Source |
|---|---|
| Tom's check | `Inv @ Round` |
| Other committed capital | diligence doc — F&F, RUVs, prior SAFEs w/ MFN, signed LOIs |
| Contested / droppable commitments | LOIs Tom or the founder dislike |
| **Genuine remaining capacity** | round size − committed |

**The gap is almost always much smaller than the headline round.** Fair (2026-08) looked like "$2M
round, Tom's in for $1M, so $1M to fill" — the real gap was $380–580k once F&F, prior MFN SAFEs, and
a contested split-SAFE LOI were counted. That's 2–3 checks, not twelve.

State the number before naming anyone. **The gap sizes the ask, not the candidate list** — Tom trims
the list himself (Step 8), so present every plausible name and let the gap inform the *sequencing*
advice rather than the length of the list.

Also extract: stage, sector, the deal's **sub-problems** (not its category label), and the diligence
doc's **named binary risk** — you'll search on that in Step 5.

---

## Step 2 — Prior-round investor CRMs (highest-value source)

Tom hand-builds an investor CRM per portfolio company in Drive. **These are the only source with
actual outcomes.** Always check them before recommending anyone.

```
Drive search_files: title contains 'Investor' and mimeType = 'application/vnd.google-apps.spreadsheet'
Drive search_files: (fullText contains 'investor pipeline' or fullText contains 'investor CRM' …)
```

Known trackers are listed at the top of `references/coinvestor-kb.md`. Read the one for the **closest
comparable deal** — same stage, same sector, ideally same founder profile.

What to extract:
- **Committed / closed** — proven yes on this deal shape. Weight heaviest.
- **Passed**, and the stated reason. `"Too early – pre-seed is non-core"` is the single most common
  rejection and it clusters hard on seed funds. Ghosted is a stronger negative than a clean pass.
- **`0 - Avoid (Conflict)`** — competing portfolio position. Deal-specific, but check before recommending.
- **`RELATIONSHIP STRENGTH (Tom Seo)`** — Tom's own Very Strong / Strong / Medium / Weak grades across
  ~110 firms. This is the authoritative relationship field; it beats any inference.
- **Angel benches** — several trackers carry curated sector-angel lists *with Tom's own check-size
  estimates*. Directly reusable.

⚠️ A prior pass is **not** a permanent no, and Tom will override it. But never present a name as a
fresh idea when he already pitched them on the nearest comparable deal and got a no. Surface the
outcome inline.

⚠️ These CRMs go stale on employer (they're point-in-time). The People DB wins on current firm.

---

## Step 3 — Intros Tom actually made (history – attached after scoring, never a score)

The People DB has an `Intros (Made)` relation, mirrored on Opportunities as `✉️ Intros (Made)` with
`🚫 Intros (Declined / NR)` as the negative. Count = how often Tom has spent social capital putting
this person in front of a founder.

```sql
SELECT Name, Company, Role, Email, LI, City, "Intros (Made)"
FROM "collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9"
WHERE Category = 'Investor' AND "Intros (Made)" IS NOT NULL
```

The `Intro'd Count` rollup is **not** SQL-queryable — count the returned relation array.

The scan attaches this automatically (`intro_history.py`, synced from these relations): 🔁 Intro'd Before vs
🆕 Never Intro'd, and a tie-break inside a score. Use the query here only as RECALL – an investor Tom has intro'd who
isn't on the list must be judged (the judge already reads intro'd people with no card from their People row).

Also pull `🚫 Intros (Declined / NR)` when you need the negative — who has turned down Tom's intros.

---

## Step 4 — Warm lists and the co-investment graph

Two cheap, high-yield sources that semantic search will never surface:

**The "friends of the firm" list.** Tom Bccs a curated list on every quarterly letter / deck. It's a
hand-maintained warm list.
```
Gmail search_threads: from:tom@invertedcap.com subject:"friends of the firm"
```
Pull the Bcc roster, filter to `Category = 'Investor'` in the People DB.

**The co-investment graph.** Who shares cap tables and portfolio syncs with Tom — the strongest
possible "will do a deal together" evidence.
- Opportunity `Coinvestors` relation
- Recurring portfolio syncs: `Gmail search_threads: subject:"Recurring:" subject:Inverted`
- Investor-update threads (the To/Cc line *is* the cap table)

---

## Step 4.5 — Judge every person card against the round (the primary pass)

```bash
S=~/.claude/skills/network-scan/scripts/scan.py
python3 $S judge --kind investor --ask "<the round profile, e.g. 'pre-seed follower checks of $50-500k, or seed funds
    that might come in early, for a $2M pre-seed in consumer property-tax appeals'  |  Mode B: 'seed leads for a $4M
    seed in vendor-risk software for banks'>" --context "<what the company does now + stage, sector sub-problems, geo,
    the named binary risk, standing mandate lines>" --exclude-firm "<the company itself>"
```

Never `--exclude-firm-types VC` here. The investor hint (`portco_ask.KIND_HINTS["investor"]`) tells the judge what a
10 is (writes at this stage / sector / geo / check size, led comparable rounds); the judge runs at its configured tiers.
Every ≥8 keeper must be in `people` or `dropped` (`network-scan/SKILL.md` Step 4).

## Step 5 — Network cache sweep (recall backstop)

Search for names the judge and the curated sources missed.

**This step is BROAD RECALL.** The judge already sends every ≥8 fit and holds the 6–7s for "more", so the list is long
by construction – Tom curates by subtraction (narrowing to a handful was *the* failure on AgentBay and Factir). Anyone
the sweep surfaces that the judge did NOT keep: read their card (`person_cards.py card <url>`) and add them only on a
real fit. Cut a name only for the wrong **type** — not a check-writer / can't lead at the target stage (operators,
CEOs, bankers, growth-PE) — **never on your own taste**; a cut ≥8 goes in `dropped` with the reason.

**Every name in the cache is reachable — it is Tom's own network, and he has a path in to anyone in
it.** "Cold" (no Gmail / relationship trace) means *no warm relationship yet* — never *unreachable*,
never *risky*, never *verify-before-use*. It does not demote a name, drop it below the line, or earn a
hedging caveat. Relationship never moves a score either way; cold cache candidates are first-class names, not a
lesser tier. (The one real
caveat is firm *freshness* — the cache can be months stale, so confirm current firm before a name goes
in front of a founder. That's an accuracy check, not a coldness demerit.)

Run `vsearch` once per **sub-problem**, not once per category label. A property-tax-appeal company
for homeowners isn't "proptech investor" — it's consumer fintech, insurtech, home finance/mortgage,
benefits access, and consumer acquisition economics. Five queries, five different result sets.

```bash
cd ~/.claude/scripts
python3 network_cache.py vsearch "<natural language sub-problem query>" --limit 18
```

Compact the JSON when scanning many queries:
```bash
python3 network_cache.py vsearch "$q" --limit 18 2>/dev/null | python3 -c "
import json,sys
for r in json.load(sys.stdin):
    c=(r.get('context_blob','') or '').replace(chr(10),' ')
    print(f\"  {r['distance']:.3f} | {r['name'][:28]:28s} | {c[:105]}\")
"
```

Distances under ~1.05 are meaningful; past ~1.12 it's noise.

Use `csearch` when the discriminating signal is on the **company** someone worked at (e.g. "operators
from Zillow / Opendoor / Hippo"):
```bash
python3 network_cache.py csearch "<query>" --no-vc --per-company 8
```

**Always run one query framed on the deal's named binary risk**, not just its sector. If diligence
says distribution/CAC is the risk, search for consumer-growth operators — that angel is worth more
than a generic sector-fit fund.

Name-variant gotcha: FTS is exact-token. `query "Mike Barbosa"` misses "Michael Barbosa". Run both,
or pull the LI URL from the People DB and `dump <url>`.

**Breadth is the whole point of this step — err toward more queries.** A missed angle is a missed
name, and the cache is the one place a non-obvious fit surfaces.

**The comprehensive sweep is the DEFAULT, not an optional deep mode — run it every time, before you
lean on the KB/corpus.** A rich prior-history match is never a reason to skip or shrink it; history
tells you who already converted, the sweep tells you who you're missing. At minimum, one vsearch per:

- each **sub-problem** in the deal (decompose the *product*, never search the category label)
- each **adjacent sector**
- the **named binary risk**, and the **incumbent / platform** that could own the layer natively
- the deal's **geography** when it isn't default-US — a London deal wants a UK/EMEA query. *(On
  AgentBay this lane returned the tightest matches in the entire run, 0.90–0.98, and nothing in prior
  history surfaced them — under-sweeping hides exactly the names Tom most wants.)*
- the **founder-lineage companies** via `csearch --no-vc` — operator-angels who share the founders'
  pedigree (corpus shows these convert on fintech-infra deals)
- **generalist-at-stage** (Step 5.5's principle — generalists who write at this stage, no sector fit)

**As code (2026-10-04) — the pool assembly is `coinvestor_pool.py`; the queries stay yours, the scoring is the judge's.**
Write the 10–14 queries to a file (one per line; `c: <query>` = csearch `--no-vc` lineage/operator lane), then
`python3 coinvestor_pool.py --queries q.txt --opp-id <Opp id> --company "<Co>"` → `~/.claude/data/coinvestor_pool/<co>/pool.json`
+ `pool.json.queries.log` (the exact commands — paste the query list into the output from it). It runs every query
read-only, merges by best distance per person, applies the ~1.12 noise cutoff (meaningful < 1.05 / borderline
tagged), cuts ONLY wrong-type People-DB `Category` Services/Startup (csearch operator lane + Tom-calibrated names
exempt; every cut listed with its reason), diffs People DB `Category='Investor'` (live read-only query) vs the KB
with the "recently added — not yet behavior-tested" flag (Step 5.5), overlays Tom's Category Calibration +
founder-scoped exclusions + firm-dedup notes (Step 7), and groups by firm with every contact nested (Step 8).
It never ranks or fit-cuts — firms are ordered by distance only; its pool is recall for the judge's list, never an output. Exit 0 = pool written (a `⚠ 0 hits` line names
dead queries — rephrase or say why); **1 = refused: fewer than 10 queries without `--narrow-reason "<why>"`
(an under-sweep must be visible), or a query failed — fix and re-run, never proceed on a partial pool**; 2 = bad
input. Harness: `tests/test_coinvestor_pool.py` (real AgentBay sweep captured 2026-10-04 + the real KB).

That's **~10–14 queries, not 3–4.** Merge results deduped by best distance per name before reading cards.
**List the exact queries you ran in the output** so coverage is auditable — an under-sweep must be
visible, never silent. If you genuinely narrow the sweep, say why.

⚠️ **This step's semantic queries surface sector *specialists* — they will NOT surface generalists who
simply write at this stage.** Those are equally valid coinvestors (a warm generalist beats a cold
thesis-match for a Follower Check). The generalist-at-stage lane is Step 5.5's People-DB sweep, gated
on stage + relationship. The two lanes together are what "broad" means — never let the sector framing
here narrow the final list to specialists.

---

## Step 5.5 — Freshness sweep: every investor in the People DB, not just the KB

**The KB is a curated annotation layer (~30 investors with Tom's notes) — it is NOT the candidate
universe.** The live universe is every `Category = 'Investor'` row in the People DB, and it grows
every time Tom saves a new investor contact (via `add-to-contacts`). A freshly-added investor with no
intro history and not yet in the quarterly cache is invisible to Steps 2–5 — **this step is what keeps
the list current and catches them.**

```sql
SELECT Name, Company, Role, Email, LI, City, Created
FROM "collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9"
WHERE Category = 'Investor'
ORDER BY Created DESC
```

Two passes over the result:

1. **Diff against the KB.** Any People-DB investor **not already in the KB** is an un-annotated
   candidate. Screen on **any fit axis — sector, stage, or geo (see the fit model above) — not sector
   alone.** The gate is: *does this investor match the deal on stage, geo, or sector — and are they a
   check-writer?* Matching **one axis is enough**: a generalist who writes at this stage, or a UK/EU
   fund on a London deal, both qualify even with zero sector fit. (Letting a sector screen thin the list
   toward specialists is *the* failure mode of this skill.) Sector fit is a **bonus that earns the Next-Round
   slot** — it sorts between categories, never gates inclusion. Newly-saved contacts land here first —
   this is how a contact Tom added last week shows up in this week's recommendation.
2. **Recency highlight.** Reading top-of-list (newest `Created`), explicitly flag investors added
   recently as **"recently added — not yet behavior-tested"** so Tom can calibrate: they have no
   corpus/CRM/intro history yet, so relevance is thesis-and-relationship only until they earn a track
   record through the loop.

Relevance still gates — don't dump the entire investor table — but relevance means **writes at this
stage + has a relationship signal**, not sector match. Absence from the KB, the cache, or the corpus
is never grounds to drop a People-DB investor who fits. When Tom confirms one in Step 8, it earns a
full KB entry and becomes annotated for next time: **People DB in → KB annotation out.**

---

## Step 6 — Resolve identity + current firm, then read the relationship

**People DB is the system of record** for Name / Company / Role / Email / LI / City / Category
(`collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`).

- **Never quote a truncated LinkedIn URL.** The KB stores them elided (`linkedin.com/in/...umeano`).
  Take the full URL from the People DB.
- **Reconcile conflicts out loud.** When the KB, a Drive CRM, the cache headline, and the People DB
  disagree on someone's firm, say so. Job changes are the most common source of an embarrassing
  recommendation. The Drive CRMs are the stalest; People DB wins.
- **Multi-affiliation investors:** the `Company` field names one hat, not necessarily the vehicle
  they'd write from. Helen Min reads True Ventures but writes from Articulate Capital, her own fund.
  Verify or ask before putting it in front of a founder.
- **Duplicate rows** — same name, different person (two Patrick Burns; one at Collaborative Fund /
  Spruce, one a PM at MaintainX). Disambiguate on email domain and LI slug.
- **Category matters.** `Services` or `Startup` usually means no longer a check-writer.
- Absent from the People DB = cache-only connection, but **still in Tom's network — he has a path in.**
  Mark it "no warm relationship yet," never demote, deprioritize, or caveat it as risky on that basis
  (see the Step 5 rule). Reachability is assumed for anyone in the cache.

**Then read the relationship — Gmail is evidence for Tom, never a score.**

```
Gmail search_threads: {email1 OR email2 OR ...} newer_than:500d
```

Use exact addresses; loose name queries return the wrong Lucas and the wrong Patrick. Read for tone:
recurring invites, two-way dealflow, and banter are strong signals – a fact for Tom when he locks the list (Step 8),
never a reason to add, drop or reorder a scored row.

⚠️ **Silence is not coldness.** Plenty of Tom's real relationships run over text, Signal, in person,
or conference circuits and leave no Gmail trace. On 2026-08-25 Lindsay Fitzgerald and Jordan Wan both
returned zero threads in 400 days and Tom put both firmly in Category 1. Never drop or demote a name
on email silence alone — surface it with the gap noted and let Tom calibrate.

---

## Step 7 — Overlay the KB (fit facts → ask / why; constraints → dropped)

Read `references/coinvestor-kb.md` last. It supplies what nothing else does: **check size range,
stage focus, co-lead appetite, Tom's category calibration, and his explicit notes.**

The KB **annotates a subset** of the People-DB investor universe (Step 5.5), not the universe itself.
A name's absence from the KB means *un-annotated* — never *disqualified*. Pull its check size and
stage from fund profile and mark it an estimate.

Check size, stage focus and co-lead appetite are FIT – they belong in the verified `why` (and, for standing
patterns, in the judge `--context`). Honor its hard constraints:
- **Tom's Category Calibration block** — his assignments drive the categories when he locks the list (Step 8).
- **Firm dedup** — the scan already shows one row per VC firm, colleagues folded as "+N more at <Firm>" (the KB's
  multi-contact firms: QED three, Fika four, Torch three…).
- **Explicit exclusions** and anyone Tom has passed on → `dropped` with the reason.
- **Vertical-fund mandates are hard constraints**, unlike thesis preferences. Keith Bender is a
  generalist despite "Vocation Capital"; Owen Willis at Opal genuinely cannot do a non-healthcare
  deal. Different things — don't collapse them.

For names not in the KB, estimate check size from fund stage and role and **say it's an estimate**.
Don't invent co-lead appetite — "Unknown" is honest.

**The two categories, in Tom's own words – assigned when Tom locks the list (Step 8), never in the scan output.**

- **Follower Checks** — "friendly follower checks who can write low to mid-6 figure checks at
  pre-seed." **Check capability at the stage is the fit; sector fit is close to irrelevant here** – put "a generalist
  who writes pre-seed checks counts" in the Mode-A ask so the judge doesn't pull the list toward thesis-matching funds
  (*the* failure mode of this skill). Relationship is Tom's call at Step 8.
- **Next-Round Investors – who may opportunistically invest at Pre-Seed** — "likely lead candidates
  for the next round (seed), but might opportunistically be able to come in at pre-seed." Thesis fit
  earns the slot here.

**Categories are not mutually exclusive — some funds straddle, and that's legitimate, not an error.**
A seed-core fund that will opportunistically do pre-seed can be *both* a friendly Follower Check now
*and* a plausible Next-Round lead (Vivek Krishnamurthy / Commerce is the standing straddle; the KB
tags straddlers explicitly). When a name genuinely fits both, dual-list it with a "straddles Cat 1/2"
note, or surface it and ask Tom which bucket for *this* deal — never force a straddler into one.

**Soft reconciliation before output — honor Tom's explicit calls, don't build a one-bucket gate.**
Check the draft against the Category Calibration block. The only hard rule is: **do not *contradict* an
explicit Tom call** — a Tom-confirmed lead-caliber name (Samit Kalra, Nick Chirls) filed as *only* a
follower, with its Cat-2 standing silently dropped, is the error. A *straddle* is not a contradiction;
a name Tom never categorized is free to go anywhere the evidence points. Surface, don't suppress.

**Angels — a third section, only when the deal warrants it.** Five-figure to low-six-figure checks
justified by domain value-add rather than capital. Add it when the prior-round evidence says angels
convert (they did on Oun Homes, where every fund passed and both closed checks were angels), or when
a specific operator addresses the deal's named binary risk. Skip it when there's no such case — don't
add an angel section by default.

---

## Step 8 — Render, get Tom's calls, persist them

**The deliverable is the rendered Investor Scan (Step 4.5 → `scan.py render --kind investor`), pasted verbatim.**
Everything in Steps 0–7 feeds it: fit facts into the ask and the `why`s, recall into `people`, constraints and stale
names into `dropped`.

```bash
python3 $S render --scan <id> --target "<short plural target, e.g. coinvestors for Fair's pre-seed | seed leads>" \
    --for "<company>" [--founder "<founder>"] <<'J'
{"people": [{"name": "...", "role": "...", "company": "...", "linkedin": "https://www.linkedin.com/in/...",
             "why": "<fit fact: stage / check size / comparable round>. <optional history fact: Passed on Fair, Aug 2026>"}],
 "dropped": [{"linkedin": "...", "why": "..."}]}
J
```

The list is long by construction (every ≥8 sent, 6–7s on "more") – Tom curates by subtraction; never pre-filter to
the gap size. Keep each `why` to one line: the fit fact first, then any prior-round outcome as a fact. Mark doubt in the
`why` (firm conflict, mandate, a straddle between categories) rather than dropping it. After the pasted text, a short
note is fine: the stated gap (Step 1), the exact sweep queries (Step 5), and the composition advice below.

Then be opinionated about composition, not just the list:
- Seed funds taking small pre-seed positions buy cheap options and create negative signal if they
  pass at the seed. Prefer funds that genuinely write pre-seed.
- **Angels convert better than funds at pre-seed.** On the closest comparable round in the record
  (Oun Homes), every institutional fund passed or stalled and the only two closed checks were angels.
- A high-signal skeptic is often worth more as a diligence call than a check. Say that instead of
  padding the list.
- Direct competitors the search surfaces are a competitive-intel side note, never a recommendation.

**Then Tom assigns categories, and you persist his calls — this is the write side of the feedback
loop, and it is not optional.** He will move names between buckets, add people the search missed, and
keep names the data argued against. The moment he locks a list, do all of the following in the same step:

1. **Persist the roster to the Investor CRM sheet — NOT Notion yet** (Tom 2026-10-13: "Once we get two way aligned
   on who to go after then you add to qualified"). Tom's locked list is the company's Investor CRM sheet
   (investor-crm skill). Write `👓 Intros (Qualified)` on the Opportunity only once Tom and the founder are
   two-way aligned on who to go after — and only when Tom says so (the trigger for that is still TBD). Outcomes
   (`✉️ Intros (Made)` / `🚫 Intros (Declined / NR)`) accrue from the intro pipeline as before.
2. **Append a block to `references/finalization-log.md`** — the interpretation layer, per the schema at
   the top of that file: **only** what Notion can't hold — the category each name went in, who Tom
   *cut* and why, founder-excluded vs Tom-cut, any category moves, mandate lines learned, and the
   one-line lesson for the next comparable deal. **Do not re-list the roster or outcomes** — those are
   the live Notion query. Newest block at the top.
3. **Mirror category moves only** into the KB's **Tom's Category Calibration** block, dated — that block
   is the standing default assignment; the ledger is the per-round record. New investors still get a
   full KB entry (identity, check size, relationship evidence); the ledger references them by name.

Only Tom-confirmed names ship to a founder.

---

## Step 9 — Draft the email (only when asked)

Route to `writing-style/portco-investor-list/` — the stylebook for this exact form. Bare artifact:
three headed sections, `Name @ Firm` bullets, name→LinkedIn / firm→site, **no greeting, closer, or
signature**. Verify every firm URL first.

Display the draft and save via `Gmail create_draft` in the same step — don't ask permission. Never send.

---

## Mode B — Next-Round Leads (deltas from Mode A)

Everything in Steps 0–9 applies **except** the overrides below. Same loop, same judge and Investor Scan output, same
three-axis fit model, same comprehensive sweep, same cache-reachability, same persist-on-finalize discipline —
**different ask and different corpus.**

**The deliverable.** An Investor Scan of who could **LEAD the company's next round** (seed or
Series A), to put in front of the founder. Not "who fills Tom's round" — "who leads the *next* one."
The organizing axis is **who leads at the target stage** – the judge's ask.

**Output shape — the same Investor Scan as Mode A** (`scan.py render --kind investor`, verbatim; Tom 2026-10-11
"Network-scan layout for all" – this replaced the flat, alphabetical by-firm list). Rows are in blind fit order, one row
per VC firm; the Dash TIER and relationship grades are history – never a rank, a section header, or a score; the stage
they lead (Seed vs Series A) and the fit rationale go in the `why`.

**Fit model, Mode-B reading of the stage axis.** Same three axes (sector / stage / geo), but here
**stage = "leads at the target next-round stage"** — a Seed-lead fund for a company raising seed, a
Series-A lead for one raising A. A fund that only writes *small* checks is **not** a next-round lead
even if it's a great pre-seed follower — that's the Mode-A bucket. Sector and geo still sort, never gate.

⚠️ **Large multistage funds now lead seed as a core strategy — include them.** a16z, Sequoia, General
Catalyst, Lightspeed, Bain Capital Ventures, Ribbit, Founders Fund, Greylock, Index, Kleiner, Redpoint,
Thrive, etc. run dedicated seed programs and will lead a seed. **Do NOT exclude a big fund because
legacy/stale data (the Dash KB, older CRMs) tags it "Series A"** — the test is whether the fund will
*lead at the target stage today*, and for these funds seed is in-scope. This is a common miss: the
seed-specialist boutiques surface easily, the multistage funds get wrongly filtered on stale stage
tags. Surface both, and pull the named fintech partners at the big funds (e.g. a16z's fintech GPs) from
the KB even when their row reads "Series A."

**Step 1 changes — size the *need*, not the gap.** Instead of remaining capacity in Tom's round,
establish what the company needs in a next-round *lead*: target stage (seed vs A), round/check size,
board-seat expectation, and the sector/geo the lead must be comfortable with. That framing is the
judge's `--ask` / `--context`.

**Sources (recall + facts for the `why`; none of them scores):**
- **Next-round KB** (`references/next-round-kb.md`) — the master tiered lead list, the Mode-B analog of
  the coinvestor KB: who leads at what stage, Tom's relationship-strength grade, tier, and LP /
  relationship notes. ⚠️ **Seeded from 2023–24 Dash Seed/Series-A CRMs — every FUND and TITLE is
  point-in-time and STALE. Defer to the People DB for current firm (Step 6) before any name ships.**
- **The LP-graph warm-lead signal is DASH-ONLY — it does not transfer to Inverted.** The strongest
  recurring signal in the Dash CRMs is "*X is a Dash LP*," because many Dash LPs were themselves VCs
  (a16z, QED, Nyca, Bain partners…), so LP status doubled as a warm next-round lead. **Inverted
  deliberately has no VC LPs**, so this engine is **absent by design** for Inverted companies. Treat
  every Dash "X is a Dash LP" note purely as relationship *history* (Tom knows them), never as an
  Inverted warm-lead flag. For an Inverted company the warm signal comes from **Tom's own
  relationship-strength grades, intros actually made (Step 3), and the co-investment graph (Step 4)** —
  history for Tom, attached after scoring as in Mode A.
- **Next-round ledger** (`references/next-round-log.md`) — per-company finalized lead lists +
  reconciled outcomes (took the intro / led / passed), read at Step 0, appended on finalize.
  **Inverted starts empty and builds through the loop.** The 10 Dash CRMs are the seed/method
  reference, not Inverted outcomes.
- Then the shared discovery — comprehensive cache sweep (Step 5) and People DB freshness (Step 5.5),
  gated on the three-axis fit with the Mode-B stage reading; plus Notion intro relations (Step 3) and
  Drive CRMs (Step 2) exactly as in Mode A.

**Persistence (Step 8), Mode B.** Same discipline, different files: roster → the Investor CRM sheet (Qualified
only after two-way alignment, per Step 8.1); interpretation block → `references/next-round-log.md` (**not** the coinvestor
ledger); standing tier/stage calibration → `references/next-round-kb.md`.

**Email (Step 9), Mode B.** Same bare-artifact discipline: the names Tom locked, no tier headers;
name→LinkedIn, firm→site. **Verify every current firm first** — the seed data is stale.

---

## Honest signals

- Check sizes are estimates unless Tom set them. Flag inferred ranges.
- Co-lead appetite is usually genuinely unknown. Don't manufacture conviction.
- Cache enrichment can be months stale (`network_cache.py stats`). Verify current firm for any name
  going in front of a founder.
- If no strong sector fit exists, say so rather than stretching a generalist into a thesis.
- Sources are exhausted at roughly: KB, cache, letter list, co-investment graph, Drive CRMs, Intros
  relation. When those are mined, say the list is as good as the records allow instead of padding.
