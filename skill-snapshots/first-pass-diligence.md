---
name: first-pass-diligence
description: >
  Run a first-pass diligence analysis on a pipeline opportunity from the Notion CRM.
  Pulls all available context (opportunity page, call notes, diligence materials, investment
  memos), evaluates the company against Tom's Inverted Lens frameworks, conducts web research
  to enrich open questions with data, and produces three outputs: (1) a Notion page with the
  full analysis as the living artifact, (2) a formatted PDF, and (3) a Signal alert via Beeper
  with the Notion link. Trigger whenever Tom says "first pass", "first-pass diligence",
  "run diligence on [company]", "diligence analysis on [company]", "first pass on [company]",
  "analyze [company]", "diligence [company]", or any variant requesting an initial structured
  diligence analysis on a named opportunity. Also trigger when Tom says "run the diligence
  skill on X" or "do a first pass on the Lex opportunity" or similar. Always trigger inline —
  no confirmation needed before acting.
---

# First-Pass Diligence

Generate a comprehensive first-pass diligence analysis for a pipeline opportunity. This skill
produces an evidence-grounded, framework-driven analysis that Tom can use to decide whether to
proceed to deeper diligence. The analysis is deliberately opinionated — it should surface yellow
flags, contradictions between founder claims and independent findings, and gaps in the evidence
base, rather than presenting a sanitized summary.

---

## Step 1: Gather All Available Context

Before writing anything, assemble the full evidence base. Completeness matters — the analysis
quality is directly proportional to how much source material you have.

### 1a. Fetch the Notion Opportunity

Search for the company in the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`) using
`notion-search`. Fetch the full opportunity page with `notion-fetch`. Extract:

- All properties (stage, round details, status, description, scores, HQ, founders)
- Full page body content (summary, team bios, round context, source context)
- All linked Notes (the ✍️ Notes relation) — fetch each linked note page
- Diligence Materials property — note any Google Drive file IDs or URLs

### 1b. Fetch All Linked Notes

These are typically call notes, research threads, backchannel references. Fetch every URL in
the Notes relation. Pay close attention to:

- **Founder meeting / call notes** — the richest source of founder signal
- **Backchannel / reference notes** — third-party perspective on the team
- **Prior Claude analysis** — earlier research or framework mapping

### 1c. Fetch Diligence Materials

The Diligence Materials field may contain four types of sources. Identify each by its URL
format and use the corresponding access method:

**Type 1: Google Drive PDFs** (`drive.google.com/file/d/{FILE_ID}/...`)
Use the PDF MCP tool `read_pdf_bytes` with the download URL format:
`https://drive.google.com/uc?export=download&id={FILE_ID}`. This works without Chrome
authentication for files shared to Tom's account.

**Type 2: Google Docs** (`docs.google.com/document/d/{DOC_ID}/...`)
Use `google_drive_fetch` with the document ID.

**Type 3: Notion-hosted attachments** (`attachment:{UUID}:{filename}.pdf`)
These are PDFs uploaded directly to Notion and cannot be accessed via `read_pdf_bytes`
(requires Notion auth cookies). Use the Chrome-based extraction pipeline:

1. Navigate Chrome to the opportunity's Notion page (if not already there).
2. Execute JavaScript to call Notion's internal API and get a signed URL:
   ```javascript
   (async () => {
     const resp = await fetch('/api/v3/getSignedFileUrls', {
       method: 'POST',
       headers: { 'Content-Type': 'application/json' },
       body: JSON.stringify({
         urls: [{
           url: 'attachment:{UUID}:{filename}',
           permissionRecord: { table: 'block', id: '{PAGE_ID_WITH_DASHES}' }
         }]
       })
     });
     const data = await resp.json();
     window.location.href = data.signedUrls[0];
   })()
   ```
3. Once Chrome navigates to the `file.notion.so` PDF URL, load pdf.js from CDN and extract
   text content:
   ```javascript
   (async () => {
     const script = document.createElement('script');
     script.src = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js';
     document.head.appendChild(script);
     await new Promise(r => { script.onload = r; });
     pdfjsLib.GlobalWorkerOptions.workerSrc =
       'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
     const resp = await fetch(window.location.href);
     const buf = await resp.arrayBuffer();
     const pdf = await pdfjsLib.getDocument({ data: buf }).promise;
     let text = '';
     for (let i = 1; i <= pdf.numPages; i++) {
       const page = await pdf.getPage(i);
       const tc = await page.getTextContent();
       text += '\n--- Page ' + i + ' ---\n' + tc.items.map(x => x.str).join(' ');
     }
     return text;
   })()
   ```
