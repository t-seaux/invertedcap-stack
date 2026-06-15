---
name: product-build-teardown
description: >
  Produce a long-form Product Build Teardown for a pipeline opportunity. Pulls all
  available context from the Notion Opportunity (page body, Diligence Materials,
  call transcripts), explores the company's public surface area (website, docs,
  API reference, demos), takes screenshots of anything informative, walks
  competitor / peer sites mentioned in materials or first-pass docs, and uses
  Exa (primary) + Parallel (fallback) for broad-based product teardown
  research. Produces a six-section analysis covering Product Anatomy, Delivery
  Mechanism, Build Cost to v1, Path to Production-Grade, Moat Read, and
  Killshots — each grounded in cited evidence with cost/time estimates tied to
  the shared cost-calibration table. Outputs a Notion Notes page (Category =
  Diligence, linked to the Opp), a formatted PDF matching first-pass-diligence
  styling, a Drive upload, and a Slack alert. Trigger whenever Tom says
  "product teardown on X", "build teardown on X", "product build teardown on
  X", "do a teardown on X", or any variant requesting a product/build
  teardown analysis on a named opportunity. Always trigger inline — no
  confirmation needed before acting.
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

### 1c. Fetch Diligence Materials

Use the same access patterns as `first-pass-diligence` Step 1c:

- **Google Drive PDFs** — `read_pdf_bytes` with
  `https://drive.google.com/uc?export=download&id={FILE_ID}`
- **Google Docs** — `google_drive_fetch`
- **Notion-hosted attachments** — Chrome + signed URL flow (see
  first-pass-diligence Step 1c for the exact JS pipeline)
- **DocSend** — read `docsend-to-pdf/SKILL.md` and convert to local PDF
- **Video files** — Whisper via
  `~/.claude/scripts/transcribe_video.py /tmp/teardown_demo_<name>.<ext> --model small`
- **Loom / hosted video** — extract transcript panel via Chrome JS, fall
  back to Whisper if no panel

Each transcribed demo gets folded into the source bundle under
`==== DILIGENCE MATERIALS ====` with `--- Material: <name> demo
(transcribed) ---`. Demo transcripts are first-class architectural
evidence — they reveal loop structure, partner names, latency targets,
and integration points that don't appear in the deck.

### 1d. Crawl the Company's Public Surface Area

Navigate Chrome to the company's primary URL (from the Opportunity
`URL` property or the deck). Walk these surfaces if present and
screenshot anything informative:

- **Marketing site** — what does the company say it does? Capture the
  hero copy and feature breakdown.
- **Product / how-it-works page** — typically the highest-density
  architectural signal on the marketing site.
- **Pricing page** — tier structure reveals delivery shape (per-seat /
  per-API-call / per-record / consumption / flat enterprise) and target
  customer size.
- **Docs / developer / API reference** — if present, this is gold.
  Extract: object model (what entities exist), endpoints (what
  operations the product supports), authentication shape (API key /
  OAuth / SSO), webhook events (what state changes the product
  externalizes), data ingestion paths (file upload / connectors / SDK),
  rate limits (capacity signal).
- **Status page / changelog** — reveals service architecture (which
  components are isolated), incident history, release cadence.
- **Public roadmap / community / Discord** — direction signal.
- **Integrations / app directory page** — every named integration is a
  data source or delivery point worth listing in §1.
- **Security page / trust center** — SOC 2 / HIPAA / ISO 27001
  certifications, sub-processor list, data residency. Subprocessor
  list is especially useful — it reveals the actual infra stack
  (Postgres host, vector DB, LLM provider, email/SMS, payments,
  observability).
- **Demo / interactive product tour** — record screenshots of the click
  path; if there's a video, transcribe it.

**Screenshots:** save to `/tmp/teardown_screens_<n>.png` and embed
later in the artifact. Take screenshots when the visual content carries
information that prose can't fully convey (UI shape, data model
visible in a screen, integration list, pricing tiers, architecture
diagram on the marketing site). Don't screenshot for decoration.

### 1e. Crawl the Public Engineering Signal

Run these in parallel with §1d:

- **GitHub** — search for the company's org (`github.com/{slug}`,
  `github.com/{company-name}`). Note: language mix, repo count, recent
  commit cadence, whether they ship an SDK / CLI / Terraform provider.
