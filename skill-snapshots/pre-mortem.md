---
name: pre-mortem
description: "Draft a structured pre-mortem for an investment opportunity Tom is actively considering or has committed to. A pre-mortem imagines the investment has already failed and works backward to surface the most plausible failure modes before confirmation bias sets in. Trigger this skill whenever Tom says 'run a pre-mortem', 'pre-mortem this deal', 'stress-test this investment', 'what could go wrong with X', 'devil's advocate on X', 'what breaks the thesis on X', or any variant asking for a forward-looking failure analysis on a specific opportunity. Also trigger when Tom asks to 'pressure-test' or 'stress-test' a deal thesis, even if he doesn't use the word pre-mortem. Always trigger inline — no confirmation needed before acting."
---

# Pre-Mortem Skill

Draft a rigorous, evidence-grounded pre-mortem for an investment opportunity and save it as a linked note in the Notion Notes database. The pre-mortem imagines a future failure and works backward — it is a structured intellectual exercise designed to surface kill shots before capital is deployed, not a refutation of the thesis.

---

## Step 1: Gather Diligence Materials

Before drafting, you need the full evidence base. Do all of the following:

1. **Fetch the Notion Opportunity page** for the company. Use `notion-fetch` on the Opportunity URL or search for it by name in the Opportunities DB (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Extract:
   - All properties (stage, round details, status, scores, open questions)
   - All linked Notes (✍️ Notes relation field) — fetch each one
   - Diligence Materials field — fetch any linked Google Docs

2. **Fetch all linked Notes** — these typically contain call notes, Claude research threads, backchannel references, and framework mapping. Fetch each URL in the Notes relation. Prioritize:
   - Founder meeting notes
   - Backchannel / reference call notes
   - Market research reports
   - Framework mapping / scorecard entries
   - Pre-memo diligence questions

3. **Fetch Diligence Materials** — use `google_drive_fetch` on any Google Doc IDs in the Diligence Materials field. These typically include the investment memo and scratch notes.

4. **Review Tom's investment frameworks** — list all files in the Google Drive investment memos
   folder (`1yqWgJf35SjZdIpFozBRQOX8ympX-gkvO`) and read every memo in the folder. These memos
   are the source of truth for Tom's Inverted Lens frameworks and portfolio vocabulary. Extract
   the recurring analytical lenses (data assets, founder orientation, intentionality, distribution
   leverage, etc.) from the memos themselves rather than relying on a static list — the frameworks
   evolve as the portfolio grows. The pre-mortem should be written in Tom's analytical voice and
   consistent with his framework vocabulary as expressed in the memos.

---

## Step 2: Draft the Pre-Mortem

Write the pre-mortem in Tom's voice: analytical, first-person, evidence-specific. Do not use generic VC risk language. Every failure mode must be grounded in something specific from the diligence record — a quote, a gap, an unresolved question, a competitive data point. See the reference file at `references/writing-guide.md` for detailed style and voice guidance.

### Structure

**Opening Frame** *(1 paragraph)*
Set the hypothetical: "It is [Year+3]. The investment in [Company] returned zero..." Name the three to four load-bearing beliefs the investment rests on. State that the exercise stress-tests each in turn.

**Failure Mode Taxonomy** *(4–5 failure modes)*
For each failure mode, write a titled section with:
- **Probability and Severity label** (e.g., *Probability: Medium-High | Severity: Fatal*)
- **2–3 paragraphs** of analytical prose grounded in the actual diligence record
- Each mode should be *specific* — not "market risk" but the precise mechanism by which the market fails for this company

Use these four canonical failure categories as your framework, then add a fifth if the company has a specific structural risk worth isolating:

1. **Category / Market Formation Risk** — Does the market category exist as a distinct, payable thing? Who owns it if it crystallizes? What incumbent or platform could absorb it?
2. **Product / Technical Loop Risk** — Is the core product loop (the thing that makes the thesis compound) actually built and validated, or is it roadmap? Where does the evidence trail go cold?
3. **GTM / Commercial Execution Risk** — Can the founder(s) sell? To whom, with what motion, at what pace? Is there evidence of a repeatable commercial motion beyond the first design partner?
4. **Team / Organizational Fragility Risk** — What does the company look like when it hits organizational stress — a bad hire, a co-founder gap, a key customer churn? Does the team structure hold?
5. **Thesis-Specific Risk** (if applicable) — A fifth failure mode unique to this deal's central thesis (e.g., the data moat thesis, the regulatory tailwind thesis, the category-definition bet).

**Kill Shot Ranking** *(table)*
A 4–5 row table ranking each failure mode by Probability, Severity, and whether it's a Kill Shot (independent path to zero) or Conditional.

**Load-Bearing Priors and What Would Break Them** *(numbered list)*
3–4 explicit prior beliefs that underpin the investment. For each, write the precise conditions under which it breaks. These should be falsifiable.

**Diligence Requirements Derived From This Pre-Mortem** *(bullet list)*
5–7 specific, actionable questions or evidence requests that flow directly from the failure mode analysis. These should be concrete enough to bring into a founder conversation or backchannel call.

**Closing note** *(1 line)*
Date the document and specify when it should be revisited (e.g., "Q3 2026 or first paid customer milestone, whichever comes first").

---

## Step 3: Save to Notion

Create a new page in the Notes database (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) with the following:

```
parent: { data_source_id: "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" }
pages: [{
  properties: {
    "Name": "Claude Pre-Mortem: [OPPORTUNITY NAME]",
    "Opportunity": "[Opportunity page URL]"
  },
  content: "[Full pre-mortem content in Notion-flavored markdown]"
}]
```

After creating the page, confirm the URL to Tom.

---

## Step 4: Set Claude Icon

Read the shared reference at `/Users/tomseo/.claude/skills/shared-references/claude-note-icon.md` and follow its instructions to set the custom Claude logo emoji as the page icon on the newly created note. This is required for all Claude-generated notes — do not skip.

---

## Timing

The pre-mortem is most valuable when written:
- **Before first substantive call** (cold, before the founder has built rapport) — highest signal, hardest to do
- **Before closing** — the second-best window; still pre-deployment, but confirmation bias has partially set in
- **After closing, before board seat** — useful for portfolio monitoring even if the pre-closing window was missed

When in doubt about timing, note it in the opening frame of the document.

---

## Quality Bar

A good pre-mortem has the following properties:
- Every failure mode cites specific evidence from the diligence record (a quote, a data point, a gap explicitly named in the notes)
- No failure mode is purely generic — each is calibrated to this company's specific product, market, and team
- The kill shot ranking reflects honest probability assessment, not wishful thinking
- The load-bearing priors are stated as falsifiable claims, not vague thesis summaries
- The diligence requirements are concrete enough to be actioned in the next founder or backchannel conversation

A bad pre-mortem reads like a generic VC risk section. If you find yourself writing "execution risk" or "market timing risk" without grounding those in specific evidence from this company's diligence record, stop and go back to the materials.


---

## Final Step: Classify the Note

After the Notion page has been created and the Opportunity relation (if any) has been set, run the `note-classifier` skill to assign the correct `Category` field.

Read the skill at `/Users/tomseo/.claude/skills/note-classifier/SKILL.md` and follow its classification logic against the note just created. Do not skip this step — every note created by this skill must have a Category set.

## Behavior Rule — PDF rendering uses the long-form spec

A pre-mortem is a long-form document. When rendering it (or any authored long-form artifact — memo, PRD, LP letter, investor report) to PDF, default to the canonical long-form PDF spec at `~/.claude/skills/shared-references/long-form-pdf-spec.md` using `~/.claude/skills/shared-references/pdf_builder_template.py` as the template.

That means: reportlab (not Chrome headless), Helvetica (not JetBrains Mono), black + white only (no orange/accent colors), underlined H1, italic 9pt #333 subtitle, page numbers bottom-center, no horizontal dividers.

**Why:** `design-language.md` (dark theme, JetBrains Mono) is for PNG exports and on-brand visual artifacts (skill maps, charts, LP letter exhibits). It is NOT the default for long-form authored documents. Tom had to redirect the first Founder Framework PRD render after it used Chrome + design-language styling; the canonical long-form spec is the right default.

**How to apply:** If the thing being rendered is prose-heavy with sections/tables/bullets (memo, PRD, pre-mortem, LP letter, investor report), use the long-form spec. If it's a single visual artifact (chart, diagram, skill map card, single-page exhibit embedded in Docs), use design-language.md. When in doubt on anything PRD-shaped, long-form wins.
