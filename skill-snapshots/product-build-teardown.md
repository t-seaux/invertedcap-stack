---
name: product-build-teardown
description: >-
  Produce a long-form Product Build Teardown for a pipeline Opportunity. Pulls Notion context (page body,
  Diligence Materials, transcripts), explores the company's public surface (site, docs, API reference, demos)
  with screenshots, walks competitor/peer sites, and researches via Exa (primary) + Parallel (fallback).
  Produces a six-section analysis — Product Anatomy, Delivery Mechanism, Build Cost to v1, Path to
  Production-Grade, Moat Read, Killshots — each cited with cost/time estimates tied to the shared
  cost-calibration table. Outputs a Notion Notes page (Category = Diligence), a first-pass-styled PDF, a Drive
  upload, and a Slack alert. Trigger: "product teardown on X", "build teardown on X", "product build teardown on
  X", "do a teardown on X", or any teardown request on a named opportunity. Always inline — no confirmation.
---

# Product Build Teardown

Generate a long-form Product Build Teardown for a single pipeline opportunity.
The teardown answers four interlocked questions:

1. What does the underlying product build look like? What are the key
   components, integrations, and data sources?
2. How hard is the build? How much time and money to get a working v1?
3. Beyond v1, what does it take to harden the product so it runs at real
   scale with real customers (not a vibe-coded MVP)?
4. What is the underlying moat? And — flipping the lens — what would kill
   this product?

The analysis is **evidence-grounded and opinionated**. Cost and time
estimates must trace to specific rows in the cost-calibration reference
table; product/architecture claims must trace to materials, calls, or
public artifacts. Speculation is allowed but must be flagged as such and
anchored to a structural reason.

This skill **shares the audit and PDF-formatting discipline of
first-pass-diligence**. Long-form factual prose runs the canonical
`research-artifact-audit` flow, and the PDF matches the first-pass spec
exactly — same font family, sizes, leading, spacing, page numbers,
citation style, and bold/italic conventions.

---

## When NOT to run this skill

- **First-pass diligence has not yet been done and Tom wants a full
  first-pass.** Run `first-pass-diligence` instead — its Product section
  covers a slim version of this teardown framework, sufficient for early
  stages where material is thin.
- **Follow-on rounds (FO).** Same exclusion as first-pass — Tom already
  has full context. Detect via `(FO)` in the Opportunity name or prior
  Opportunity entries for the same company.
- **No materials AND no public product surface AND no call transcript.**
  Halt and ask Tom for at least one of: deck/demo, website + product page,
  call notes. Do not fabricate a teardown from thin air.

---

## Step overview (runbook spine)

1. **Step 1** — Gather all available context (Notion Opp, notes, materials, public/eng/competitor crawl, framework + calibration specs).
2. **Step 2** — Draft the six-section analysis to the citation + section-order spec.
3. **Step 3** — Pre-Publish Lint (MANDATORY GATE).
4. **Step 4** — LLM Audit (MANDATORY SURFACE / HARD EXIT GATE).
5. **Step 5** — Create the Notion page.
6. **Step 6** — Build the PDF (matches first-pass formatting exactly).
7. **Step 7** — Upload to Drive, link in Notion, send the Slack alert.

---

## Step 1: Gather All Available Context

Anchor the start time first so the audit-start Slack alert (Step 4b) can
report elapsed-from-start minutes:

```bash
date +%s > /tmp/teardown_start_ts.txt
```

### 1a. Fetch the Notion Opportunity

