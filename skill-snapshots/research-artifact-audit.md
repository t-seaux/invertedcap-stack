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

For drafts that emit clickable `^N [label](url)` footnotes pointing at Notion
pages or Drive files, run the URL/title accuracy check to catch the "label says
X, URL points at Y" failure mode. Full procedure (invocation, exit-code meaning,
`--self-page-ids` exemption) in `references/step-a-source-bundle.md`; **read it
now before proceeding.**

### A.3 — Round-terms freshness (finalize-diligence + update-priors)

For artifacts that state the round headline in an Overview / opening paragraph,
compare the draft's round figures against the Opp page's canonical `Round
Details` property to catch stale draft numbers. Skip for artifacts that don't
reference round terms (pre-mortem, product-build-teardown). Full procedure
(invocation, exit-code meaning) in `references/step-a-source-bundle.md`; **read
it now before proceeding.**

### A.4 — Claims-to-verify pre-extraction (MANDATORY — proactive phantom-claim prevention)

Before invoking the runner, extract from `$DRAFT` every inline-cited factual
sentence **verbatim** and append it to the END of `$SOURCES` as a `==== CLAIMS
TO VERIFY (VERBATIM FROM DRAFT) ====` section, so the judge anchors `claim_text`
extraction on real draft sentences instead of fabricating its own framings (the
phantom-extraction failure mode). This is prevention; the Step C
phantom-claim_text grep stays in force as belt-and-suspenders detection — do NOT
remove or skip it because A.4 ran. Full procedure (exact extraction rules,
section format) in `references/step-a-source-bundle.md`; **read it now before
proceeding.**

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

Run `$AUDIT_RUNNER` against `$DRAFT` / `$SOURCES`, passing `$CHUNK_FLAG` bound by
the B.2.0 chunking gate, and snapshot the pre-audit draft to
`<ITER_SNAPSHOT_PREFIX>0.md` before the first audit. Full procedure (exact
invocation, source-bundle chunking behavior, prefer-un-chunked rationale, draft
batching / `_failed_draft_batches` behavior) in
`references/step-b-audit-iteration.md`; **read it now before proceeding.**

**HARD RULE — invoke the runner SYNCHRONOUSLY (foreground), never with
`run_in_background: true`.** When this audit runs inside a subagent, backgrounded
children can die silently the moment the subagent's turn yields — 0-byte output
file, no process, no error surfaced — and the run hangs forever on a wait that
never fires. Foreground with a generous explicit Bash timeout (up to the 600000ms
max; the runner routinely takes several minutes on large bundles). If a single
invocation would genuinely exceed the max timeout, use the B.2.0 chunking gate to
shrink the per-call work rather than backgrounding. A visible timeout/error beats
a silent zombie wait. Incident: Fair 2026-07-29 (update-diligence-priors run) —
backgrounded audit + relabel jobs all died at spawn; run hung 2+ hours. Memory:
`feedback_subagent_background_jobs_die_silently`.

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
`untraced > 0` (bundle > 350KB), re-run the audit ONCE with `--chunk-size 0`
(forces un-chunked single pass) and treat that as authoritative — the chunked
verdict on cross-section claims is unreliable per the chunk-merge gap. If
un-chunked STILL reports untraced > 0, those are real findings — proceed to B.3.
Full procedure (exact re-run + `$AUDIT_JSON` reassignment) in
`references/step-b-audit-iteration.md`; **read it now before proceeding.**

### B.2.2 — Failed-batch re-run (MANDATORY when `_failed_draft_batches` is non-empty)

A non-empty `_failed_draft_batches` means one or more draft batches produced zero
parseable audits, so their claims are UNAUDITED and the gate blocks publish
regardless of `untraced`. Re-run ONCE with a halved `--draft-batch-size` to break
the offending section into smaller batches. If it STILL reports
`_failed_draft_batches`, do NOT publish clean — surface `⚠️ Audit incomplete:
batches <list> unaudited — judge output overflow` in the Slack publish-summary
and stop (treat like residual untraced after the iteration cap). Full procedure
(exact re-run) in `references/step-b-audit-iteration.md`; **read it now before
proceeding.**

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

Runs after the HARD EXIT GATE in Step B.2 resolves to "publish" (`untraced==0`
OR `iter_count>=MAX_ITER`), but BEFORE the caller's publish step, and ONLY when
`audit.summary.partial > 0`: in a single source-grounded pass, rewrite each
partial claim to match the language of its own cited source (`source_quote`
already on disk in `$AUDIT_JSON`) and save to `$NORMALIZED_DRAFT` — no new web
searches, no bundle additions, no re-audit, no iteration, no strengthening
beyond what the source supports. The published draft is `$NORMALIZED_DRAFT` when
this step ran, otherwise `$DRAFT`. Full procedure — the trigger check, the
MANDATORY phantom-claim_text grep, the one-pass procedure, the rewrite rule, and
this step's hard prohibitions — in
`references/step-c-partial-normalization.md`; **read it now before proceeding.**

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
