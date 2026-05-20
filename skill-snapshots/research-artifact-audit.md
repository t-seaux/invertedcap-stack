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

---

## Step B — Run the audit + iterate

### B.1 — Invoke the runner

```bash
python3 "$AUDIT_RUNNER" \
  --draft   "$DRAFT" \
  --sources "$SOURCES" \
  --output  "$AUDIT_JSON"
```

Default model is `claude-sonnet-4-6`. Typical latency: 1–3 minutes per chunk.

**Source-bundle chunking — automatic above 80KB.** The runner auto-chunks the
source bundle at section boundaries (`==== … ====`) when it exceeds 80KB,
targeting ~50KB per chunk. Each chunk is judged independently against the full
draft; results are merged with a claim marked `traced` if ANY chunk found a
source for it (positive evidence trumps absence in any single chunk). To force a
single-pass un-chunked audit (small bundles or debugging), pass `--chunk-size 0`.

Snapshot the pre-audit draft to `<ITER_SNAPSHOT_PREFIX>0.md` before the first
audit so the original is preserved.

### B.2 — HARD EXIT GATE (check after every audit run, iter1 included)

```bash
UNTRACED=$(python3 -c "import json; print(json.load(open('$AUDIT_JSON'))['summary']['untraced'])")
ITER_N=$(ls "${ITER_SNAPSHOT_PREFIX}"*.md 2>/dev/null | wc -l | tr -d ' ')
if [ "$UNTRACED" = "0" ] || [ "$ITER_N" -ge "$MAX_ITER" ]; then
  # PUBLISH NOW — skip to Step C. Do not edit draft. Do not re-run audit.
  echo "audit gate: exit (untraced=$UNTRACED iter=$ITER_N)"
else
  # Iterate per Step B.3.
  echo "audit gate: continue (untraced=$UNTRACED iter=$ITER_N)"
fi
```

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
3. Re-run the audit (Step B.1) against the updated draft.
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
