---
name: research-artifact-audit
description: Canonical audit + iteration discipline for long-form research artifacts (diligence, memos, deep dives, comp landscapes). Callers bind file paths and judge-prompt; this skill defines the audit gate, the iteration loop, Step 3.5 partial normalization, and the hard prohibitions.
---

# Research Artifact Audit — Canonical Spec

This skill is **not invoked standalone**. It is read by other skills (e.g.,
`first-pass-diligence`) that draft long-form factual artifacts and need to
guarantee every entity-anchored or numeric claim traces to the source bundle.
It can also be applied ad-hoc when Tom asks for a memo, deep dive, comp
landscape, or briefing — see the `feedback_research_artifact_self_audit.md`
memory for trigger phrases.

The shape, rules, and prohibitions below are **the single source of truth**.
Caller skills must NOT restate any of this — they only bind the file paths
and any caller-specific wiring (Slack alert formats, source bundle section
structure, publish targets).

---

## Bindings the caller provides

A calling skill must specify these before invoking the audit flow:

- **DRAFT** — path to the markdown draft to audit
- **SOURCES** — path to the concatenated source bundle
- **AUDIT_JSON** — output path for the audit JSON
- **JUDGE_PROMPT** — path to the judge prompt file (canonical:
  `~/.claude/skills/first-pass-diligence/first_pass_audit.prompt.md`)
- **AUDIT_RUNNER** — path to the runner script (canonical:
  `~/.claude/skills/first-pass-diligence/first_pass_audit.py`)
- **MAX_ITER** — iteration cap (default 3)
- **WEB_RESEARCH_CAP** — max claims/iter resolvable via path (d) below
  (default 6)
- **ITER_SNAPSHOT_PREFIX** — path prefix for iteration snapshots, e.g.
  `/tmp/firstpass_draft.iter` (snapshots saved as `<prefix>N.md`)
- **NORMALIZED_DRAFT** — output path for Step 3.5 (e.g.
  `/tmp/firstpass_draft.normalized.md`)

Everything below operates on those bindings.

---

## Step A — Build the source bundle

The audit is **only as good as the bundle**. Concatenate every piece of source
material the drafter consumed into a single markdown file with clear section
delimiters. The exact section structure is caller-specific (diligence has
Opp/notes/memos/analogs; a market memo might have research-notes/comps/reports).
Use `==== SECTION NAME ====` as the section delimiter and `--- Item: <title> ---`
as the item delimiter.

If you omit a source the drafter actually used, the judge will flag false-positive
untraced verdicts. If the bundle includes a source the drafter never drew on,
no harm — the judge only checks claims, not coverage.

### A.0 — Verbatim mandate (MANDATORY)

**Every section housing reference-source content — call notes, backchannel
notes, third-party docs, Drive PDFs, founder memos — must contain the full
verbatim body of each cited source.** Pointer manifests, summaries, and
"see Section X for what this contains" prose are forbidden. The audit judge
compares draft claims against this bundle LITERALLY; if the cited source's
body isn't here, every claim citing it will be flagged untraced — spuriously,
since the draft is correct.

For Notion-housed sources (call notes, meeting notes): fetch with
`notion-fetch include_transcript: true` and concatenate the full body
(Notion ID anchor, page URL, AI-summary block, transcript, structured
fields). For Drive PDFs: extract the full text via pdftotext (Drive's
`read_file_content` MCP tool sometimes returns truncated output — verify
the byte count is plausible against the file size). For external web URLs:
include them in the bundle if cited as inline evidence, with the page text
extracted; otherwise the judge will flag them untraced.

**Failure mode this prevents** (Factir 2026-05-23): Subagent A built a 18-line
LINKED NOTES section listing `^N = Person Name` pointers with prose
"contents are characterized in the Update block's prose." The audit flagged
16 spurious untraced claims because the verbatim call bodies weren't in the
bundle to trace against. Memory `feedback_finalize_diligence_bundle_verbatim_mandate`.

### A.1 — Bundle completeness gate (MANDATORY, before audit)

After Step A and BEFORE invoking the runner in Step B, run the deterministic
guard:

```bash
python3 ~/.claude/skills/shared-references/bundle_completeness_check.py \
    --draft  "$DRAFT" \
    --bundle "$SOURCES" \
    [--notes-section 'LINKED NOTES'] \
    [--required-sections 'LINKED NOTES,DILIGENCE MATERIALS'] \
    [--min-chars-per-note 500] \
    [--footnotes-header 'Footnotes for this Section'] \
    [--self-page-ids <hex32>,<hex32>]
```

**Hard gate.** Exit 0 = proceed to Step B. Exit 1 = bundle is missing
verbatim content for at least one cited source. STOP. Re-fetch the missing
notes/docs, rebuild the bundle, re-run the guard. Do NOT run the audit until
the guard passes — the audit verdict on a broken bundle is structurally
meaningless.

The guard checks (canonical IDs B1-B4):

- **B1** — Required bundle sections present (caller binds via
  `--required-sections`; default: `LINKED NOTES,DILIGENCE MATERIALS`).
- **B2** — Every Notion ID emitted as a clickable footnote URL in the draft
  appears literally somewhere in the bundle. Self-page references (the
  artifact citing its own page anchor) exempt via `--self-page-ids`.
- **B3** — Notes section size floor: >= 500 chars per cited Notion note
  (default; tunable). Pointer-manifest entries average <150 chars/note;
  real verbatim bodies are thousands. Catches the manifest pattern even
  if IDs happen to be present.
- **B4** — Every footnote label has at least one 4+ letter distinctive
  token appearing somewhere in the bundle. Catches Drive docs / external
  sources cited but never extracted into the bundle.

**Caller-specific bindings:**

| Caller | --notes-section | --required-sections | --self-page-ids |
|---|---|---|---|
| `finalize-diligence` | `LINKED NOTES` | `LINKED NOTES,DILIGENCE MATERIALS,ORIGINAL FIRST-PASS MEMO,UPDATE BLOCKS` | the Master Diligence Doc page ID |
| `first-pass-diligence` | `LINKED NOTES` | `LINKED NOTES,DILIGENCE MATERIALS` | (none) |
| `update-diligence-priors` | `LINKED NOTES` | `LINKED NOTES,DILIGENCE MATERIALS,ORIGINAL FIRST-PASS MEMO` | the Master Diligence Doc page ID |
| `pre-mortem` | `LINKED NOTES` | `LINKED NOTES,DILIGENCE MATERIALS,MASTER DILIGENCE DOC` | the Master Diligence Doc page ID |
| `product-build-teardown` | `RESEARCH NOTES` | `RESEARCH NOTES,SCREENSHOTS,COMPETITOR REFERENCES` | (none) |
| `draft-investment-memo` | `LINKED NOTES` | `LINKED NOTES,DILIGENCE MATERIALS,REFERENCE MEMOS` | the Master Diligence Doc page ID |

If a caller's bundle uses different section names, override via
`--notes-section` and `--required-sections`. The guard is structurally
agnostic; only the flag values change per caller.

### A.2 — Footnote URL accuracy (RECOMMENDED, runs in seconds)

For drafts that emit clickable `^N [label](url)` footnotes pointing at
Notion pages or Drive files, run the URL/title accuracy check to catch
the "label says X, URL points at Y" failure mode (finalize-diligence
2026-05-23 ^37: labeled "Product Build Teardown" but the Drive URL
resolved to "Preseed plan notes.pdf"):

```bash
python3 ~/.claude/skills/shared-references/footnote_url_check.py \
    --draft "$DRAFT" \
    [--footnotes-header 'Footnotes for this Section'] \
    [--self-page-ids <hex32>,<hex32>]
```

Exit 0 = no Notion URL mismatches detected. Drive URLs emit a
"MANUALLY VERIFY" list with `(label, url)` pairs (script can't fetch
Drive titles without OAuth — operator confirms by clicking each).
Exit 1 = at least one Notion footnote's label and target page title
share zero content tokens — almost certainly a wrong URL.

Pass `--self-page-ids` for the same reason as A.1: self-references
(footnote points at the artifact's own page anchor describing an
in-page section) exempt the label/title overlap requirement.

