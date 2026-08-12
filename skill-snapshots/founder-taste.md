---
name: founder-taste
description: Tom's founder-taste corpus + the synthesis entry point for questions about his own investing style. TRIGGER on questions like "what's my investing style", "what's my founder taste", "why do I pass", "why do I invest", "what do I actually look for", "how has my taste changed", "what are my blind spots" — see Query Mode below. Otherwise this is a corpus, not a workflow: do not trigger it for drafting or pipeline tasks. Single home for the doctrine and evidence of how Inverted Capital evaluates founders. RUBRIC.md is the corpus index (LPAC three-lens thesis + 6-signal rubric + 4 archetypes + scoring); SYSTEM.md holds the execution system, data model, and feedback engines; CASEBOOK.md worked examples; DECISION_RETROS.md the retro nugget log; PILLARS.md the Argument Pillars (recurring substantive arguments, currently pass-side); ONLINE_SOURCES.md the per-signal research taxonomy. The decision ledger (~/.claude/data/decision_ledger.db) is the corpus's structured spine. Consumers: neg1-enricher, founder-outreach, decision-retro(+listener), retro-weekly-summary, network-scan, draft-feedback, first-pass-diligence, pipeline-agent, pass-note-drafter. RUBRIC.md edits are human-gated — Tom's explicit call only.
---

# Founder Taste

Read `RUBRIC.md` first — its §0 corpus index maps everything else. This directory replaced `founder-outreach/FOUNDER_EVAL_FRAMEWORK.md` + the evidence files formerly in `neg1-enricher/` (2026-07-16 consolidation; stubs remain at the old paths).

For everything except Query Mode below, this is a **corpus, not a workflow** — other skills read from it; it does not run a pipeline of its own.

---

## Query Mode — "what's my investing style / founder taste?"

Tom's own taste is the one question this corpus answers directly. Trigger on any
variant: *what's my investing style*, *what's my founder taste*, *why do I pass*,
*why do I invest*, *what do I actually look for*, *what are my blind spots*,
*how has my taste changed*.

**The core move is stated vs. revealed.** `RUBRIC.md` is what Tom *says* his
framework is. The ledger, retros, and Pillars are what his decisions *show*.
Where those disagree is the most valuable output — do not smooth it over.

### Sources, and what each is good for

| Source | Read it for | Caveat |
|---|---|---|
| `RUBRIC.md` | Stated doctrine: 6 signals, 4 archetypes, LPAC three-lens thesis | Aspirational — it is the hypothesis, not the evidence |
| `~/.claude/data/decision_ledger.db` | The spine: 137 decisions (30 invested, 107 passed/no-outreach), signal scores, rubric verdict vs. override | `why` is backfilled placeholder on ~39 rows — exclude with `why NOT LIKE '%BACKFILL%'`. 29 invest rows carry a real retro. |
| `DECISION_RETROS.md` | Unfiltered reasoning, incl. cold dismissals that never got a pass note | The candid version; outranks pass notes on what he actually thought |
| `PILLARS.md` | Revealed recurring arguments, with conditions and verbatim framings | **Pass-side only.** Do not present as his whole taste — see Known gaps |
| `CASEBOOK.md` | Worked calibration cases | Curated, so it over-represents clean examples |
| `pass_note_vs_decision` view | Where the founder-facing reason differs from the private one | The candor gap |
| `decision_ledger.db` → `candidates` table + `no-outreach` decisions | **The -1 (pre-founder) corpus — the purest founder-taste signal in the system.** 48 candidates (10 being pursued: tracked / drafted / reached-out; 36 passed), with `eval_rationale` on 42 and written reasons on 23 of the 36 `no-outreach` calls. Uniquely uncontaminated: no company, market, product, or model to confound the read — the only variable is the person | Blunt reads on named individuals who never pitched him. Arguments are publishable; names and identifying detail never are |
| `writing-style/letters-and-memos/VOICE_EXAMPLES.md` | **5 LP letters** (incl. the Founding Letter, Q2–Q4 2025, Q1 2026) + **5 full investment memos** (Tuor, Signal7, Quiet AI, Rengo, Oun Homes). ~46k tokens | The most deliberate thesis articulation Tom produces. LP-facing, so audience-aware |
| Drive investment memos (`1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`) | The canonical, current memo folder — newer memos than the backfilled five above | Requires Drive access; recency-weight it |
| `~/code/invertedcap-decks/decks/lpac-july-2026/` (`CONTENT-CONTEXT.md`, `DECK-NOTES.md`) | The LPAC three-lens thesis as presented, plus the reasoning behind the framing | Most polished; a performance of the thesis, not the thesis forming |
| Notion Notes DB (`e8afa155-…`) | Pre-mortems — risk articulation at decision time | Per-deal, not thematic |