4. Navigate Chrome back to the Notion page afterward if you need to process additional
   attachments.

**Important:** This Chrome-based pipeline requires Chrome access via the Claude in Chrome
MCP. If Chrome is unavailable (e.g., scheduled task without Chrome), note the Notion
attachment in the analysis as "material available but not readable in current environment"
and flag it as a gap in the Sources section.

**Type 4: DocSend links** (`docsend.com/view/...`)
Read the `docsend-to-pdf` skill (`**/docsend-to-pdf/SKILL.md`) and follow its instructions
to convert the DocSend document to a local PDF. Once converted, read the resulting PDF with
`read_pdf_bytes` using the local file path. DocSend data rooms (multi-document links) should
have each document converted individually.

### 1d. Read Tom's Investment Framework Memos

List all files in the Google Drive investment memos folder (`1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`)
using `google_drive_search` or by listing the folder contents. Then read **every memo** in the
folder using the appropriate access method (Google Drive fetch for Docs, `read_pdf_bytes` with
the download URL format `https://drive.google.com/uc?export=download&id={FILE_ID}` for PDFs).

**Why read all memos, not a subset:** The frameworks Tom applies evolve as the portfolio grows.
New memos may introduce new frameworks, refine existing ones, or reveal cross-portfolio patterns
that only emerge across the full body of work. Reading every memo ensures the analysis reflects
the complete and current Inverted Lens vocabulary — not a stale snapshot. This is the mechanism
by which the Inverted Lens stays current as Tom writes new memos.

From the memos, extract the recurring investment frameworks Tom has applied across the portfolio.
These are the analytical lenses he uses to evaluate founder quality, business model durability,
and structural advantages. Typical framework themes include (but are not limited to): data assets
as drivers of enduring value, forward-deployed founder orientation, intentionality and intellectual
honesty, unbounded end states with focused wedges, structural distribution leverage, capital
efficiency, founder slope over y-intercept, domain expertise without arrogance, and non-transactional
relationship-building style. But **derive the actual framework menu from the memos themselves** —
do not rely on this illustrative list as a substitute for reading the source material. The memos
are the source of truth.

Select the frameworks most relevant to the specific opportunity — not all will apply with equal
force to every deal. The selection should feel natural and evidence-driven, not forced. Evaluate
each selected framework against the opportunity, with attention to whether the founders are
proactively surfacing these themes (unprompted) vs. responding to investor questioning.

---

## Step 2: Conduct Web Research

Before drafting, run targeted web searches to enrich the analysis with independent data points.
The goal is to verify founder claims, size markets, and ground open questions in real numbers
rather than leaving them as generic unknowns. Typical research areas:

- **Market sizing** — TAM/SAM data, growth rates, comparable market benchmarks
- **Regulatory environment** — licensing timelines, requirements, costs (compare to founder
  representations)
- **Competitive landscape** — comparable company metrics, loss rates, unit economics where
  public
- **Customer population** — sizing the target segment with demographic and economic data
- **Technology landscape** — infrastructure maturity, data availability, API ecosystem

When research findings contradict founder claims, flag this explicitly. The gap between what
the founder says and what independent research shows is one of the most valuable outputs of
this analysis.

Every data point cited in the final output must be traceable to a source. Collect URLs as you
research — they will populate the Sources section.

---

## Step 3: Draft the Analysis

Write the analysis in a professional, analytical voice. The tone should be factual and objective —
not promotional, not dismissive. Each section should contain substantive paragraphs (not one-liners)
that convey ideas with high fidelity. When you lack sufficient context to make an informed assessment
on a topic, say so explicitly rather than filling space with generic language.

### Section Order

The analysis follows this exact structure:

**Framework Mapping — Inverted Lens**
Evaluate the opportunity against the Inverted Lens frameworks that are most relevant to this
specific deal. The full menu is documented in Step 1d — select whichever apply with force
based on the evidence, not a fixed set.

Open with a brief preamble paragraph (2-4 sentences) explaining which frameworks were selected
and why, noting that the evaluation considers both structural fit and whether the founders are
proactively surfacing these themes unprompted. This orients the reader before the subsections.

