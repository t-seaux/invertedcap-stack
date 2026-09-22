---
name: coinvestor-recommender
description: |-
  Produce ranked investor candidates to fill out a round Tom is leading — friendly follower checks + likely next-round leads who might write a smaller check now (plus angels when the deal warrants); evidence sources, ranking, and persistence live in the skill body. Trigger: "recommend coinvestors for [company]", "who should I bring into [company]", "who could co-invest in [deal]", "who should come into the cap table", "build out the syndicate for [company]", "find coinvestors for [deal]", "who could fill out the round", "help [founder] fill the rest of the round", "suggest investors to bring in", or Tom describes a round structure (check size, round size, stage) and asks who in his network fits. ALSO Mode B — Next-Round Leads: who should LEAD a portfolio company's NEXT round (seed / Series A); trigger on "who should lead [company]'s next round", "next-round investors for [company]", "who leads [company]'s seed / Series A", "who do we take [company] to for their raise", "recommend next-round leads for [company]".

---

# Investor Recommender

This skill runs in **two modes.** Detect which from Tom's ask; **default to Mode A.**

- **Mode A — Coinvestors (default):** who fills the **pre-seed round Tom is leading now** — friendly
  follower checks + next-round funds who might opportunistically come in early. Triggers: "recommend
  coinvestors for X", "who should I bring into X", "fill / build out the round".
- **Mode B — Next-Round Leads:** who should **LEAD a portfolio company's NEXT round** (seed / Series A)
  — connecting the founder to the right next-stage lead. Triggers: "who should lead X's next round",
  "next-round investors for X", "who leads X's seed / Series A", "who do we take X to for their raise".
  See **## Mode B — Next-Round Leads** near the bottom for its deltas.

**Almost everything is shared.** The fit model, Step 0 loop, the comprehensive sweep (Step 5), the
People DB freshness sweep (Step 5.5), identity/relationship gating (Step 6), cache-reachability, and
the persist-on-finalize loop (Step 8) are **identical across modes.** Steps 0–9 below are written for
Mode A; **Mode B changes only the deliverable, the Step 1 framing, the primary sources, the category
buckets, and the corpus files — all specified in ## Mode B.** Read that section first when in Mode B,
then run the shared steps with its overrides in mind.

---

## Mode A — Coinvestors (default): the deliverable

Find investors to fill out a round Tom is leading. **The output is a list across two categories. These
headers are canonical — use them verbatim, in the research output and in the founder-facing email:**

1. **Follower Checks**
2. **Next-Round Investors – who may opportunistically invest at Pre-Seed**

(Plus **Angels** as a third section when the deal warrants it — see Step 7.)

Everything below exists to produce that list. Tom then reassigns names between categories, and those
calls get persisted.

## The core principle — rank on behavior, not on similarity

The instinct is to semantic-search the network for thesis fit. **That is the wrong primary signal.**
A friendly follower writes the check because they trust Tom and the round is easy — not because the
deal matches their thesis. The sources below are ordered by *evidence quality*, and the top of that
list is always behavioral: who has actually written a check, taken an intro, or shared a cap table.

The network cache is for **discovery and breadth**. It is not for ranking.

**Signal hierarchy, strongest to weakest:**

| Rank | Signal | Where it lives |
|---|---|---|
| 0 | **Tom's locked roster + what came back on a prior round** — Qualified / Made / Declined-NR relations, read with the ledger's category & cut context | Notion Opp relations + `references/finalization-log.md` (Step 0) |
| 1 | **Outcome on a comparable round** — committed / passed / ghosted | Drive investor CRMs (Step 2) |
| 2 | **Intros Tom actually made** — revealed preference | Notion `Intros (Made)` (Step 3) |
| 3 | **Co-investment** — shares a cap table or portfolio sync | Opportunity `Coinvestors`, portfolio syncs (Step 4) |
| 4 | **Tom's own hand-graded relationship strength** | Asset CRM `RELATIONSHIP STRENGTH (Tom Seo)` column |
| 5 | **Curated warm lists** — "friends of the firm" letter Bcc | Gmail sent mail (Step 4) |
| 6 | **Recurring 1:1 cadence / reciprocal dealflow** | Gmail + Calendar (Step 6) |
| 7 | **Thesis fit** | Network cache (Step 5) |

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

