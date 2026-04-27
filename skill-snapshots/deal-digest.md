---
name: deal-digest
description: Log deal digest content to a monthly rolling page in the Notion Notes database. Every ingest — Chris Oh's periodic batches, one-off competitor intel, single traction snippets, anything else — prepends as a dated source-tagged block to the current month's page. Newer entries on top, no merging across ingests, no benchmark synthesis at log time. Trigger whenever Tom says "deal digest", "log this digest", "add digest to Notion", "save Chris Oh's notes", "log deals", "package up deals", "add to digest corpus", "log this traction datapoint", or pastes/forwards any block of company notes with ARR, valuation, fundraising, or competitive traction data. Also trigger when Tom shares a single competitor or company traction snippet (e.g. competitive intel forwarded by a portfolio CEO). Always trigger inline — no confirmation needed before acting.
---

# Deal Digest Logger

Log deal digest content into a single monthly rolling page in the Notes DB (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`). Every ingest — Chris Oh batch, ad-hoc competitor intel, single traction snippet — prepends as a dated, source-tagged block to that month's page. Newer entries on top. No merging across ingests, no benchmark synthesis.

## Workflow

1. **Determine the page** — derive the title from the current month: `Deal Digest – [Month YYYY]` (e.g. `Deal Digest – April 2026`).
2. **Locate or create** — search Notes DB for that page.
   - **If exists** — fetch its current content.
   - **If missing** — create it via `notion-create-pages`. Title above, Category=`Research`, Icon=🤝, body empty (the first ingest fills it).
3. **Build the new ingest block** (see Ingest Block Format below).
4. **Prepend** at the top of the page. Use `notion-update-page` with `update_content`; `old_str` = first line of existing content, `new_str` = `[new block] + [old first line]`. If the page is empty (just created), use `update_content` with empty old_str OR write content directly during create.
5. **Classify** — run `note-classifier` to confirm Category=Research.

## Ingest Block Format

Every ingest is a self-contained block with this shape:

```
YYYY-MM-DD – [Source description]

[verbatim content of the ingest — bullets, subsections, whatever the source provided]

---
```

- **Date stamp line** is **bold paragraph** — but with a trailing non-breaking space (`\u00a0`) to prevent Notion from auto-promoting standalone-bold lines to heading blocks. NEVER use `#`/`##`/`###` headings.
  - Format: `**YYYY-MM-DD – [Source]**` + trailing nbsp, with optional ` (context)` for retrieval framing.
  - Examples: `**2026-04-25 – Chris Oh WhatsApp digest**\u00a0`, `**2026-04-26 – Emily Man iMessage (competitive intel for Rengo)**\u00a0`, `**2026-04-20 – Pat Grady email**\u00a0`.
  - **Why the trailing nbsp:** A pure `**...**` standalone paragraph trips a Notion heuristic that promotes it to a heading block (rendered huge, like h2). Adding ANY non-bold character at the end keeps it a normal paragraph. A regular trailing space gets trimmed; a non-breaking space (U+00A0) survives serialization.
- **Body** preserves the source content as is. Don't merge with prior ingests, don't re-format across blocks. Apply per-bullet formatting rules below.
- **Trailing `---`** separates this block from the previous one. Visual only.

### Body formatting rules (within a block)

- Each company gets its own bullet — never comma-separate.
- Company name bolded up to colon: `- **Company Name:** commentary`. No colon → bold just the name: `- **Name** (backer or context)`. Inline bold inside a bullet is fine — Notion only auto-promotes standalone bold-only paragraphs.
- Don't alter underlying text — preserve original wording, figures, commentary verbatim. Formatting only.
- If the source had subgroup labels (Main / Seed / Other / Additional from Chris Oh), preserve them as bold paragraphs with trailing nbsp (`**Main**\u00a0`) — same heading-promotion workaround as the date stamp line. Never `##`/`###`.

## Page Metadata

| Field | Value |
|---|---|
| **Parent DB** | Notes DB — `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb` |
| **Title** | `Deal Digest – [Month YYYY]` (e.g. `Deal Digest – April 2026`) |
| **Category** | `Research` |
| **Icon** | 🤝 |
| **Opportunity relation** | NEVER set. Even if an ingest references a portfolio company (competitive intel framing), do NOT add an Opportunity relation. The corpus is queryable on its own; per-Opportunity linking would clutter the rolling page.|

## Source Handling

- **WhatsApp via Beeper** — search Beeper for the Chris Oh chat. Beeper truncates long messages; if the full text isn't returned, ask Tom to paste it directly.
- **Pasted text / forwarded email / iMessage screenshot** — use the text as is.

## Final Step

After updating the page, run `note-classifier` (`/Users/tomseo/.claude/skills/note-classifier/SKILL.md`) to confirm Category=Research.

## Example Page (after a few ingests)

Note: every `**bold paragraph**` below has an invisible trailing U+00A0 (non-breaking space) to prevent Notion from auto-promoting it to a heading.

```
**2026-04-26 – Emily Man iMessage (competitive intel for Rengo)**

- **Meridian:** Private equity CRM (meridian-ai.com); raising Series A; on a path from $1M to $5M ARR by year-end with blue-chip logos like Apax Partners, Point72, HIG Capital, and Zurich live on the platform with 100% retention

---

**2026-04-25 – Chris Oh WhatsApp digest**

**Main**

- **Eleven Labs:** Audio AI; $330M ARR (~3x YoY); profitable; raised at $11B
- **Fal:** Generative media inference; ...

**Seed**

- **Bob McGrew** (ex-OpenAI Chief Research Officer; 8VC)
- **Reactor** (Fal for world models; LSVP + Amplify)

**Other**

- **Surf** (Accel)
- **Applied Compute** (KP)

---

**2026-04-20 – Pat Grady email**

- **Foo:** ...

---
```

## Historical Note

Pre-2026-04-26 the skill produced two page shapes: dated batch pages (`Deal Digest – March 25, 2026`) with a benchmark summary, and ad-hoc rolling pages (`Deal Digest – Ad Hoc – April 2026`). Those existing pages stay as historical artifacts. The unified monthly format applies going forward only.
