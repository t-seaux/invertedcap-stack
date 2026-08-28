---
name: coinvestor-recommender
description: >
  Produce a list of investor candidates to fill out a round Tom is leading, across two categories — friendly follower checks, and likely next-round leads who might opportunistically write a smaller check now (plus angels when the deal warrants it). Ranks on evidence of who actually says yes: mines prior-round investor CRMs in Drive, the Notion Intros relation, the co-investment graph, the network cache (~5,800 profiles), and the People DB — then confirms categories with Tom and persists his calls. Trigger when Tom says "recommend coinvestors for [company]", "who should I bring into [company]", "who could co-invest in [deal]", "who should come into the cap table", "build out the syndicate for [company]", "find coinvestors for [deal]", "who could fill out the round", "help [founder] fill the rest of the round", "suggest investors to bring in", or any variant where Tom is looking to identify investors to co-invest alongside him. Also trigger when Tom describes a round structure (check size, round size, stage) and asks who in his network would fit.
---

# Coinvestor Recommender

Find investors to fill out a round Tom is leading.

**The output is a list of investors across two categories. These headers are canonical — use them
verbatim, in the research output and in the founder-facing email:**

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
| 1 | **Outcome on a comparable round** — committed / passed / ghosted | Drive investor CRMs (Step 2) |
| 2 | **Intros Tom actually made** — revealed preference | Notion `Intros (Made)` (Step 3) |
| 3 | **Co-investment** — shares a cap table or portfolio sync | Opportunity `Coinvestors`, portfolio syncs (Step 4) |
| 4 | **Tom's own hand-graded relationship strength** | Asset CRM `RELATIONSHIP STRENGTH (Tom Seo)` column |
| 5 | **Curated warm lists** — "friends of the firm" letter Bcc | Gmail sent mail (Step 4) |
| 6 | **Recurring 1:1 cadence / reciprocal dealflow** | Gmail + Calendar (Step 6) |
| 7 | **Thesis fit** | Network cache (Step 5) |

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
- Absent from the People DB = cache-only LinkedIn connection. Treat as cold.

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

**Then expect Tom to reassign categories, and persist his calls.** He will move names between
buckets, add people the search missed, and keep names the data argued against. Write every decision
into the KB's **Tom's Category Calibration** block with the date, so the next run starts from his
assignments instead of re-deriving them. New names get a full KB entry with relationship evidence.

Only Tom-confirmed names ship to a founder.

---

## Step 9 — Draft the email (only when asked)

Route to `writing-style/portco-investor-list/` — the stylebook for this exact form. Bare artifact:
three headed sections, `Name @ Firm` bullets, name→LinkedIn / firm→site, **no greeting, closer, or
signature**. Verify every firm URL first.

Display the draft and save via `Gmail create_draft` in the same step — don't ask permission. Never send.

---

## Honest signals

- Check sizes are estimates unless Tom set them. Flag inferred ranges.
- Co-lead appetite is usually genuinely unknown. Don't manufacture conviction.
- Cache enrichment can be months stale (`network_cache.py stats`). Verify current firm for any name
  going in front of a founder.
- If no strong sector fit exists, say so rather than stretching a generalist into a thesis.
- Sources are exhausted at roughly: KB, cache, letter list, co-investment graph, Drive CRMs, Intros
  relation. When those are mined, say the list is as good as the records allow instead of padding.