Search the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`) for
the named company. Fetch the full page with `notion-fetch`. Extract:

- All properties (stage, round details, status, description, scores, HQ,
  founders, website if present)
- Full page body (summary, team bios, round context, source context)
- All linked Notes (the ✍️ Notes relation) — fetch each
- Diligence Materials property — note Drive URLs and inline links

### 1b. Fetch All Linked Notes (with transcripts)

For every Notes-relation page, call `notion-fetch` with
`include_transcript: true`. Pay close attention to:

- **Founder call transcripts** — demo walkthroughs are usually here.
  When a founder narrates a click-through demo on a call, the
  transcript captures the verbal description of every screen / data
  flow / integration. Treat that prose as a primary architectural
  source.
- **Backchannel notes** — references may volunteer architectural detail
- **Existing first-pass-diligence note** (if one exists) — its Product
  section is the skeleton; this teardown deepens it. Competitor / peer
  names listed in first-pass are seed inputs for §2c below.

### 1c–1g. Crawl the surfaces (Diligence Materials, public product surface, engineering signal, competitors, broad web research)

Fetch and crawl every evidence surface — diligence materials, the company's
public product surface, public engineering signal, competitor/peer surfaces,
and broad-based web research (Exa primary, Parallel fallback). Full procedure
in `references/step-1-context-gathering.md` — read it now before proceeding.

### 1h. Read the Canonical Framework Spec + Cost-Calibration Reference

Read both of these **in full** at the start of every run:

1. **Framework spec** —
   `/Users/tomseo/.claude/skills/shared-references/product-build-teardown-framework.md`.
   This is the canonical six-section framework (Product Anatomy, Delivery
   Mechanism, Build Cost & Time to v1, Path to Production-Grade, Moat
   Read, Killshots) — structure, depth requirements, table formats,
   citation discipline, killshot taxonomy. The same file is read by
   `first-pass-diligence` §2 Product and `update-diligence-priors`
   Product refreshes — single source of truth, no drift across
   invocation paths.

2. **Cost-calibration** —
   `/Users/tomseo/.claude/skills/shared-references/product-build-cost-calibration.md`.
   Calibration source for every FTE-month, infra unit cost, LLM API
   rate, and integration cost the teardown writes. Every estimate must
   cite a specific section as `(per calibration §<N>)`. If a cost
   figure doesn't correspond to a row, either (a) cite the closest row
   and explain the adjustment, or (b) flag the gap explicitly in the
   artifact and update the calibration file in a separate pass.

---

## Step 2: Draft the Analysis

Write the six-section analysis (Context + Sections 1–6 + Suggested Additional
Analysis + Sources) in a professional analytical voice, applying the canonical
framework spec, the Citation Discipline (bracket footnotes / `^N` block /
`[1,2]` multi-source), and the exact Section Order. Full procedure in
`references/step-2-analysis-spec.md` — read it now before proceeding.

---

## Step 3: Pre-Publish Lint — MANDATORY GATE

Write the draft markdown to `/tmp/teardown_draft.md`, then run the
deterministic lint (first-pass hallucination-class lint + the
calibration-cite resolution check). Exit 0 proceeds; exit 1 stops and
requires fix-and-rerun. Full procedure in `references/step-3-4-gates.md`
— read it now before proceeding.

---

## Step 4: LLM Audit — MANDATORY SURFACE

Run the canonical `research-artifact-audit` flow (HARD EXIT GATE, iteration
loop, exit codes, never-use-the-bypass-alert prohibition), applying the
teardown-specific bindings and source-bundle structure, and firing the
audit-start Slack alert before building the bundle. **Read
`~/.claude/skills/research-artifact-audit/SKILL.md` in full**, then follow
`references/step-3-4-gates.md` — read it now before proceeding. Both the
lint and the audit must pass before publish.

---

## Step 5: Create the Notion Page

Select the final draft (normalized if present), create the page in the Notes
database (Category = Diligence) with the mandatory title format, and set the
`:claude-color:` icon. Full procedure in `references/step-5-7-publish.md` —
read it now before proceeding.

---

## Step 6: Build the PDF (matches first-pass-diligence formatting EXACTLY)

Build the PDF from the canonical template, reading the long-form-pdf-spec in
full so the teardown PDF is visually indistinguishable from a first-pass PDF.
Full procedure in `references/step-5-7-publish.md` — read it now before
proceeding.

---

## Step 7: Upload to Drive, Link in Notion, Send Alert

Upload the PDF to Drive (**MANDATORY — always a NEW Drive file, never
overwrite in place**), link it in the Opp page body AND the **Diligence
Materials Files property (MANDATORY, verify the write)**, then send the
3-line Slack alert. Full procedure in `references/step-5-7-publish.md` —
read it now before proceeding.

---

## Quality Bar

A good Product Build Teardown:

- Names specific components, data flows, and integrations — anchored
  in cited evidence, never invented to look thorough
- Anchors every cost / time / FTE-month estimate to a specific section
  of the cost-calibration reference, and surfaces material disagreements
  with disclosed company data as analytical findings
- Distinguishes clearly between v1 ("real customer can use it") and
  production-grade ("real customers can scale on it") — the gap is the
  hardening work in Section 4
- Makes the moat read honest, including the counter-case — well-rounded
  mediocrity on moat is a negative, not a wash
- Writes Killshots that are specific failure mechanisms, not generic
  category-risk language; each killshot has structural evidence rooted
  in the product's actual architecture
- Cites every numeric, technical, and entity-anchored claim — passes
  the `first_pass_audit.py` judge with zero untraced after iteration
- Matches first-pass-diligence visual formatting in the PDF exactly —
  the two artifacts should look like siblings in the Diligence
  Materials folder

A bad Product Build Teardown reads like a category overview or a
generic "what would you build" exercise. If you find yourself
speculating about architecture or cost without grounding it in the
materials, the public surface area, or the calibration table, stop
and either gather more evidence or flag the gap.