### A.3 — Round-terms freshness (finalize-diligence + update-priors)

For artifacts that state the round headline in an Overview / opening
paragraph (finalize-diligence Final Assessment, update-diligence-priors
Update blocks), compare the draft's round figures against the Opp page's
canonical `Round Details` property. Catches stale draft numbers
(finalize-diligence 2026-05-23: draft said `$2.5M total` while the round
closed at `$2.3M`; caught as a partial in the audit but should fire here
earlier):

```bash
python3 ~/.claude/skills/shared-references/round_terms_freshness_check.py \
    --draft "$DRAFT" \
    --opp-page-id <Notion Opportunity page ID> \
    [--draft-section Overview]
```

Exit 0 = all Opp round figures appear in the draft section (extra
figures in draft like Inverted check size emit a soft warning but
don't fail).
Exit 1 = at least one Opp figure is missing from the draft — likely
a stale draft.

Skip this check for artifacts that don't reference round terms
(pre-mortem, product-build-teardown).

### A.4 — Claims-to-verify pre-extraction (MANDATORY — proactive phantom-claim prevention)

Before invoking the runner, extract from `$DRAFT` every factual sentence that
carries an inline citation (`[N]`, `[N,M]`, `^N`, or `(per calibration §N)`
style cites) **verbatim** — character-for-character, no paraphrase, no merging
or splitting — and append the list to the END of `$SOURCES` as its own section:

```markdown
==== CLAIMS TO VERIFY (VERBATIM FROM DRAFT) ====

- <verbatim cited sentence 1>
- <verbatim cited sentence 2>
```

The judge prompt binds on this section: it anchors `claim_text` extraction on
these sentences, so the judge only adjudicates claims actually present in the
draft instead of fabricating its own framings (the phantom-extraction failure
mode — upskill 2026-05-19, 3/3 partials were phantom `claim_text` the draft
never contained). The list is a floor and an anchor, not a ceiling — the judge
may still extract additional uncited entity-anchored claims per its inclusion
criteria.

This is prevention. The Step C phantom-claim_text grep below stays in force as
belt-and-suspenders detection — do NOT remove or skip it because A.4 ran.

---

## Step B — Run the audit + iterate

### B.0 — Pre-audit cleanup (MANDATORY on every run)

Before invoking the runner, unconditionally delete any stale audit artifacts
left on disk from a prior run. This guarantees the audit re-runs from scratch
on retries and prevents the skill from silently reusing yesterday's verdict
when the current draft is freshly built.

```bash
rm -f "$AUDIT_JSON" "${AUDIT_JSON%.json}".iter*.json "${AUDIT_JSON%.json}".unchunked.json
rm -f "${ITER_SNAPSHOT_PREFIX}"*.md
rm -f "$NORMALIZED_DRAFT"
```

**Do not skip this even when the prior artifacts "look fresh".** Webhook
retries (e.g. first-pass-diligence re-fired after a `claude --print` timeout)
can leave perfectly-shaped audit JSON on disk that the iteration gate will
happily honor without ever re-running the judge. Concrete prior incident:
Kalos 2026-05-28 retry completed in 24 min because the skill consumed yesterday's
iter1 audit (untraced=32, chunked) instead of running fresh — shipping a draft
with residual misattributions the audit had already flagged. The cleanup
makes the no-reuse rule structural rather than reliant on caller discipline.

`$DRAFT` and `$SOURCES` are NOT cleaned here — they're freshly produced by
the current run's Step A (or upstream draft step) before B is entered.

### B.2.0 — Chunking decision gate (MANDATORY — decide BEFORE the B.1 invocation)

Check the bundle size on disk and force un-chunked mode for any bundle under
the runner's auto-chunk threshold (350KB):

```bash
BUNDLE_BYTES=$(wc -c < "$SOURCES")
if [ "$BUNDLE_BYTES" -lt 350000 ]; then
  CHUNK_FLAG="--chunk-size 0"   # under AUTO_CHUNK_THRESHOLD — force un-chunked from the start
else
  CHUNK_FLAG=""                 # >=350KB — let the runner auto-chunk
fi
echo "chunking gate: bundle=${BUNDLE_BYTES}c chunk_flag='${CHUNK_FLAG}'"
```

Pass `$CHUNK_FLAG` in the B.1 invocation. Under the threshold there is no
reason to ever run chunked — un-chunked is strictly more accurate (see the
chunking notes under B.1). The post-hoc un-chunked re-verification in B.2.1
stays in force as the backstop for >350KB bundles that auto-chunked and
reported untraced > 0.

### B.1 — Invoke the runner

```bash
python3 "$AUDIT_RUNNER" \
  --draft   "$DRAFT" \
  --sources "$SOURCES" \
  --output  "$AUDIT_JSON" \
  $CHUNK_FLAG    # bound by the B.2.0 chunking decision gate (forces --chunk-size 0 under 350KB)
```

Default model is `claude-sonnet-4-6` (1M context). Typical latency: 3-12 minutes
un-chunked; 1-3 minutes per chunk when chunked.

**Source-bundle chunking — auto-chunks above 350KB.** The runner auto-chunks at
section boundaries (`==== … ====`) when sources exceed `AUTO_CHUNK_THRESHOLD`
(350KB as of 2026-05-22; was 80KB historically — bumped after Sonnet 4.6's 1M
context window). Each chunk is judged independently against the full draft;
results are merged with a claim marked `traced` if ANY chunk found a source for
it (positive evidence trumps absence in any single chunk).

**Prefer un-chunked when bundles fit.** Sonnet 4.6's 1M context comfortably
handles 400-500KB bundles in a single pass. Un-chunked is meaningfully more
accurate than chunked for two reasons:

1. **No cross-chunk merge gap.** When the FA cites content from one section and
   the draft is in another, chunk-isolated judges can't see both at once. The
   claim_text exact-match merge across chunks silently misses near-matches —
   producing spurious untraced verdicts (Factir 2026-05-23: chunked v3 reported
   6 untraced; un-chunked v4 against the same bundle reported 0 untraced + 1
   phantom partial).
2. **No claim duplication.** Chunked judges extract claims independently per
   chunk, sometimes double-counting cross-section claims. Un-chunked dedupes
   naturally (Factir v3 chunked = 52 claims; v4 un-chunked = 22 unique claims).

The auto-chunk threshold remains for safety on >350KB bundles, but for typical
diligence runs (50-400KB) pass `--chunk-size 0` explicitly to force a single
un-chunked pass. Memory `feedback_audit_script_chunk_merge_gap` documents the
failure mode.

**Draft batching — auto-batches claim-heavy drafts (independent of source size).**
Source chunking splits the *input* sources, but every judge call still emits
*all* of the draft's claims, so a claim-heavy draft overflows the judge's
output-token ceiling no matter how small the sources are. This is the Fair
2026-07-20 failure: a 127.6KB / 81-claim draft's single-pass audit report
overran the 32K output cap, crashed `claude --print`, and the full-audit retries
blew the 3h job-queue wall-clock cap. The runner now auto-splits the DRAFT at
`## ` section boundaries when it exceeds `DRAFT_AUTO_BATCH_THRESHOLD` (110KB,
target 80KB/batch ≈ 50 claims), auditing each batch against the sources and
concatenating the disjoint claim sets. Judge output ceiling is also raised to
64K tokens. Batching is orthogonal to `--chunk-size`; both can apply. Force with
`--draft-batch-size` (`-1` auto, `0` single-pass, `>0` explicit chars). A batch
that produces zero audits is a coverage gap — the runner records it in
`_failed_draft_batches` and the B.2 gate blocks publish on it (see B.2.2).

Snapshot the pre-audit draft to `<ITER_SNAPSHOT_PREFIX>0.md` before the first
audit so the original is preserved.

### B.2 — HARD EXIT GATE (check after every audit run, iter1 included)

```bash
UNTRACED=$(python3 -c "import json; print(json.load(open('$AUDIT_JSON'))['summary']['untraced'])")
FAILED_BATCHES=$(python3 -c "import json; print(len(json.load(open('$AUDIT_JSON')).get('_failed_draft_batches', [])))")
ITER_N=$(ls "${ITER_SNAPSHOT_PREFIX}"*.md 2>/dev/null | wc -l | tr -d ' ')
if [ "$FAILED_BATCHES" != "0" ]; then
  # NOT clean, regardless of untraced: one or more draft batches produced zero
  # audits, so their claims went entirely UNAUDITED. `untraced` only counts
  # claims the judge actually saw — a failed batch is a silent coverage gap.
  # Re-run the audit ONCE with a halved --draft-batch-size to shrink the
  # offending batch under the output ceiling (see B.2.2). Do NOT publish.
  echo "audit gate: BLOCKED — $FAILED_BATCHES draft batch(es) unaudited; re-run smaller (B.2.2)"
elif [ "$UNTRACED" = "0" ] || [ "$ITER_N" -ge "$MAX_ITER" ]; then
  # PUBLISH NOW — skip to Step C. Do not edit draft. Do not re-run audit.
  echo "audit gate: exit (untraced=$UNTRACED iter=$ITER_N)"
else
  # Iterate per Step B.3.
  echo "audit gate: continue (untraced=$UNTRACED iter=$ITER_N)"
fi
```

**`external_research` verdict is NOT an iteration trigger.** The judge prompt
(see `first_pass_audit.prompt.md`) emits `external_research` for claims that
are legitimately analyst context — public-company financial metrics, generic
regulatory norms, pricing at named public competitors, Tom's prior personal
history with named people. These are tracked in `summary.external_research`
separately from `summary.untraced` and the gate ignores them. Surfaced in the
human report under `=== EXTERNAL_RESEARCH (N) ===` so they're auditable
without polluting the iteration loop.

**`forward_looking` verdict is likewise NOT an iteration trigger.** Open
Questions sections are audited, not skipped: genuine forward-looking
diligence questions receive the `forward_looking` verdict and pass, while
declarative factual assertions embedded inside an Open Questions section
(e.g., "Competitor X already ships this feature") are adjudicated with the
normal traced / partial / untraced verdicts. `summary.forward_looking` is
tracked separately and ignored by the gate.

**You MUST run this check on disk after every audit and obey its verdict.** Do
not reason about whether to iterate based on the audit's prose findings, the
`partial` count, the source notes, or anything else. The gate is `untraced==0`
OR `iter_count>=MAX_ITER`. Full stop.

The 2026-05-19 upskill timeout happened because the drafter ran iter1
(untraced=0, partial=3), then chose to iterate anyway to "fix" the partials —
burning 20+ minutes of audit-judge time on iter2/iter3/iter4 that the spec
explicitly forbids. That run never reached publish and got SIGKILL'd by the
processor with no Notion page, no PDF, no alert.

**`partial` claims are NEVER an iteration trigger.** They are resolved in a
single deterministic pass at Step C below (no re-audit, no search). Do NOT
edit the draft in this loop to upgrade `partial` → `traced`, do NOT re-run
the audit hoping a partial resolves, and do NOT search for new sources in
this loop on partials' behalf.

### B.2.1 — Un-chunked re-verification (MANDATORY when chunked + untraced > 0)

Before entering the iteration loop in B.3, if the audit was chunked AND
`untraced > 0`, re-run the audit ONCE with `--chunk-size 0` (forces un-chunked
single pass). The chunked verdict on cross-section claims is unreliable per the
chunk-merge gap; un-chunked is ground truth.

```bash
if [ "$UNTRACED" -gt 0 ] && [ "$(wc -c < $SOURCES)" -gt 350000 ]; then
  python3 "$AUDIT_RUNNER" \
    --draft   "$DRAFT" \
    --sources "$SOURCES" \
    --output  "${AUDIT_JSON%.json}.unchunked.json" \
    --chunk-size 0
  # Use the un-chunked result as authoritative
  AUDIT_JSON="${AUDIT_JSON%.json}.unchunked.json"
  UNTRACED=$(python3 -c "import json; print(json.load(open('$AUDIT_JSON'))['summary']['untraced'])")
fi
```

This consumes one extra audit run (~3-12 min) but eliminates the
chunk-merge false-positive class entirely. Factir 2026-05-23 result:
chunked v3 had 6 untraced; un-chunked v4 same bundle had 0 untraced.
All 6 were cross-section traceability artifacts.

If un-chunked STILL reports untraced > 0, those are real findings —
proceed to B.3 iteration loop.

### B.2.2 — Failed-batch re-run (MANDATORY when `_failed_draft_batches` is non-empty)

A non-empty `_failed_draft_batches` means one or more draft batches produced
zero parseable audits — usually because a single `## ` section was still large
enough to overflow the judge's output ceiling. Those claims are UNAUDITED, so
the gate blocks publish regardless of `untraced`. Re-run ONCE with a halved
`--draft-batch-size` to force the offending section into smaller batches:

```bash
if [ "$FAILED_BATCHES" != "0" ]; then
  # Halve the per-batch target (default 80KB -> 40KB) to break the oversized
  # section apart. This is orthogonal to $CHUNK_FLAG — keep passing that too.
  python3 "$AUDIT_RUNNER" \
    --draft   "$DRAFT" \
    --sources "$SOURCES" \
    --output  "$AUDIT_JSON" \
    --draft-batch-size 40000 \
    $CHUNK_FLAG
  FAILED_BATCHES=$(python3 -c "import json; print(len(json.load(open('$AUDIT_JSON')).get('_failed_draft_batches', [])))")
  UNTRACED=$(python3 -c "import json; print(json.load(open('$AUDIT_JSON'))['summary']['untraced'])")
fi
```

If it STILL reports `_failed_draft_batches` after the halved re-run, a single
section's claims genuinely can't fit one judge pass — do NOT publish clean.
Surface it in the Slack publish-summary (append `⚠️ Audit incomplete: batches
<list> unaudited — judge output overflow`) and stop; treat like residual
untraced findings after the iteration cap (Step B.4 partial-publish discipline).

### B.3 — Iteration loop (only when the gate says "continue")

For each `untraced` finding, classify and resolve:

a. **Load-bearing fabrication** → cut the claim from the draft. If the
   surrounding paragraph collapses without it, rewrite the paragraph without
   the claim.
b. **Real claim, missing citation** → the substance is sourced but the
   citation isn't marked inline. Add the inline citation pointing to the
   source (e.g., `(per the March 19 call)`, `the deck states:…`).
c. **Forced characterization** → the named entity is real but the
   characterization is invented (e.g., "Rengo compounds a data asset" is
   real; "Rengo's CEO is forward-deployed" is invented). Cut the invented
   half, keep the real half.
d. **Real claim, source missing from bundle** → the claim is plausibly
   true but the drafter pulled it from training data without sourcing. Do
   web research now, find a real source, append it under a new
   `--- Source: <URL or title> ---` block in `$SOURCES` (so the next audit
   pass sees it), and add the inline citation in the draft.
   **Cap: at most `WEB_RESEARCH_CAP` claims per iteration use this path.**
   If more untraced findings classify here, prioritize the load-bearing ones
   (claims the surrounding paragraph depends on) and cut the rest using
   path (a). The cap exists because each research cycle costs 2–5 min —
   uncapped, this path would blow the job timeout.

After resolving:

1. Re-write `$DRAFT` with the resolutions applied.
2. Snapshot to `<ITER_SNAPSHOT_PREFIX>N.md` so the gate's iter count
   advances on disk.
3. Re-run the audit **incrementally** — carry forward verdicts for `## `
   sections the resolutions didn't touch; judge only changed sections.
   Sound because verdicts are per-claim against a fixed evidence bundle
   (the runner fingerprints the bundle and auto-falls-back to a full
   audit if it changed non-append-only). Added 2026-07-21 — a full
   re-judge of an unchanged 130KB draft section costs ~25 min per
   iteration for zero information.

   ```bash
   cp "$AUDIT_JSON" "${AUDIT_JSON%.json}.prev.json"
   # Baseline draft = the snapshot the PRIOR audit judged (second-newest snapshot).
   PREV_SNAP=$(ls "${ITER_SNAPSHOT_PREFIX}"*.md | sort | tail -2 | head -1)
   python3 "$AUDIT_RUNNER" \
     --draft   "$DRAFT" \
     --sources "$SOURCES" \
     --output  "$AUDIT_JSON" \
     --baseline-draft "$PREV_SNAP" \
     --incremental-from "${AUDIT_JSON%.json}.prev.json" \
     --no-clean \
     $CHUNK_FLAG
   ```

   `--no-clean` is MANDATORY on iteration re-runs: the runner's default
   stale-artifact cleanup deletes iter snapshots, which resets the
   gate's `ITER_N` count and defeats the MAX_ITER cap. (Cleanup on the
   FIRST B.1 invocation stays default-on — that's the retry-staleness
   protection.)
