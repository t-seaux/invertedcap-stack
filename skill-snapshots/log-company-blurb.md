---
name: log-company-blurb
description: Log a new company blurb (Company Overview) to an Opportunity page in the Notion CRM using the canonical blurb-versioning format — the latest blurb lives in a blue 📚 callout headed by its date, and every prior version is collapsed into a "Company Overview (History)" toggle beneath it. Trigger when Tom says "log company blurb", "log company overview", "log the overview for [X]", "company overview for [X]", "log this blurb for [X]", "log the blurb", "company blurb for [X]", "add this blurb to [X]", "update the blurb for [X]", "update the overview for [X]", or pastes/settles a company blurb in conversation and asks to log it to Notion. Manual-only — no scheduled or webhook entry point. Works on any Opportunity row (portfolio or pipeline).
---

# Log Company Blurb

Log a company blurb to its Opportunity page using the canonical blurb-versioning format. The page shows exactly one current blurb (blue callout) and hides all history behind one toggle.

## Canonical format (target state of every page)

```
<callout icon="📚" color="blue_bg">
	*Company Overview (<mention-date start="YYYY-MM-DD"/>)*
	[current blurb paragraph 1]
	[current blurb paragraph 2 …]
</callout>
<details>
<summary>*Company Overview (History)*</summary>
	<mention-date start="YYYY-MM-DD"/> [first paragraph of the most recent prior version]
	[its subsequent paragraphs, if any]
	<mention-date start="YYYY-MM-DD"/> [next older version …]
	(Old) [an undated version's first paragraph]
</details>
```

Format rules:
- **The callout is ALWAYS the very first block of the page body** (History toggle immediately after it). If an existing blurb lives mid-body, move the structure to the top — never transform it in place and leave it buried under other content.
- Callout: icon `📚`, color `blue_bg`. Header is italic `*Company Overview (<mention-date …/>)*` — a real Notion date mention, not plain text. If the blurb's date is genuinely unknown, use `*Company Overview (Latest)*`.
- History toggle: entries newest-first. No per-entry headers and NO `<empty-block/>` spacers between entries — each version's FIRST paragraph is prefixed inline with its `<mention-date …/>` (or `(Old) ` when undated); subsequent paragraphs of a multi-paragraph version follow as plain tab-indented paragraphs.
- Children of the callout and toggle are tab-indented in Notion-flavored markdown.
- Escape `$` as `\$` (and other spec-required chars) in markdown ops. Preserve pasted rich text verbatim — links `[text](url)`, bold, en dashes. Never reword the blurb.
- Sections elsewhere on the page (TS House Take, meeting notes, Materials, etc.) are never touched.

## Workflow

1. **Resolve the blurb text.** Usually the final version settled in the current conversation, or pasted with the trigger. If multiple candidate texts exist in context, use the most recent "final" one; ask only if genuinely ambiguous.

2. **Resolve the Opportunity page.** `notion-search` with `data_source_url: "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"` on the company name. If semantic search comes up empty, fall back to SQL before concluding the row doesn't exist: `notion-query-data-sources` with `SELECT "Name","url" FROM "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6" WHERE "Name" LIKE ?` (`%name%`). Multi-round companies: the blurb ALWAYS lives on the company's original/primary row — never on a `(… FO)` follow-on row (FO rows don't carry blurbs; per Tom, 2026-08-03). If Tom invokes this on an FO row's name, log to the primary row.

3. **Fetch the page** (`notion-fetch`) and classify the body:
   - **(a) Already canonical** — has the 📚 callout (and possibly the History toggle).
   - **(b) Legacy** — one or more flat blurb sections headed by italic lines like `*Company Overview (…)*` / `*Company Blurb (May 2025)*` / `*Company Overview (OLD)*`.
   - **(c) No blurb** — nothing blurb-like in the body.

4. **Apply edits with `notion-update-page` `update_content` (targeted `old_str`/`new_str` ops) ONLY. NEVER `replace_content`** — pages can hold `<meeting-notes>`/transcript blocks that do not round-trip through a full-page replace. `old_str` must reproduce the fetched markdown exactly, including leading tabs, `\$` escapes, apostrophe styles, non-breaking spaces (` `), and existing `<mention-date>` tags. On "No matches found", suspect a stale `notion-fetch` cache before suspecting your string: pull ground truth via `ntn api /v1/pages/{id_no_dashes}/markdown` (the MCP fetch can serve snapshots months old and normalizes ` ` to plain spaces) and rebuild `old_str` from that. Never escalate to a full replace. Also beware Notion's auto-linkifier: rewriting text containing bare `@domain.tld` / `domain.tld` tokens via markdown re-linkifies them — if that corrupts a paragraph, patch that block's `rich_text` directly via `ntn api -X PATCH /v1/blocks/{block_id}`.

   - **Case (a):** Two logical moves in one call: (1) demote the current callout body — prepend it at the TOP of the History toggle's children, converting the callout header's date mention into the inline prefix of its first paragraph; (2) replace the callout's contents with the new blurb, headed by today's date mention. If no History toggle exists yet (first update after migration), create it immediately after the callout with the demoted version as its only entry.
   - **Case (b):** Migrate to canonical in the same pass: newest existing section (by date; undated = oldest) → the History toggle's top entry along with all other priors (dates → inline mention prefixes, undated → `(Old) `), and the NEW blurb → the callout with today's date. Original section headers are dropped; prose is preserved verbatim.
   - **Case (c):** Create the callout at the top of the page body with the new blurb and today's date. No toggle until a second version exists.

5. **Verify.** Re-fetch the page: callout shows the new blurb with today's date mention; the previous version's full text is present at the top of the toggle; nothing else changed. Report the page link and what was demoted.

## Notes

- Date for the new blurb = today, unless Tom supplies an explicit date ("log this as the July blurb" → use that date).
- This skill only writes the page body. It does not touch the `Description` property — if the new blurb's one-liner materially diverges from `Description`, mention that in the reply so Tom can decide.
- The blurb logged here is also what intro drafters paste under `--` / *About [Company]* — after logging, the callout is the canonical source for the company's current blurb.