For each selected framework, write a subsection with a descriptive header (e.g., "The Data Asset
as the Driver of Enduring Value" — not numbered, not abbreviated). Each framework subsection
must contain **multiple substantive paragraphs** (3-5 paragraphs minimum) that:
- Develop the analytical argument with evidence from the diligence record
- Cross-reference how the framework has been applied in other Inverted portfolio memos (e.g.,
  "Just as Rengo compounds a data asset for each PE firm through structured portfolio monitoring
  data..." or "The parallel to the Inverted portfolio is direct...") — this grounding in the
  portfolio vocabulary is essential
- Surface tensions, caveats, and honest assessments of where the mapping is strong vs. weak
- Note whether the founder is proactively raising the theme vs. responding to investor questioning

**NEVER** use scorecard-style assessment labels like `**Assessment: STRONG FIT**` or
`**Assessment: MODERATE-TO-STRONG**`. These reduce nuanced analysis to a rubric grade and strip
out the reasoning that makes the section valuable. The assessment should emerge from the prose,
not be stamped at the top of each subsection.

End the full section with a **Summary Assessment** subsection (2 paragraphs minimum) that weighs
the frameworks against each other, identifies where the mapping is strongest and weakest, and
surfaces the most important signal across all frameworks taken together.

**Need to Believe**
Working backwards from what must be true for this to be a compelling Inverted Capital investment.
Each item is a **bullet** (not numbered) with a bold label and a multi-sentence explanation. These
are necessary conditions — if any breaks, the thesis is materially impaired.

**NEVER use numbered lists** (1., 2., 3.) for Need to Believe items. Use bullet points only.

End with a synthesis paragraph identifying the highest-priority beliefs to diligence — which ones
have the most binary outcomes and the least existing evidence.

**1. Market**
Three subsections: Market Size & Dynamics, Competitive Landscape, Regulatory Environment.

Market Size & Dynamics should use **flowing prose with data woven in naturally** — not a bullet-
point dump of statistics. Data points should be in service of an argument, not presented as a
standalone list. A dedicated "Market Data Points" bullet list at the end of the subsection is
acceptable for reference, but the analytical prose must come first.

Competitive Landscape should analyze competitive dynamics through **prose paragraphs**, not
capability/checkmark matrices. NEVER use comparison tables with ✓/✗ or ✓✓ symbols across
capability columns — these strip out the nuance of competitive positioning. Instead, explain in
prose why specific competitors are or aren't positioned to compete, using the founder's own
reasoning from the calls where relevant. A data table for competitor funding/stage/metrics is
fine; a feature-comparison grid is not.

Regulatory Environment should compare founder representations against independent research and
flag any gaps explicitly, using a **bold "Independent Finding"** callout format when research
contradicts the founder's timeline, cost, or complexity estimates.

Each subsection ends with ***Open Questions*** in bold-italic.

**2. Product**
Three subsections: Product Architecture & Roadmap, Underwriting/Data/Technical Strategy (adapt to
the company), Technical Differentiation. End each with Open Questions.

**3. Go-to-Market & Distribution**
Three subsections: Initial Customer Acquisition (is there an "unfair" advantage?), Steady-State
Scalable Distribution, Where Do Customers "Exist" Today? End each with Open Questions.

**4. Operating Model**
Three subsections: Pricing Model Hypotheses, Capital Efficiency, Long-Term Steady-State Unit
Economics. End each with Open Questions.

Pricing Model Hypotheses should be grounded in **evidence from the diligence record** — actual
pricing mentioned in the one-pager, deck, or calls, comparable pricing from competitors found in
research, and the specific revenue mechanics of the company's product. NEVER present speculative
hypothetical pricing tiers (e.g., "Hypothesis 1: Per-Volume", "Hypothesis 2: Per-Seat") that are
not traceable to any source material. If the company hasn't disclosed pricing, say so, then
analyze what comparable companies charge and what the product's value proposition implies about
willingness-to-pay.

For the **Capital Efficiency** subsection, the central question is: *can this business operate
capital efficiently at inception — and thus justify a true pre-seed round — or is there something
structural about the business that requires a large round out of the gate?* Factors that could
necessitate a larger initial raise include: building in a consensus or well-funded category where
mindshare must be established early; a product or customer profile where counterparties (enterprise
buyers, regulated partners, institutional clients) want to see material capital on the balance
sheet before engaging; infrastructure or licensing costs that cannot be deferred; or a GTM motion
that is inherently capital-intensive from day one (e.g., sales-led into large enterprises, physical
operations, regulated product deployment). Assess whether the business as scoped is genuinely
compatible with a lean initial raise, or whether the pre-seed framing understates what it will
actually take to get to a meaningful proof point.

Long-Term Steady-State Unit Economics should model what the business looks like at maturity —
cost of capital, loss rates, operating costs, revenue per customer — and identify the key unknowns
that make the model speculative at this stage. NEVER fabricate burn rate analyses with monthly
figures that aren't traceable to the diligence record. If the company hasn't disclosed burn rate
or team size, note the gap rather than inventing numbers.

**5. Team**
Three subsections: Founder-Market Fit, Team Dynamics & Composition, Founder Evaluation Through the
Inverted Lens. Include a team experience table.

The **Founder Evaluation Through the Inverted Lens** subsection is required — do not skip it.
This evaluates the founders specifically against the Inverted frameworks (intentionality and rigor,
intellectual honesty, data asset as moat driver, slope over y-intercept, etc.) with evidence from
the diligence record. It should reference specific moments from the calls, how the pitch evolved
between conversations, and where intellectual honesty has or hasn't been tested.

**6. Risks**
Isolate the most plausible failure modes, framed as specific mechanisms by which the thesis breaks.

Each risk gets a numbered subsection with a **descriptive title** that names both the risk category
and the specific mechanism (e.g., "6.1 Platform Dependency — Rain as Single Point of Failure" or
"6.2 Underwriting Model — Unproven Credit Engine in a Data-Scarce Market"). Do NOT append
probability, severity, or impact labels to the title — no "(High Impact, High Probability)", no
"(Fatal/Kill Shot)", no "(Medium-High)". The title names the risk; the prose and evidence below
it convey severity. Use the pre-mortem
skill's failure mode taxonomy: Category/Market Formation Risk, Product/Technical Loop Risk,
GTM/Commercial Execution Risk, Team/Organizational Fragility Risk, and a fifth Thesis-Specific
Risk if applicable.

Each risk has two components:
1. An **analytical paragraph** (or multiple paragraphs) grounding the risk in specific evidence
   from the diligence record — data points, founder claims vs. independent findings, comparable
   company metrics, specific dollar amounts.
2. A ***Failure mode:*** paragraph in bold-italic describing the precise mechanism by which it
   breaks, with quantified impact where possible. This paragraph should read as a specific
   narrative of how the thesis collapses — not an abstract description.

**CRITICAL — formatting rules for the Risks section:**
- Do **NOT** use subjective probability/severity labels (e.g., "HIGH PROBABILITY, HIGH IMPACT",
  "MODERATE (65%)", "Medium-High / Fatal"). These are hand-wavy without data and make the
  analysis look like a generic risk matrix instead of investor-grade thinking.
- Do **NOT** create a Risk Summary table with "Probability" or "Impact" columns.
- **DO** end with a Risk Summary table with exactly these columns: **Failure Mode | Key Evidence |
  Kill Shot?** — where Kill Shot uses "Yes — independent path to zero" or "Conditional —
  compounds other risks / slows thesis" (not probability labels).
- Anchor every risk in **specific evidence** that makes it material, not subjective assessments
  of likelihood.

**7. Path to Next Round**
Backsolve what traction the company needs to unlock the next financing round. Four subsections:

*Benchmarks — Comps from Deal Flow:* Search the Notion Notes database for Deal Digest entries
(Category = Research, Name contains "Deal Digest"). Pull relevant comps — both sector-fit (same
industry) and stage-fit (comparable ARR/valuation regardless of sector). Sort by stage earliest to
latest, weight more recent data points higher. Do not mislabel companies — if the product category
is not stated in the digest, omit it rather than guessing. Present as a table with columns:
Company | Stage | ARR / Revenue | Valuation | Lead | Notes.

*Implied Parameters for Next Round:* Based on comps, define two scenarios for the next round.
Scenario A should be the round achievable from the current raise (e.g., seed from pre-seed).
Scenario B should be the larger round that likely requires an intermediate raise first (e.g.,
Series A). **Name each scenario with its round type** (e.g., "Scenario A — Seed: Pedigree + Early
Signal ($8-12M raise at $40-60M post)" not just "Scenario A"). Each scenario should describe the
archetype — what the company looks like at that stage, what metrics are required, what investors
would need to see.

*Revenue Build — Can [Raise Amount] Get [Company] to the Next Round?:* The central analytical
exercise. Start with **capital allocation**: how much of the raise funds the product/portfolio vs.
operations vs. key hires vs. licensing/regulatory costs. Then build the **per-customer revenue
math** — this must be specific to the business model (e.g., for consumer credit: # of active
accounts × revenue per customer including annual fee + interchange; for enterprise SaaS: # of
customers × ACV). Then present a **chronologically ordered traction requirements table** for
Scenario A, with columns: Milestone | Target | Timeline | Rationale. The annualized revenue
run-rate is the anchor milestone; every other milestone (licensing, hires, loss rates, warehouse
lines, etc.) should connect back to what's required to hit the revenue target. Reconcile
explicitly: does the raise provide enough runway to reach these milestones?

If Scenario B is not reachable from the current raise, state this clearly and explain the
intermediate step required. Include a separate, shorter traction requirements table for Scenario B.

*What Breaks the Path:* Identify the tightest sequencing constraint and how delays cascade. This
should be a prose analysis (1-2 paragraphs) that traces the critical path — what has to happen
first, what depends on it, and how a slip at the bottleneck compresses everything downstream.

**Suggested Additional Analysis**
Bulleted list of specific follow-up diligence workstreams.

**Sources**
Grouped into exactly two sections. The overriding principle is: **every source should be a
clickable link wherever possible.** During Step 1 you fetched Notion pages, Drive files, and
web sources — you have the URLs. Use them. A source entry without a hyperlink when one was
available is a miss.

1. **Internal Materials** — every entry MUST be a clickable hyperlink to the Notion page or
   Google Drive file it references. You fetched these URLs in Step 1; link them here. Format:
   `[Document Name](https://www.notion.so/<page-id>)` with a brief annotation after the link.
   For example: `[Reed & Nolan Intro Call — March 19, 2026](https://www.notion.so/abc123) —
   Founder backgrounds, solution architecture, GTM strategy`. For Google Drive files, link
   directly: `[Clusia Memo](https://drive.google.com/file/d/<id>/view)`.

2. **External References** — every web source researched in Step 2 MUST include the URL.
   Format: `[Source Name](URL) — brief relevance annotation and key data contributed`. Group
   logically (Founder Backgrounds, Competitive Landscape, Market Data, Regulatory, etc.) with
   subheadings. For backchannel feedback from specific people, link to the Notion feedback note
   if one exists.

Include a **Key Data Points Referenced** sub-section where every market statistic or external
data point cited in the analysis is linked to its source URL.

### Writing Guidance

- **Be specific, not generic.** "Execution risk" is not useful. "Neither founder has built
  underwriting models for Mexican consumers" is useful.
- **Ground claims in evidence.** Every assertion should trace to a call note, a deck, a data point,
  or a research finding.
- **Flag contradictions.** When founder claims diverge from independent research, present both
  with a "yellow flag" notation.
- **Include visual artifacts.** For data-dense topics (market sizing, competitor comparison,
  regulatory findings, team experience), create tables. Tables are more information-dense than
  prose and help Tom scan quickly.
- **Open Questions should be answerable.** If you can take a stab at answering an open question
  with web research, do so. Only leave questions open when they genuinely require founder input
  or proprietary data.
- **Paragraphs must be substantive.** Each paragraph should be 4-6 sentences minimum, developing
  a single analytical thread with evidence and caveats. One-to-two sentence paragraphs lack the
  nuance and fidelity this analysis demands. If a paragraph is too short, it either needs more
  evidence or should be merged into an adjacent paragraph.
- **Prose over lists.** Default to flowing prose with data woven into the argument. Use bullet
  lists only for Open Questions, Need to Believe items, and Suggested Additional Analysis. For
  everything else — market dynamics, competitive positioning, product analysis, team assessment,
  risk analysis — write in paragraphs. If you find yourself building a bulleted list of observations,
  convert it to prose.

### Common Deviations — Do NOT Do These

The following are specific anti-patterns observed in prior outputs that deviate from this skill.
If you catch yourself doing any of these, stop and correct:

1. **Scorecard labels on frameworks** — writing `**Assessment: STRONG FIT**` instead of developing
   the assessment through prose. The assessment should emerge from the argument, not be stamped
   as a header.
2. **Numbered Need to Believe items** — using 1., 2., 3. instead of bullet points. Always use
   bullets with bold labels.
3. **Probability/severity risk labels anywhere** — writing "HIGH PROBABILITY, HIGH IMPACT",
   "MODERATE (65%)", or "(High Impact, Medium Probability)" in the Risks section — whether in
   subsection titles, body text, or summary tables. The skill explicitly prohibits this. The
   title names the risk mechanism; the prose conveys severity through evidence. Use the "Kill
   Shot?" framework in the summary table instead of probability/impact grids.
4. **Checkmark capability matrices** — using ✓/✗ tables to compare competitors. Analyze
   competitive dynamics in prose.
5. **Speculative pricing hypotheticals** — inventing pricing tiers not grounded in the diligence
   record. Ground pricing analysis in actual evidence.
6. **Fabricated burn rate numbers** — creating monthly burn estimates when no team size, salary,
   or operating cost data exists in the record. Note the gap instead.
7. **Missing "Founder Evaluation Through the Inverted Lens"** — skipping the third Team
   subsection. This is required.
8. **Generic Path to Next Round** — not following the four-subsection structure (Benchmarks,
   Implied Parameters with named scenarios, Revenue Build with capital allocation and per-
   customer math, What Breaks the Path). Each subsection has specific required content.
9. **Wrong Sources grouping** — using ad hoc categories instead of "Internal Materials" and
   "External References" with "Key Data Points Referenced."
10. **Thin paragraphs** — writing 1-2 sentence paragraphs that state conclusions without
    developing the reasoning. Every major claim needs evidence and context.
11. **Broken tables in Notion** — pasting standard markdown pipe tables that render with visible
    separator rows (`|---|---|`) as literal text. See Step 4's "Notion Table Formatting" section
    for how to handle this.
12. **Unlinkable sources** — listing source names as plain text when you have the Notion page URL
    or web URL available from the evidence-gathering phase. Every source entry should be a
    clickable hyperlink.
13. **Backslash-escaped special characters in Notion content** — writing `\\~2 yrs` or `\\$50M`
    instead of `~2 yrs` or `$50M`. Notion does not require backslash escaping of `~`, `$`, or
    other special characters. Write them directly — backslashes render as literal text.

---

## Step 4: Create the Notion Page

### Notion Table Formatting

Notion's markdown parser is finicky with tables. Standard markdown tables with separator rows
(|---|---|) often render the separator as literal text instead of being parsed as a table
delimiter. To avoid broken tables in Notion:

- **Use simple tables** with the Notion enhanced markdown `<table>` block syntax when available.
  If the Notion MCP supports it, prefer that over raw pipe tables.
- **If using pipe tables**, ensure each row has exactly the same number of columns. The separator
  row must use at least three dashes per column (`|---|---|`) with no leading/trailing whitespace
  on the line. Some Notion environments require the separator row to use exactly `---` (three
  dashes, no more).
- **Test table rendering** after creating the page. If tables render as raw text with visible
  pipe characters and separator rows, replace the table content using `notion-update-page` with
  `update_content` — try removing the separator row entirely, or reformatting as a bulleted
  summary if the table refuses to render.
- **Fallback**: If tables consistently fail to render in Notion's markdown, convert the data into
  bold-label prose format instead (e.g., "**Scrut** — Seed+, $13M ARR, $65M–$100M valuation,
  led by Boldstart"). A clean prose summary is always better than a broken table.

### Character Escaping in Notion Content

When writing the analysis content to Notion, do NOT backslash-escape special characters like
`~`, `$`, or other markdown-adjacent symbols. Notion's markdown parser does not require these
escapes, and they render as literal backslashes (e.g., `\\~2 yrs` instead of `~2 yrs`). Write
tildes, dollar signs, and other special characters directly without any preceding backslash.

Create a new page in the Notes database (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) using
`notion-create-pages`:

```
parent: { data_source_id: "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" }
pages: [{
  properties: {
    "Name": "Claude: [Company Name] First-Pass Diligence — MM.DD.YYYY",
    "Category": "Diligence",
    "Opportunity": "[Opportunity page URL]"
  },
  content: "[Full analysis content in Notion-flavored markdown]"
}]
```

Save the resulting Notion page URL — it will be referenced in the PDF and the Signal alert.

### 4b. Set the Claude Logo Icon

Immediately after creating the page — before moving to Step 5 — set the page icon to the custom
Claude logo emoji. This is what lets Tom visually distinguish Claude-generated analysis from his
own notes in the Notion list view, so skipping it defeats the purpose of having a structured
notes system.

Call `notion-update-page` with the icon field set to `:claude-color:`:

```
notion-update-page({
  page_id: "<newly created page ID>",
  command: "update_properties",
  properties: {},
  content_updates: [],
  icon: ":claude-color:"
})
```

The colon-wrapped name format is the only format that works with the Notion MCP for custom emojis.
Do not use `custom_emoji::` prefix, raw UUIDs, or JSON objects — those will silently fail or set
the wrong icon. Use `:claude-color:` exactly as shown above.

---

## Step 5: Build the PDF

**Start from the canonical template** at
`/Users/tomseo/.claude/skills/first-pass-diligence/pdf_builder_template.py` — copy it to the
outputs directory and fill in the four CUSTOMIZE blocks at the top. Do NOT write a PDF script
from scratch; the template already contains all known encoding fixes and formatting logic.

Install reportlab first: `pip install reportlab --break-system-packages -q`

**CRITICAL — Notion API encoding artifacts:** The Notion MCP returns content as a JSON string
where certain characters are double-escaped. You MUST apply these three preprocessing lines to
the raw markdown content extracted from the tool-result JSON **before** passing it to the parser:

```python
md_content = md_content.replace('\\n', '\n')   # literal \n two-char → real newline
md_content = md_content.replace('\\"', '"')     # literal \" → real quote
md_content = re.sub(r'\\+\$', '$', md_content) # \\$ or \\\$ → $
md_content = re.sub(r'\\+~', '~', md_content)  # \\~ → ~ (tildes)
```

Skipping these lines produces a broken PDF with literal `\n` characters everywhere, raw `\$50M`
strings, and HTML table tags (`<table>`, `<tr>`, `<td>`) appearing as body text instead of
rendered tables.

**HTML table format:** Notion enhanced markdown uses `<table header-row="true"><tr><td>…</td></tr></table>`
syntax for tables, not standard markdown pipe tables. The canonical template's `parse_content()`
function handles both formats. If you write a custom parser, it must handle both.

Read the formatting spec at `references/pdf-format-spec.md` for the exact style definitions.

**Post-table / post-chart spacing:** After every `Table` flowable (and any chart or visual
exhibit), insert a `Spacer(1, 10)` to enforce 10pt of whitespace before the next element.
This prevents body text or headers from crowding the bottom edge of tables and exhibits.

**Page numbers:** Include page numbers on every page — bottom centered, 8pt Helvetica, black.
Use reportlab's `onFirstPage` and `onLaterPages` callbacks with `canvas.drawCentredString()`.

**Markdown-to-PDF text conversion:** When converting markdown content to reportlab Paragraphs,
properly convert all markdown formatting to reportlab XML tags. Do NOT allow raw markdown
asterisks to appear in the rendered PDF. Write a `clean_text()` helper that handles:
- `***text***` → `<b><i>text</i></b>`
- `**text**` → `<b>text</b>`
- `*text*` → `<i>text</i>`
- `[text](url)` → `<a href="url" color="black"><u>text</u></a>`
Apply `clean_text()` to every string before passing it to `Paragraph()`.

**CRITICAL — XML escaping order in `clean_text()`:** Reportlab's `Paragraph` uses XML-like tags
(`<b>`, `<i>`, `<a>`) for inline formatting. If you escape `<` and `>` to `&lt;`/`&gt;` before
converting markdown to reportlab tags, the tags themselves get escaped and render as literal text
(e.g., `<b>Sponsor banks</b>` visible in the PDF). The correct order is:
1. First, escape `&` to `&amp;` (to avoid breaking XML entities).
2. Then, escape `<` and `>` to `&lt;`/`&gt;` (to prevent user content from being parsed as XML).
3. **Then** convert markdown formatting (`**bold**`, `*italic*`, `[links](url)`) to reportlab
   XML tags (`<b>`, `<i>`, `<a>`).
This ensures user content is safely escaped while the formatting tags are valid XML that
reportlab can render.

**Title and subtitle as flowables, not page template:** The title and subtitle must be added as
`Paragraph` flowables at the top of the story — NOT drawn via `onFirstPage`/`onLaterPages`
canvas callbacks. Canvas callbacks execute on every page (or the first page), which causes the
header to repeat. Page numbers should still use canvas callbacks (`onFirstPage` and
`onLaterPages`), but the title/subtitle should be regular flowables that appear once.

Save the PDF to the outputs directory: `[Company]_First_Pass_Diligence_MM.DD.YYYY.pdf`
(replace spaces in company name with underscores).

The PDF header should be:
- **Title (14pt bold):** `[Company Name] — First-Pass Diligence`
- **Subtitle (9pt italic, dark gray):** `[Date] | Notion` (where "Notion" is a clickable
  hyperlink to the Notion page URL)

---

## Step 6: Upload to Drive, Link in Notion, and Send Alert

Upload the PDF to Drive first so the Drive URL is available to include in the alert, then link
it in Notion and send the alert with the URL (no file attachment).

### 6a. Upload PDF to Google Drive and Link in Notion

Upload the generated PDF to the company's opportunity folder on Google Drive, then add the
Drive link to the Diligence Materials field on the Notion opportunity page. This makes the
PDF accessible alongside the other diligence materials (deck, memo, one-pager) in a single place.

**Uploading:** Use the Google Apps Script endpoint for all Drive operations. **NEVER use Zapier** — it is deprecated and unreliable. Read the full reference at `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`.

The workflow is:
1. **Create a company subfolder** under the Diligence root folder (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`) via the Apps Script `createFolder` action. This is idempotent — if the folder already exists, it returns the existing one.
2. **Upload the PDF** into the company subfolder via the Apps Script `upload` action, passing the returned `folderId`.
3. The upload response includes a direct `url` field (e.g. `https://drive.google.com/file/d/<fileId>/view`) — use this directly in the Notion link. No need to search for the file ID after upload.

```python
import requests, base64

DRIVE_URL = "https://script.google.com/macros/s/AKfycbzRPkebxLe-VoJq1UDxUOR8bujyG0T8_rskdmF66lcUYD_JeMh8ODZ6cpeayU61_h8z/exec"
DILIGENCE_ROOT = "1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d"

# 1. Create company subfolder (idempotent)
folder_resp = requests.post(DRIVE_URL, json={
    "action": "createFolder",
    "folderName": "<COMPANY_NAME>",
    "parentFolderId": DILIGENCE_ROOT
}, allow_redirects=True, timeout=60)
folder_result = folder_resp.json()
subfolder_id = folder_result["folderId"]

# 2. Upload PDF
with open(pdf_path, 'rb') as f:
    pdf_b64 = base64.b64encode(f.read()).decode('utf-8')

upload_resp = requests.post(DRIVE_URL, json={
    "action": "upload",
    "fileName": "[Company]_First_Pass_Diligence_MM.DD.YYYY.pdf",
    "fileBase64": pdf_b64,
    "mimeType": "application/pdf",
    "folderId": subfolder_id
}, allow_redirects=True, timeout=120)
upload_result = upload_resp.json()
file_url = upload_result["url"]  # Direct link — use this in Notion
```

**Important:** Python `requests.post(..., allow_redirects=True)` handles Apps Script 302 redirects correctly. `curl -L` does NOT work for POST.

Act autonomously — do not ask for permission. Report what was done in the summary.

**Linking in Notion — two places:**

1. **Page body** — append a single-line entry to the `## 📎 Diligence Materials` section,
   following the materials-handler convention:
   ```
   - [**[Company]_First_Pass_Diligence_MM.DD.YYYY.pdf**](https://drive.google.com/file/d/<fileId>/view) — Claude first-pass diligence analysis, [date]
   ```
   Use `notion-update-page` with `update_content` for this.

2. **Diligence Materials Files property field** — Read the shared reference at
   `/Users/tomseo/.claude/skills/shared-references/add-link-to-diligence-materials.md` and follow its
   instructions to add the Drive file link to the Diligence Materials property on the
   opportunity page. Pass the opportunity page ID, the Drive file URL
   (`https://drive.google.com/file/d/<fileId>/view`), and a display name like
   `[Company]_First_Pass_Diligence.pdf`.

   If Chrome tools are not available at all (no browser session), skip this step — the page
   body link is sufficient and the property field can be updated in the next session where
   Chrome is available.

### 6b. Send the alert

Read the `send-alert` skill (`**/send-alert/SKILL.md`) for the chatID and guardrails. Send
a text-only message (no file attachment) that includes both the Notion page URL and the
Drive URL from step 6a:

```
📋 [Company Name] — First-Pass Diligence Complete

[X] pages. Framework mapping, need to believe, market/product/GTM/operating model/team
analysis with visual tables and enriched research.

Notion: [Notion page URL]
PDF: [Drive file URL from step 6a]

Key flags:
- [Most important yellow flag or finding]
- [Second key finding]
- [Third key finding]
```

Include 3-4 of the most important flags or findings — the things Tom should know about before
even opening the document. These should be specific and punchy, not generic summaries.

---

## Quality Bar

A good first-pass diligence analysis:

- Cites specific evidence from the diligence record for every major claim
- Flags every instance where founder representations diverge from independent findings
- Includes data tables that make information-dense sections scannable
- Grounds open questions in research rather than leaving them as generic placeholders
- Makes the case for and against the investment with equal rigor
- Is explicit about where the evidence base is thin and what additional context would help
- Uses substantive, multi-paragraph prose throughout — not bulleted lists of observations
- Cross-references Inverted portfolio memos in the Framework Mapping section
- Follows the exact Risk Summary table format (Failure Mode | Key Evidence | Kill Shot?)
- Includes all four Path to Next Round subsections with the prescribed analytical content

A bad first-pass diligence analysis reads like a pitch deck summary or a generic VC memo.
If you find yourself restating the company's own narrative without interrogating it, stop
and go back to the source materials.

**Reference output:** The Lex Financial First-Pass Diligence (March 27, 2026) represents the
quality bar for this skill. When in doubt about formatting, depth, or analytical voice, use
that output as the benchmark.