4. **Return to the HARD EXIT GATE (B.2)** — do not chain iterations
   without re-checking the gate.

---

## Step C — Step 3.5 Partial-Claim Normalization (single-shot, source-grounded)

### When this runs

After the HARD EXIT GATE in Step B.2 resolves to "publish" (`untraced==0` OR
`iter_count>=MAX_ITER`), but BEFORE the caller's publish step, check:

```bash
PARTIAL=$(python3 -c "import json; print(json.load(open('$AUDIT_JSON'))['summary']['partial'])")
if [ "$PARTIAL" -gt 0 ]; then
  # Run the normalization pass below, publish from $NORMALIZED_DRAFT
  echo "normalization: $PARTIAL partial(s) to fix"
else
  # No partials — publish directly from $DRAFT
  echo "normalization: skip (0 partial)"
fi
```

### What it does

Rewrites each partial claim to match the language of its own cited source.
The audit JSON already contains `source_quote` (what the source says) and
`notes` (the exact gap between draft and source) for every partial — no search
needed, diagnosis is on disk.

### What it does NOT do

- No new web searches.
- No new source bundle additions.
- No re-audit afterwards.
- No iteration.
- No strengthening of the claim beyond what the cited source supports.

### Procedure — ONE pass over all partials, not per-claim

1. Load every partial record from `$AUDIT_JSON`:
   ```bash
   python3 -c "import json; print(json.dumps([c for c in json.load(open('$AUDIT_JSON'))['claims'] if c['verdict']=='partial'], indent=2))"
   ```
   Each record has `claim_text`, `source_quote`, `source_document`, `notes`.