### Weight sources on two axes — do not average them

Conflicts between these sources are signal, not noise. Place each on both axes
before reconciling:

**Candor** (private → public): retros and cold dismissals → investment memos →
pass notes → LP letters → LPAC deck. The private end tells you what Tom
*believes*; the public end tells you what he has *decided to stand behind*. A
thesis that appears at both ends is load-bearing. One that appears only at the
public end is positioning; only at the private end, an unarticulated instinct.

**Timing** (decision-time → hindsight): investment memo → pre-mortem → LP letter
→ decision retro. A memo says what Tom believed when he wired the money; a retro
says what he thinks now. The gap between the two on the same company is where his
taste actually moved — the single most valuable thing in this corpus, and the
reason a retro should never be silently substituted for the memo that preceded it.

Concretely: five companies (Tuor, Signal7, Quiet AI, Rengo, Oun Homes) have
**both** a decision-time memo and a hindsight retro. Those five are the only
places the learning delta is directly observable. Read both halves before
characterizing any of them.

**Excluded sources** (Tom's call, 2026-07-27 — do not reintroduce silently):

- **"First-pass" memos as a distinct source.** `~/.claude/data/firstpass_memos/`
  is misnamed: it is the **investment-memo Drive cache**, maintained by
  `first-pass-diligence/memo_cache.py`, named for its first consumer rather than
  its contents. Verified 2026-07-27 byte-identical to the final memos. So there
  is no "first read vs. conviction" delta to derive — any such claim would be
  fabricated. The cache itself is NOT excluded: it is the freshest memo source in
  the system and is bridged into `documents` by
  `lp_letter_cache.py bridge-memos`.
- ~~**The AgentBay memo.**~~ No longer excluded — finalized 2026-08-10 (`[WIP]`
  dropped, cache synced). It is now the freshest and most framework-explicit
  taste artifact: its six thesis pillars map one-to-one onto the LPAC lens names
  (Solution Shape, Self-Reinforcing Moat, Multi-Act Sequencing, Founder Shape,
  Company-Building Style, Opportunity Cost). That makes **7 usable investment
  memos**: Oun Homes, Quiet AI, Rengo, Signal7, Tuor, Factir, AgentBay.
  (AgentBay has no hindsight retro yet — it joins the memo corpus, not the
  memo-vs-retro delta set.)

Scale the read to the question. "Why do I pass" needs Pillars plus pass rows;
"what's my taste" needs the ledger's invest rows and retros too. Don't read all
400KB for a narrow question.

### Step 0: refresh the Drive-sourced corpus (on demand — there is no scheduler)

Retros and -1 decisions reach the corpus on their own: DB triggers enqueue them
and the processor picks them up within seconds. **Drive documents do not** —
Google cannot push to a local machine without a public webhook endpoint, and LP
letters and memos change roughly monthly, so a background poll would be almost
entirely wasted work. The sync is therefore pulled by whoever needs fresh data,
which is you, here:

```bash
python3 ~/.claude/skills/draft-feedback/lp_letter_cache.py sync          # LP letters
python3 ~/Projects/invertedcap-skills/first-pass-diligence/memo_cache.py sync   # memos → files
python3 ~/.claude/skills/draft-feedback/lp_letter_cache.py bridge-memos  # memos → corpus
python3 ~/.claude/skills/draft-feedback/processor.py --sweep            # fold in now
```

Both syncs are mtime-invalidated — when nothing changed they cost a single
folder listing and report `refreshed: 0`. Skip this step only for a narrow
question that touches neither memos nor letters.

**Consequence worth knowing:** a letter written and never followed by a taste
question stays unread until the next time someone asks. That is the accepted
trade for having no background poll, not an oversight.

### Method

1. **Query the ledger first.** Counts and rates before prose — they establish
   weight and stop the answer from being anecdote-led.
2. **Separate stated from revealed**, and name the gaps explicitly. An override
   of the rubric verdict is a high-signal row: it is where taste beat framework.
   Only 6 rows carry one today, and all 6 run the same direction — *"rubric said
   reach out; Tom passed"* — which is itself a finding worth reporting.
3. **Ground every claim in a verbatim quote or a number.** Paraphrase reads as
   Claude's interpretation of Tom, which is exactly what this should not be.
4. **Check the invest side.** The Pillars corpus is pass-derived, so a naive
   read produces a purely negative portrait. Pull the 30 invest rows and their
   retros directly rather than inferring the positive case from absences.
5. **Read the -1 corpus for any founder-quality question.** Every other source
   observes a founder *through a deal*, which conflates the person with the
   company, the market, and the timing. The `candidates` table and the
   `no-outreach` decisions isolate the person — and because there is no founder
   on the other end of the note, the reasoning is blunter than anything in the
   pass-note corpus. Disqualifiers appear here that appear nowhere else
   ("too comfortable", career-stage ceilings, a named employer no longer
   reading as signal). It is also where the rubric *overrides* are explained.
6. **Check for self-similarity between the firm level and the deal level.** The
   Founding Letter runs the same arguments about Inverted that Tom runs about
   his portfolio companies: he counter-positions against "Empire Building Seed
   Funds" scaling AUM to drive down cost of capital, and treats his own high
   cost of capital as a discipline — *"removing 'strategic' distractions that
   detract from my ability to solve for absolute returns."* That is IP2 (the
   refused shortcut) and IP1/P13 (counter-positioning) applied to himself.
   When a Pillar shows up at both levels, it is doctrine, not preference — say
   so, because it is the strongest form of evidence in the corpus.