- **Careers / job postings** — `{company}.com/careers`, plus Greenhouse
  (`boards.greenhouse.io/{slug}`), Lever (`jobs.lever.co/{slug}`),
  Ashby. Engineering JDs reveal the actual stack ("Postgres, Kafka,
  Rust, Kubernetes" in a JD is more honest than marketing). Team shape
  matters too — "3 backend + 1 ML + 1 platform" reveals build
  complexity. **Capture verbatim stack strings + the JD URL** so the
  audit can trace.
- **Engineering blog** — `{company}.com/engineering`,
  `{company}.com/blog`. Architecture deep-dives, lessons-learned posts,
  and "how we built X" articles are dense signal.
- **LinkedIn company page** — confirm headcount, eng vs. non-eng mix.

### 1f. Crawl Competitor / Peer Surfaces

Build the competitor list from:

1. Competitor mentions in the founder call transcript or Tom's notes
2. The Competitive Landscape section of the first-pass-diligence note
   (if one exists)
3. Deck "competitive landscape" slides
4. Exa "competitors of {company}" query (broad-based search, see §1g)

For each named competitor, repeat the §1d crawl at a lighter touch —
product page, docs, pricing, integrations. The goal is to triangulate
on a **delivery mechanism** baseline (do peers ship as a standalone
app, an embedded SDK, an API only, a Salesforce app, etc.) and a
**build complexity** baseline (do peers have 5 integrations or 50?
how mature is their docs?). Screenshot meaningful differences.

### 1g. Broad-Based Web Research

For broader teardown-style research — "how does product X actually
work under the hood", "what's the typical build for category Y" —
use this provider order:

1. **Exa search** (primary). Issue 3–8 targeted queries:
   - `"{company name}" architecture OR "how it works" OR "tech stack"`
   - `"{company name}" engineering blog`
   - `"{competitor name}" architecture OR "how it works"`
   - `"{category}" "tech stack" OR "build" OR "architecture"`
   - Founder name + technical talks / podcasts
2. **Parallel** (fallback) — if Exa results are thin, low quality, or
   skewed toward marketing pages. Document in the published artifact
   that Parallel was used and why.

For every external source the artifact ends up citing, save the URL +
key quote into the source bundle under `==== EXTERNAL RESEARCH ===`.

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

Write in a professional analytical voice — factual, opinionated where
warranted, not promotional. Each section contains substantive
paragraphs (4–6 sentences minimum), not one-liners. When evidence is
thin, say so explicitly rather than padding.

### Citation Discipline

**Citation style is identical to first-pass-diligence:** footnote
numbers in brackets — `[1]`, `[2]`, etc. — throughout the entire
document. NEVER write parentheticals like `(per the deck)`, `(per the
March 19 call)`, `(per the docs)`. At the end of the Context section,
include a compact footnote block where each entry uses `^N` at the
start of the line (caret + number, no brackets, no colon):

> The docs describe a per-tenant database isolation model [1], and the
> April 8 founder call confirms the integration stack [2].
>
> ^1 Company Name — Developer Documentation (api.{company}.com/docs)
> ^2 Founder Intro Call — April 8, 2026

**Multi-source citations — single bracket, no space after comma:**
`[1,2]`, `[1,2,3]`, never `[1] [2]`, `[1][2]`, `[1, 2]`, or `[1-3]`.

**Reserve `[N]` markers for cases where the specific source artifact
matters.** Sentences making generic observations about the category
("most modern B2B SaaS run on Postgres") don't need citation.

### Section Order

Do NOT lead the body with a date stamp — the Notion page title and PDF
subtitle both carry it. Start the body directly with the **Context**
section (three bold sub-headers: Company Overview, Materials & Sources
Reviewed, Cost-Calibration Anchor).

The analysis follows this exact structure:

**Context**

***Company Overview.*** A factual 1–2 paragraph description: what the
product does, who it serves, business model, stage. Descriptive, not
evaluative.

***Materials & Sources Reviewed.*** Bulleted inventory grouped by source
type — Notion Opportunity, call notes, Diligence Materials, public
surface area crawled (with high-level URL list), engineering signal
pulled (GitHub / careers / blog), competitors crawled, external
research categories. Terse, no URLs (the terminal Sources section
carries them).

***Cost-Calibration Anchor.*** A single sentence pointing the reader at
`product-build-cost-calibration.md` as the unit-cost source. Format:
"Cost and time estimates throughout this teardown trace to the
calibration table at [product-build-cost-calibration.md] (sections
referenced inline)."

Then the Context footnote block (compact `^N` entries).

---

**Sections 1–6 — Apply the canonical framework spec.**

Sections 1–6 below all follow the spec at
`/Users/tomseo/.claude/skills/shared-references/product-build-teardown-framework.md`
verbatim — structure, depth, table formats, citation discipline, and
prohibitions all live in the canonical file. The standalone teardown
inherits the spec directly; the section labels below are pointers, not
restatements.

**1. Product Anatomy** — apply framework spec §1. Core Components
(flowing prose + bullet inventory, lead with primary loop), Data
Flows, Integrations & Data Sources table. End with ***Open
Questions***.

**2. Delivery Mechanism** — apply framework spec §2. Name primary
delivery archetype, call out secondary surfaces, assess distribution
implications. When citing a competitor's integration or feature counts
as a baseline, anchor comparable maturity explicitly — note whether the
competitor is peer-stage or much more mature (years in market, funding
stage), so the count reads as a trajectory marker rather than an
apples-to-apples gap. End with ***Open Questions***.

**3. Build Cost & Time to v1** — apply framework spec §3. Both
dollars and calendar time get explicit treatment — neither is the
"primary" axis. Engineering Scope (FTE-month range), Team
Composition, Calendar Time to v1 (range in months from kickoff with
friction adjustments named), Infra & API Run-Rate table, Total v1
Capital Required, Assumptions paragraph. End with ***Open Questions***.

**4. Path to Production-Grade** — apply framework spec §4. Data
quality maintenance, operational reliability, scale stress points,
customer-facing edge cases, security & compliance gates, support
tooling. End with a Production-Hardening Cost & Time estimate
(calibration §7). End with ***Open Questions***.

**5. Moat Read** — apply framework spec §5. Survey the moat sources
in prose (unique data, integration depth / switching costs,
distribution lock, regulatory barrier, technical complexity / scarce
expertise, network effects), then write an honest counter-case
paragraph. Be opinionated.

**6. Killshots — What Kills This Product** — apply framework spec
§6. Each killshot is a numbered sub-subsection (6.1, 6.2, …) with
descriptive title, analytical paragraph, and bold-italic
***Failure mode:*** paragraph. End with a Killshot Summary table
(Failure Mode | Key Evidence | Kill Shot?). Standalone teardown
has no separate §6 Risks counterpart (unlike first-pass-diligence),
so no cross-reference applies — each killshot is fully self-contained
here. Each killshot must additionally note whether a named competitor
has already solved or executed that killshot mechanism (with the
evidence cited); "no known competitor has executed this" is an
acceptable explicit finding — silence is not.

---

**Suggested Additional Analysis**

Bulleted list of specific follow-up workstreams that would tighten the
teardown — questions for the founder, materials worth requesting, peer
products worth deeper hands-on testing, calibration-table rows that
need refining for this specific category.

---

**Sources**

Grouped into exactly two sections. Every source MUST be a clickable
hyperlink wherever a URL exists.

1. **Internal Materials** — Notion Opportunity, linked notes, Diligence
   Materials (Drive URLs).
2. **External References** — every web source researched (company
   website pages, docs URL, careers / JD URLs with verbatim stack
   strings extracted, competitor URLs, Exa / Parallel search results,
   engineering blog posts). Include a **Key Data Points Referenced**
   sub-section for the numeric facts cited.

---

## Step 3: Pre-Publish Lint — MANDATORY GATE

Write the draft markdown to `/tmp/teardown_draft.md`, then run the same
deterministic lint used by first-pass-diligence. The lint catches
hallucination classes (fabricated founder names, fabricated portfolio
references, fabricated numbers, fabricated memo citations).

```bash
python3 /Users/tomseo/.claude/skills/first-pass-diligence/first_pass_lint.py \
  --draft /tmp/teardown_draft.md \
  --manifest /tmp/teardown_manifest.json
```

Build the manifest with the same shape as first-pass-diligence Step 4a
(see `first-pass-diligence/SKILL.md` for the manifest schema). Exit 0
proceeds; exit 1 stops and requires fix-and-rerun.

**Calibration-cite resolution check (deterministic, same hard gate).**
Every `(per calibration §N)` citation in the draft must resolve to an
actual section of the calibration reference (its sections are `## N. Title`
headings):

```bash
CAL=/Users/tomseo/.claude/skills/shared-references/product-build-cost-calibration.md
FAIL=0
for N in $(grep -oE 'per calibration §[0-9]+' /tmp/teardown_draft.md | grep -oE '[0-9]+' | sort -un); do
  grep -qE "^## ${N}\." "$CAL" || { echo "UNRESOLVED CITE: calibration §${N}"; FAIL=1; }
done
[ "$FAIL" = "0" ] && echo "calibration cites: all resolve"
```

Any unresolved cite = fix before publish (re-point to the right section, or
apply the Step 1h closest-row / flag-the-gap rule). Named-section cites
(`per calibration's Engineering FTE rates`) resolve trivially and are the
preferred form per the calibration file's "How to cite" section. The audit's
bundle-level verification remains as backstop.

---

## Step 4: LLM Audit — MANDATORY SURFACE

The audit mechanics (HARD EXIT GATE, iteration loop, Step 3.5 partial
normalization, exit codes, the never-use-the-bypass-alert prohibition)
are defined ONCE in `~/.claude/skills/research-artifact-audit/SKILL.md`.
**Read that file in full before continuing.**

**Teardown-specific wiring — apply these bindings:**

| Binding (per research-artifact-audit) | Value (product-build-teardown) |
|---|---|
| `DRAFT` | `/tmp/teardown_draft.md` |
| `SOURCES` | `/tmp/teardown_sources.md` |
| `AUDIT_JSON` | `/tmp/teardown_audit.json` |
| `JUDGE_PROMPT` | `/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.prompt.md` |
| `AUDIT_RUNNER` | `/Users/tomseo/.claude/skills/first-pass-diligence/first_pass_audit.py` |
| `MAX_ITER` | `3` |
| `WEB_RESEARCH_CAP` | `6` |
| `ITER_SNAPSHOT_PREFIX` | `/tmp/teardown_draft.iter` |
| `NORMALIZED_DRAFT` | `/tmp/teardown_draft.normalized.md` |

The first-pass judge prompt + runner are reused — the audit's
epistemic rules (founder_name, third_party_business_fact,
technical_specificity, numeric_claim, etc.) cover the teardown's claim
types without modification. The technical_specificity and
numeric_claim taxa are especially important for this skill — they're
where teardown drafts most often drift into training-data
pattern-matching.

**Fire the audit-start Slack alert BEFORE building the source bundle:**

```bash
ELAPSED_MIN=$(( ($(date +%s) - $(cat /tmp/teardown_start_ts.txt)) / 60 ))
COMPANY="<subject company name>"
cat <<EOF | /Users/tomseo/.claude/skills/send-alert/send.sh
🧪 Product build teardown audit starting for **${COMPANY}** — T+${ELAPSED_MIN} min from job start
EOF
```

Do NOT include `💬 Reply in thread` in this start ping (reserved for the
final publish alert).

**Source bundle structure** for `/tmp/teardown_sources.md` (the runner
chunks at the `==== … ====` boundaries):

```markdown
==== OPPORTUNITY PAGE ====

<full Opp page body, including properties summary>

==== LINKED NOTES ====

--- Note: <title> (<date>) ---
<full content, including transcript where present>

==== DILIGENCE MATERIALS ====

--- Material: <name> ---
<full extracted text>

==== PUBLIC PRODUCT SURFACE ====

--- Page: <URL> ---
<key extracted text / structured signal from the company's marketing,
product, pricing, docs, integrations, security pages>

==== ENGINEERING SIGNAL ====

--- JD: <URL> ---
<verbatim stack mentions + team-shape signals from careers pages>

--- Blog: <URL> ---
<key passages from engineering blog posts>

--- GitHub: <org URL> ---
<observable signal: languages, repo names, commit cadence>

==== COMPETITOR / PEER SURFACES ====

--- Peer: <name> (<URL>) ---
<extracted signal from each peer's product / docs / pricing>

==== EXTERNAL RESEARCH ====

--- Source: <URL or title> ---
<key data points and quotes from Exa / Parallel queries>

==== COST CALIBRATION ====

<full text of product-build-cost-calibration.md so the audit can
verify every cited row>
```

The cost-calibration file is folded into the bundle so the audit can
verify every "(per calibration §X)" cite resolves to an actual row.

**Lint/audit relationship** identical to first-pass-diligence: the
deterministic lint catches surface-form hallucinations with regex; the
LLM-judge audit catches subtler claims not anchored in the bundle.
Both gates must pass before publish.

---

## Step 5: Create the Notion Page

**Select the final draft.** Before writing to Notion:

```bash
if [ -f /tmp/teardown_draft.normalized.md ]; then
  FINAL_DRAFT=/tmp/teardown_draft.normalized.md
else
  FINAL_DRAFT=/tmp/teardown_draft.md
fi
echo "publishing from: $FINAL_DRAFT"
```

Create the page in the Notes database
(`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) via `notion-create-pages`:

```
parent: { data_source_id: "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" }
pages: [{
  properties: {
    "Name": "[COMPANY NAME]: Product Build Teardown — MM.DD.YYYY",
    "Category": "Diligence",
    "Opportunity": "[Opportunity page URL]"
  },
  content: "[Full analysis content in Notion-flavored markdown]"
}]
```

**Title format is mandatory:** `[COMPANY NAME]: Product Build Teardown
— MM.DD.YYYY`. Use the colon (per Tom's note-title rule) and an en
dash before the date. Company name as Tom writes it (e.g., "Upskill",
not "upskill").

Save the resulting Notion page URL — referenced in the PDF and Slack
alert.

### Set the Claude Logo Icon

After creation, set the page icon to `:claude-color:` via
`notion-update-page`:

```
notion-update-page({
  page_id: "<newly created page ID>",
  command: "update_properties",
  properties: {},
  content_updates: [],
  icon: ":claude-color:"
})
```

---

## Step 6: Build the PDF (matches first-pass-diligence formatting EXACTLY)

**Start from the canonical template** at
`/Users/tomseo/.claude/skills/shared-references/pdf_builder_template.py`.
Copy it to the outputs directory and fill in the four CUSTOMIZE blocks
at the top. Do NOT write a PDF script from scratch.

`pip install reportlab --break-system-packages -q` first.

**Read the formatting spec** at
`/Users/tomseo/.claude/skills/shared-references/long-form-pdf-spec.md`
**in full** — this is the authoritative format for fonts, sizes,
leading, spacing, table styles, citation rendering, footnotes, page
numbers, header structure. The teardown PDF must be visually
indistinguishable from a first-pass-diligence PDF in every respect
except the title line.

Specifically that means:

- **Page size:** US Letter, 0.85" L/R margins, 0.75" T/B margins
- **Page numbers:** bottom centered, 8pt Helvetica black, via
  `onFirstPage` / `onLaterPages` canvas callbacks
- **Fonts:** Helvetica family throughout (Helvetica, Helvetica-Bold,
  Helvetica-Oblique, Helvetica-BoldOblique)
- **Title (14pt bold):** `[COMPANY NAME]: Product Build Teardown`
- **Subtitle (9pt italic, #333333):** `[Date] | Notion` where "Notion"
  is a clickable hyperlink to the Notion page URL
- **H1 (12pt bold underlined):** section headers (Context, Product
  Anatomy, Delivery Mechanism, etc.)
- **H2 (11pt bold):** subsection headers
- **Body (10.5pt justified):** standard body text
- **Citation rendering:** `[N]` inline markers render as superscript
  with the `\s?[N]` consuming whitespace regex (see
  long-form-pdf-spec.md for the exact regex); `^N` footnote-definition
  lines render as 8.5pt with hanging indent
- **Tables:** identical TableStyle to first-pass — `#E0E0E0` header
  row, `#F5F5F5` alternating bands, `#E0E0E0` grid lines, 0.5pt grid,
  9pt cell text, `Spacer(1, 10)` after every Table flowable
- **No horizontal rules between sections** — H1 `spaceBefore`
  whitespace only
- **Title and subtitle as flowables**, not canvas callbacks (canvas
  callbacks repeat on every page; title must appear once)

**Apply the Notion API encoding artifact cleanup** before parsing the
content extracted from the `notion-fetch` result — same rules as
first-pass-diligence Step 5. Skipping these produces literal `\n`
strings, `\$` artifacts, and escaped HTML table tags in the PDF.

Save the PDF to
`/tmp/[Company]_Product_Build_Teardown_MM.DD.YYYY.pdf` (replace spaces
in company name with underscores). Do NOT save to iCloud Downloads or
`~/Downloads` — Drive upload in Step 7 is the permanent artifact;
`/tmp/` is staging only.

---

## Step 7: Upload to Drive, Link in Notion, Send Alert

### 7a. Upload PDF to Google Drive and Link in Notion

Use the Apps Script Drive endpoint (see
`/Users/tomseo/.claude/skills/shared-references/drive-upload.md`).
**MANDATORY — always create a NEW Drive file. Never overwrite an
existing one in place.** Reason: Notion caches PDF previews keyed on
file ID; a fresh file ID forces a fresh preview.

1. **Create / reuse company subfolder** under the Diligence root
   (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`) via the Apps Script
   `createFolder` action. Idempotent.
2. **Upload the PDF** as a new file (action `upload`, passing the
   subfolder ID). Save the returned `url` field.

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"
DILIGENCE_ROOT = "1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d"

folder_resp = requests.post(DRIVE_URL, json={
    "action": "createFolder",
    "folderName": "<COMPANY_NAME>",
    "parentFolderId": DILIGENCE_ROOT
}, allow_redirects=True, timeout=60)
subfolder_id = folder_resp.json()["folderId"]

with open(pdf_path, 'rb') as f:
    pdf_b64 = base64.b64encode(f.read()).decode('utf-8')

upload_resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": "[Company]_Product_Build_Teardown_MM.DD.YYYY.pdf",
    "fileBase64": pdf_b64,
    "mimeType": "application/pdf",
    "folderId": subfolder_id
}, allow_redirects=True, timeout=120)
file_url = upload_resp.json()["url"]
```

**Link the PDF in two places on the Opportunity page** (matches
first-pass-diligence Step 6a):

1. **Page body** — append a single-line entry to the
   `## 📎 Diligence Materials` section via `notion-update-page` with
   `update_content`:
   ```
   - [**[Company]_Product_Build_Teardown_MM.DD.YYYY.pdf**](https://drive.google.com/file/d/<fileId>/view) — Claude product build teardown, [date]
   ```

2. **Diligence Materials Files property field — MANDATORY:** shell out
   to the public-API helper:
   ```bash
   python3 ~/.claude/scripts/notion_files_property.py \
       --page-id <opportunity_page_id> \
       --prop "Diligence Materials" \
       --url "<drive_url>" \
       --label "<Company>_Product_Build_Teardown.pdf"
   ```
   Exit 0 = ok. **Verify** the write took by re-fetching the page and
   confirming the URL is in the files array — never trust the 200
   response alone (Factir 2026-05-15 showed silent write failures).

### 7b. Send the Slack alert

Read `send-alert/SKILL.md` for delivery conventions. Send as **exactly
3 lines** (4 if there's an audit caveat). GFM link syntax, never Slack
mrkdwn.

Template:

```
🔧 <u>**Product Build Teardown: [COMPANY_NAME](OPP_URL) ([PDF](PDF_URL))**</u>
ONE_LINER_SUMMARY
💬 Reply in thread with any takeaways for next time.
```

Conventions:
- **Line 1:** `🔧` (wrench) emoji outside the underline+bold wrapper,
  then the title with two links. Company name links to the Notion
  Opportunity URL; literal `PDF` (in parens) links to the Drive PDF.
- **Line 2:** 1–2 sentences. Lead with the strongest signal — typically
  the moat read or the killshot most worth tracking.
- **Line 3:** Literal `💬 Reply in thread with any takeaways for next
  time.` Static — this is what `claude-alerts-listener` keys on.

If the lint or audit surfaced findings that publish-proceeded with
caveats, append a fourth line starting with `⚠️ ` and naming the
count + category. If the PDF upload failed and only the Notion analysis
exists, replace `([PDF](PDF_URL))` with `([analysis](NOTION_URL))`.

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