2. **Phantom-claim_text check (MANDATORY before any rewrite).** For each
   partial, grep `$DRAFT` for the literal `claim_text`. If it's absent, the
   judge fabricated the framing — log "judge fabricated claim_text — no
   rewrite needed" and skip that claim. Do NOT trust audit JSON's claim_text
   as ground truth about what the draft actually says. (Observed 2026-05-19
   on upskill first-pass: 3/3 partials were phantom extractions; the draft
   already matched source quotes verbatim.)

3. Read `$DRAFT` once.

4. For each surviving partial, locate the matching prose (literal grep) and
   rewrite per the rule below.

5. Apply all rewrites in a single edit. Save to `$NORMALIZED_DRAFT`.

6. **Do NOT re-run the audit.** The final published draft is
   `$NORMALIZED_DRAFT` when this step ran; otherwise `$DRAFT`. The caller's
   publish step loads its content from the appropriate path — make this
   selection explicit before the publish call.

### The rewrite rule

- **Wrong label, right number** → fix the label, keep the number. E.g., draft
  says "$620B combined market"; source says "$620B recruiting, $637B
  combined." Fix to "$620B recruiting market" (use the source's framing).
- **Wrong stage/series, right round** → fix the stage. E.g., draft says
  "Mercor Series A $100M at $2B"; source says "Series B." Fix to "Series B".