Relationship is **orthogonal** to all three: it *promotes* a name within Follower-Check ranking, but
its absence never *demotes*, and cache membership already means reachable (Step 5). The comprehensive
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

Query these on the current company's Opp and on the closest comparable companies' Opps. This is
signal-hierarchy rank 0 — it beats every inference from sector or warmth.

**2. `references/finalization-log.md` is the interpretation layer** — the things a flat relation can't
hold: which **category** each name went in, **why** Tom cut someone, hard **mandate lines** ("Rapha —
no consumer"), whether an absence was **founder-excluded vs Tom-cut vs not-considered**, and the
**cross-deal lesson** ("at homeownership pre-seed, angels convert and funds don't"). Read the block
whose deal shape best matches the current one (stage + sector + sub-problems, not category label).

**Pair them to seed this run:**
- **Same company, new round** → the `Qualified` roster is your working draft; only search the delta.
- **Comparable shape** → lead with those names, each carrying its live Notion outcome inline. A name in
  `Made` on a comparable shape is the best reason to lead with someone; a name in `Declined / NR` is
  surfaced with that fact, never re-pitched as fresh; a name the ledger records as cut or
  mandate-excluded starts below the line with the reason shown.

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

## Step 3 — Intros Tom actually made (best behavioral signal)

The People DB has an `Intros (Made)` relation, mirrored on Opportunities as `✉️ Intros (Made)` with
`🚫 Intros (Declined / NR)` as the negative. Count = how often Tom has spent social capital putting
this person in front of a founder.

```sql
SELECT Name, Company, Role, Email, LI, City, "Intros (Made)"
FROM "collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9"
WHERE Category = 'Investor' AND "Intros (Made)" IS NOT NULL
```

The `Intro'd Count` rollup is **not** SQL-queryable — count the returned relation array.

**Relationship strength and intro count diverge, and both matter.** Some names grade Very Strong with
one intro; others have 10. Strength measures how well Tom knows someone; intro count measures whether
he'll spend capital on them. For filling a round, the second is closer to what's being asked.

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

## Step 5 — Network cache (discovery and breadth)

Now, and only now, search for names the curated sources missed.

**This step is BROAD RECALL, and its output goes to Tom for escalation — do NOT pre-curate it.**
Surface the **full deduped investor surface** (every investor-category hit out to the noise threshold
~1.12), presented flat. The division of labor is fixed: **the sweep casts the wide net; Tom gives
feedback and escalates which names to actually reach out to.** Narrowing to the handful you judge
best-fit steals his call and buries candidates he wanted — this is the single most common failure of
this skill (it recurred on AgentBay and Factir). Cut a name only if it's the wrong **type** — not a
check-writer / can't lead at the target stage (operators, CEOs, bankers, growth-PE) — **never on your
own judgment of fit.** State how many you cut and why. Err massively long; Tom curates by subtraction.

**Every name in the cache is reachable — it is Tom's own network, and he has a path in to anyone in
it.** "Cold" (no Gmail / relationship trace) means *no warm relationship yet* — never *unreachable*,
never *risky*, never *verify-before-use*. It does not demote a name, drop it below the line, or earn a
hedging caveat. A warm relationship **promotes** within Follower-Check ranking; its absence never
**demotes**. Present cold cache candidates as first-class names, not as a lesser tier. (The one real
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

That's **~10–14 queries, not 3–4.** Merge results deduped by best distance per name before ranking.
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
   fund on a London deal, both qualify even with zero sector fit. (This mirrors Step 7: Follower Checks
   rank on relationship × check capability, and letting a sector screen thin the list toward
   specialists is *the* failure mode of this skill.) Sector fit is a **bonus that earns the Next-Round
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

## Step 6 — Resolve identity, then gate on relationship

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

**Then gate on relationship — Gmail promotes, it never demotes.**

```
Gmail search_threads: {email1 OR email2 OR ...} newer_than:500d
```

Use exact addresses; loose name queries return the wrong Lucas and the wrong Patrick. Read for tone:
recurring invites, two-way dealflow, and banter are strong signals.

⚠️ **Silence is not coldness.** Plenty of Tom's real relationships run over text, Signal, in person,
or conference circuits and leave no Gmail trace. On 2026-08-25 Lindsay Fitzgerald and Jordan Wan both
returned zero threads in 400 days and Tom put both firmly in Category 1. Never drop or demote a name
on email silence alone — surface it with the gap noted and let Tom calibrate.

---

## Step 7 — Overlay the KB, dedupe, and categorize

Read `references/coinvestor-kb.md` last. It supplies what nothing else does: **check size range,
stage focus, co-lead appetite, Tom's category calibration, and his explicit notes.**

The KB **annotates a subset** of the People-DB investor universe (Step 5.5), not the universe itself.
A name's absence from the KB means *un-annotated* — never *disqualified*. Pull its check size and
stage from fund profile and mark it an estimate.

Honor its hard constraints:
- **Tom's Category Calibration block overrides everything** — his assignments beat any inference.
- **Firm dedup** — one contact per firm unless asked otherwise. The KB lists the known multi-contact
  firms (QED has three, Fika four, Torch three…). The cache will happily return all of them.
- **Explicit exclusions** and anyone Tom has passed on.
- **Vertical-fund mandates are hard constraints**, unlike thesis preferences. Keith Bender is a
  generalist despite "Vocation Capital"; Owen Willis at Opal genuinely cannot do a non-healthcare
  deal. Different things — don't collapse them.

For names not in the KB, estimate check size from fund stage and role and **say it's an estimate**.
Don't invent co-lead appetite — "Unknown" is honest.

**The two categories, in Tom's own words. This is the deliverable.**

- **Follower Checks** — "friendly follower checks who can write low to mid-6 figure checks at
  pre-seed." Rank on **relationship × check capability. Sector fit is close to irrelevant here.** A
  warm generalist beats a cold thesis-match. Letting a sector-driven cache scan pull this list toward
  thesis-matching funds is *the* failure mode of this skill.
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

## Step 8 — Propose, get Tom's calls, persist them

**The deliverable is a list of investors across the two categories. Everything in Steps 1–7 exists
to produce that list — it is not the output.**

**Present a generous list Tom can trim from.** He curates by subtraction: he'll cut names, add ones
the search missed, and move people between categories. That means **err long, not short** — surface
every plausible candidate with its evidence and let him cut. A name you withheld because you judged
it a weak fit is a name he never got to consider; a name he doesn't want costs him one keystroke.
This overrides any instinct to pre-filter down to the gap size.

Lead with the ranked names under their category headers. Keep supporting detail to one compact line
per name: `Name / Role @ Firm` · `[LinkedIn](url)` · email · check-size expectation · **the specific
relationship evidence** ("recurring biweekly Oun Homes call" beats "Working") · any prior-round
outcome. Don't write a research report — Tom wants the list and the reasoning available, in that order.

**Group the output by FIRM, and nest the people.** One bullet per **firm**; within it, list every
person Tom knows there (`Lightspeed — Faraz Fatemi · Bucky Moore · Katherine Zhang · Mercedes Bent`),
so Tom picks the contact. This complements firm-dedup: show Tom **all** known contacts per firm here;
the founder-facing final artifact still lands one contact per firm (Step 9). Applies to the **broad
cache surface (Step 5)** too — group those hits by firm as well. (Tom, 2026-08-31)

Mark anything you're unsure about rather than dropping it: firm conflicts, prior-round passes, thin
relationships, mandate constraints. Flagged-and-included beats silently-excluded.

Then be opinionated about composition, not just the list:
- Seed funds taking small pre-seed positions buy cheap options and create negative signal if they
  pass at the seed. Prefer funds that genuinely write pre-seed.
- **Angels convert better than funds at pre-seed.** On the closest comparable round in the record
  (Oun Homes), every institutional fund passed or stalled and the only two closed checks were angels.
- A high-signal skeptic is often worth more as a diligence call than a check. Say that instead of
  padding the list.
- Direct competitors the search surfaces are a competitive-intel side note, never a recommendation.

**Then Tom reassigns categories, and you persist his calls — this is the write side of the feedback
loop, and it is not optional.** He will move names between buckets, add people the search missed, and
keep names the data argued against. The moment he locks a list, do all of the following in the same step:

1. **Persist the roster to Notion — this is the ground truth.** Write one People DB row per approved
   name into the Opportunity's `👓 Intros (Qualified)` relation; excluded names are simply absent.
   Outcomes (`✉️ Intros (Made)` / `🚫 Intros (Declined / NR)`) then accrue automatically as the
   intro/outreach pipeline runs — you don't hand-maintain them.
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

Everything in Steps 0–9 applies **except** the overrides below. Same loop, same three-axis fit model,
same comprehensive sweep, same cache-reachability, same persist-on-finalize discipline — **different
deliverable and different corpus.**

**The deliverable.** A ranked list of investors who could **LEAD the company's next round** (seed or
Series A), to put in front of the founder. Not "who fills Tom's round" — "who leads the *next* one."
The organizing axis is **who leads at the target stage**, tiered by priority.

**Output shape — a single FLAT, UNRANKED bullet list. No tiers, no hierarchy, no implied ordering
(Tom, 2026-08-31).** The candidates are **peers** — a set of qualified options for Tom to judge, not a
ranking. Any ordering reads as a priority call, and that call is Tom's, not the skill's. Present **grouped by firm (alphabetical), with the people Tom knows nested inside each firm's bullet**
(per Step 8) — `Firm — Person A · Person B` — the stage they lead (Seed vs Series A) · one-line fit
rationale · provenance · any prior-round flag (already passed / already engaged) inline. The Dash TIER
and relationship grades feed the *rationale text*, never a rank or a section header. Give Tom the full
option set; he decides priority.

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
board-seat expectation, and the sector/geo the lead must be comfortable with. That framing sets the
tier and the stage tags.

**Sources — the priority order shifts (behavior still ranks first):**
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
  exactly the behavioral hierarchy Mode A already ranks on.
- **Next-round ledger** (`references/next-round-log.md`) — per-company finalized lead lists +
  reconciled outcomes (took the intro / led / passed), read at Step 0, appended on finalize.
  **Inverted starts empty and builds through the loop.** The 10 Dash CRMs are the seed/method
  reference, not Inverted outcomes.
- Then the shared discovery — comprehensive cache sweep (Step 5) and People DB freshness (Step 5.5),
  gated on the three-axis fit with the Mode-B stage reading; plus Notion intro relations (Step 3) and
  Drive CRMs (Step 2) exactly as in Mode A.

**Persistence (Step 8), Mode B.** Same discipline, different files: roster → the Opp's `👓 Intros
(Qualified)` relation; interpretation block → `references/next-round-log.md` (**not** the coinvestor
ledger); standing tier/stage calibration → `references/next-round-kb.md`.

**Output / email (Step 9), Mode B.** Same bare-artifact discipline, but a **single flat, UNRANKED
bullet list** — no tier headers, no implied priority ordering (Tom's preference); neutral order
(alphabetical by firm); name→LinkedIn, firm→site. **Verify every current firm first** — the seed data
is stale.

---

## Honest signals

- Check sizes are estimates unless Tom set them. Flag inferred ranges.
- Co-lead appetite is usually genuinely unknown. Don't manufacture conviction.
- Cache enrichment can be months stale (`network_cache.py stats`). Verify current firm for any name
  going in front of a founder.
- If no strong sector fit exists, say so rather than stretching a generalist into a thesis.
- Sources are exhausted at roughly: KB, cache, letter list, co-investment graph, Drive CRMs, Intros
  relation. When those are mined, say the list is as good as the records allow instead of padding.