7. **Follow the synthesis shape in `feedback_synthesis_output_shape`** — abstract
   to a few load-bearing points grouped by what the argument *does*, each with
   quotes and counts; name the unifying thesis; close with one uncomfortable
   observation. Never return the raw catalog.

### Output format

Two modes. Pick by what's being asked.

**Analytical answer** (default — "why do I pass?", "what's my blind spot?"):
abstract to a few load-bearing points grouped by what the argument *does*, per
`feedback_synthesis_output_shape`. Compression is the value.

**Taste asset** ("what's my investing style?", or anything headed for an external
audience): **section header, then a dense flurry of bullets that reads as a
comprehensive list.** Here volume IS the message — the artifact exists to show
how many reps Tom has run and how opinionated he is, so exhaustiveness and sharp
declarative bullets beat elegant compression. Each bullet should stand alone as a
position, in his voice, ideally with his own phrasing embedded. Lead sections with
counts (companies, decisions, years) because the reps are part of the claim.

Canonical shape (Tom's spec, 2026-07-27):

```
Investing Style (v0.1)
Updated <YYYY-MM-DD>

<One-line thesis: "Your investing style is rooted in ___" — the single
through-line that all four sections are instances of.>

Market
* <position>
* <position>

Team
* ...

Product
* ...

Model
* ...
```

- **Audience is FOUNDERS, not LPs** (Tom's call, 2026-07-27). This is the single
  most important constraint on the artifact:
  - **Cut the firm layer entirely** — cost structure vs. cost of capital, AUM
    dynamics, Empire Builders, fund size, ownership targets, the Messy Middle,
    graduation-rate math, portfolio construction. That material is the spine of
    the LP letters and is genuinely his sharpest thinking, but a founder does not
    care how he competes with other funds. It also drags in fund economics that
    can't be published anyway.
  - Keep only the slice of market-structure view that changes what a founder
    should expect from him — e.g. that he invests before consensus forms, which
    explains why he engages years early and why he won't move in a 48-hour
    auction.
  - Write every bullet so a founder can self-assess against it.
- **Sections map 1:1 to the `family` field on each Pillar** — `market`, `team`,
  `product`, `model`, plus `fit`. Generate from the data; don't hand-assemble.
  All four core sections are about the COMPANY, which is the right frame here.
- **Model is load-bearing**, not an afterthought: operating model, delivery model
  (services vs. product), capital intensity, margin structure, how the business
  scales. Tom flagged it explicitly as something he cares a lot about.
- **Keep `fit` for a founder audience** — reframed as "when I'm not your
  investor." Against LPs it reads as disclosure; to a founder it is a genuine
  service (saves them a cycle) and it is on-brand with the intellectual honesty
  he underwrites for. State the constraints plainly: domain gaps, stage, and
  balance-sheet-dependent models.
- Version the artifact (`v0.1`, `v0.2`) and date it. It is meant to evolve as
  reps accumulate, and showing the version history is itself evidence of reps.
- Bullets should be declarative and opinionated. "I don't invest in categories
  where capital is the moat" beats "I tend to prefer less competitive markets."
- **Every bullet is an abstraction, never an instance.** State the pattern, not
  the anecdote it came from. The corpus is built out of specific people and
  specific deals, so the default failure is a bullet that reads as a story —
  *"one founder told me…"*, *"I declined to re-back a founder whose…"*, a
  gendered singular referent, a recounted meeting. Each of those fingerprints a
  real person to anyone with context, and the abstraction is stronger writing
  besides: a pattern applies to the reader, an anecdote only applies to whoever
  it happened to. Audit for first-person anecdote openers, singular pronouns,
  and recounted events before publishing.

### Publication safety — apply whenever output may go external

The corpus contains material that must not be published as-is. Strip or
generalize before any external use:

- **Never attribute a pass to a named company.** Pass-side Pillars quote real pass
  notes sent to real founders. "Why I passed on <Company>" is relationship-damaging
  and irreversible. Publish the *pattern* and the reasoning; drop the company name
  and any detail that identifies it.
- **Portfolio companies may be named** (already public) — but not their metrics,
  valuations, ownership, or check sizes.
- **Never publish unflattering characterizations of named founders.** Several
  retros contain blunt negative reads on identifiable people. These are internal
  calibration notes, full stop.
- **No fund economics** — fund size, LP names, cost of capital specifics, reserve
  ratios, MOIC/TVPI, ownership targets.
- Tom's own self-critique IS publishable and is among the most credible material
  in the corpus — admitted mistakes are the strongest proof of real reps.

When in doubt, publish the argument and cut the referent.

### Known gaps — state these when they bear on the answer

- **Pillars are pass-only.** All 19 derive from pass notes, so the revealed-taste
  layer currently models Tom by his negative space. Invest-side Pillars are not
  yet extracted; read invest retros directly instead.
- **~39 ledger rows have placeholder `why` text**, not real retros.
- **Archive send dates** in the writing-style corpus are a `2026-04-25` bulk-import
  stamp, not true dates. Company name is the reliable discriminator.
- **Outcomes are thin.** `decisions.outcome` is mostly unset, so "was I right?"
  is largely unanswerable today. Say so rather than implying calibration.