- **Range overstates the data** → narrow to what the source supports. E.g.,
  draft says "$11B–$16B per adjacent vertical (sales, deal orig, expert
  networks, consumer)"; source says Sales=$11B, Deal Orig=$1.5B, Experts=$2.5B,
  Consumer=$16B. Call out the two large verticals by name and drop the bogus
  range, or drop the figure and name the verticals only.
- **No salvageable rewrite** → drop the specific number, keep the directional
  claim with the source citation ("sizable adjacent vertical market" + cite).
  Never invent a substitute number.

### Hard prohibitions in this step

- Do not strengthen any claim beyond `source_quote`. If the source says
  "~$600B," the draft must not say "$620B" — even if a different source you
  remember says $620B. You are bound to what the audit attested to.
- Do not add new citations. The bundle is closed.
- Do not introduce hedge language ("approximately," "roughly") if the source
  states the number flat. Match the source's register.
- Do not delete more than the claim itself. Keep surrounding prose intact.

### Why no re-audit

Re-auditing reopens the loop the HARD EXIT GATE exists to prevent. The
rewrite is source-quote-grounded — verifiable by
`grep "<rewritten claim>" $NORMALIZED_DRAFT` followed by a literal compare
against `source_quote`. That is the verification; a second LLM-judge pass adds
latency without rigor.

---

## Step D — Surface what remains in the publish summary

After the loop terminates and Step C has run, the caller's publish summary
(Slack alert, response message, etc.) must include:

- The final iteration count and the audit JSON output path.
- Every remaining `untraced` claim (if any) with the judge's notes — so Tom
  can see what the drafter couldn't resolve.
- Every normalized partial as a before → after diff (Step C output) so Tom
  can spot-check the rewrite. Unnormalized partials should not exist in the
  published draft — Step C is mandatory when `audit.summary.partial > 0`.
- Every `external_research` claim whose judge confidence is below 0.9,
  flagged as `borderline — check manually`. (The judge emits a numeric
  0.0–1.0 `confidence` on external_research verdicts; sub-0.9 surfaces in
  the run report only — it does NOT block the gate or trigger iteration.)

Caller-specific surface formats (Slack rich-text shape, page-summary block,
etc.) are the caller's responsibility — but the substance above is required.

---

## Exit codes (from the audit runner)

- `0` = zero untraced this run
- `1` = at least one untraced this run
- `2` = judge invocation or parse failure

Codes 0 and 1 drive the iteration loop. **Code 2 STOPS the iteration loop
and the publish** — investigate the failure.

---

## Hard prohibitions (across the whole audit flow)

### Never use the "audit invocation failed" Slack escape hatch

Tom explicitly rejected this pattern on 2026-05-15: publishing the draft with
a `⚠️ Audit invocation failed on chunk N` line as a workaround when the audit
hit exit 2 is **not acceptable**. The `⚠️` 4th line in the publish summary is
reserved ONLY for the case where the audit RAN cleanly but finished with
residual untraced/partial findings after the iteration cap.

If the audit script itself failed to execute (exit 2: timeout, parse failure on
all chunks, judge crash), the response is to fix the audit script or the judge
prompt and re-run — never to publish the draft with a caveat line. The audit
is the only check that catches subtler hallucinations; bypassing it puts the
burden on Tom to spot what the judge would have caught.

**Diagnostics for common parse failures** (all patched 2026-05-15 in
`first_pass_audit.py`): judge emits bare JSON without the `<audit_report>`
wrapper (runner falls back to bare-JSON extraction); single chunk's judge
crashes (runner continues past, contributes zero claims from that chunk, only
aborts if ALL chunks fail). If a new failure mode appears the script can't
recover from, patch the runner inline and re-run — do not ship.

### Never auto-strip silently

Every iteration's revisions must be visible to Tom in the diff between
`<ITER_SNAPSHOT_PREFIX>0.md` (pre-audit) and the final draft. Save each
intermediate iteration to `<ITER_SNAPSHOT_PREFIX>N.md` so the loop's edits can
be reconstructed.

---

## What lives in the judge prompt (not here)

The claim taxonomy (founder_name / portfolio_status / memo_citation /
analog_biographical / analog_business_fact / numeric_claim / regulatory_claim /
founder_history / third_party_business_fact / market_norm /
competitive_dynamics / technical_specificity / gtm_motion_qualitative), the
verdict definitions (traced / partial / untraced), and the output JSON schema
live in `$JUDGE_PROMPT`. Read that file for the audit's epistemic rules. This
SKILL.md owns the *operational* layer: when to run, how to gate, how to
iterate, how to normalize, how to surface.
