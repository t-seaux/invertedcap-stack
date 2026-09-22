---
name: add-conversation-to-notion
description: Save the current Claude conversation as a new entry in the Notion Notes database. Trigger whenever the user says "add this thread to Notion", "log this conversation", "save this chat to Notion", "add this to Notes", "save this thread", "add to Notion", "log this", or any variant asking to record/archive the current Claude conversation in Notion. CONTEXT-DEPENDENT: a bare "log" / "log this" / "log to notes" during an ongoing Claude session where we've been working — with NO text-thread screenshot or video URL attached — routes HERE (log the session). If instead Tom has shared/attached a text-thread screenshot or pasted a third-party chat thread, "log" routes to log-thread-to-notes (verbatim transcript); a video/interview/YouTube URL → log-transcript-to-notion; a letter/report/memo link or PDF → log-document-to-notes (investor-firm letter → log-investor-letter-to-notion). Not this whenever any such object is present. Always trigger inline — no confirmation needed before acting.
---

# Add Conversation to Notion (Notes Database)

Create a new page in the ✏️ Notes database containing the full conversation transcript, with the Claude conversation URL linked at the top.

**Notes database data_source_id:** `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`

## Step 1: Determine the Conversation URL (optional)

If the user has explicitly provided a conversation URL (format: `https://claude.ai/chat/<conversation-id>`), include it at the top of the page content as `🔗 [Claude conversation](<url>)`. If no URL has been provided, skip this entirely — do not ask for it and do not create a placeholder link.

## Step 2: Determine the Note Title

Default title format:
```
Claude Thread: <brief topic>
```

Infer `<brief topic>` from the dominant subject of the conversation (e.g., "Tuor Pitch Deck", "Series A Benchmarks", "Rengo Investment Memo"). Keep it concise — 3–6 words.

If the user specifies a title, use it verbatim. Do NOT ask the user to confirm or approve the inferred title — proceed directly.

## Step 3: Identify the Opportunity (optional, best-effort)

The Notes database has an `Opportunity` relation field that links to the Opportunities database. An auto-tagger Notion agent will attempt to link the note automatically, but set it proactively if the topic maps clearly to a known opportunity.

Look at the conversation subject. If it clearly relates to a specific company or deal (e.g., the whole thread is about Tuor), search for the matching Opportunity:

```
Tool: notion-search
Query: <company name>
```

If a confident single match is found in the Opportunities database (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`), include its URL in the `Opportunity` property at creation time. If ambiguous or no match, leave it blank — the auto-tagger will handle it.

## Step 4: Build the Page Content

Structure the page body as follows:

If a conversation URL was provided:
```
🔗 [Claude conversation](<conversation_url>)
---

**Frameworks**

<frameworks content — see below>

---

**Letter**

<full conversation transcript — see below>
```

If no URL was provided, omit the link line entirely:
```
**Frameworks**

<frameworks content — see below>

---

**Letter**

<full conversation transcript — see below>
```

---

### Frameworks Section

Use `**Frameworks**` as a bolded text label (not a Markdown header `##`). Scan the full conversation and identify 3–6 key mental models, investment theses, analytical frameworks, or recurring conceptual threads that emerge from the exchange. For each framework:

- State it as a bolded short title (e.g. `**Data Asset as Moat**`)
- Follow with 2–4 sentences explaining the framework as it was developed or applied in this specific conversation
- Ground it with a concrete reference to the conversation content — paraphrase the relevant turn or reasoning rather than writing frameworks in the abstract

**Traceability check (MANDATORY after generating frameworks)**: for each framework, verify it traces to actual conversation turns — the reasoning it describes must have been argued or developed in the thread, not inferred from the topic. If a framework can't be anchored to specific turns, re-generate it once with the constraint "only extract frameworks actually developed in this conversation." Drop any framework that still can't be traced.

The goal is to make the Frameworks section a standalone, scannable summary of the intellectual content of the thread — the kind of section Tom can re-read weeks later without needing to re-read the full transcript to reconstruct what was actually being argued.

---

### Letter Section

Use `**Letter**` as a bolded text label (not a Markdown header `##`). Reproduce the conversation in full, alternating between labeled turns. Tom's questions/messages must be rendered in blue using Notion's inline color syntax to make them visually distinct from Claude's responses. Use standard formatting (bullets, bold, line breaks) for Claude's responses.

```
**Tom:** {color="blue"}<message>

**Claude:** <response with full formatting — bullets, bold, line breaks — preserved>
```

**Color EVERY block of Tom's message, not just the first.** The `{color="blue"}` annotation only spans a single block (one paragraph / list item / quote); it does NOT carry across a blank-line break or into list items. So when Tom's turn has multiple paragraphs, a numbered/bulleted list, or any multi-block structure, prefix `{color="blue"}` to the content of EACH block — every paragraph and every list item — or the tail blocks render white. Example of a multi-block Tom turn done right:

```
**Tom:** {color="blue"}Here are my concerns:

1. {color="blue"}First concern text.

2. {color="blue"}Second concern text.

{color="blue"}And a closing paragraph after the list.
```

Include all turns from the beginning of the thread up to — but NOT including — the turn in which Tom issues the log instruction. Do not summarize or truncate the turns you DO include — the goal is a complete, searchable archive of the *thinking*.

**Stop at the log command (capture the thinking, not the plumbing).** Tom's log request — "log this", "log this convo", "log to notes", or any turn whose actionable ask is to log / draft / execute — is an operational instruction, not part of the substance he wants preserved. Find that turn and EXCLUDE it and every turn below it: his log/execute message, Claude's execution report, and any follow-on operational back-and-forth about the drafting or logging itself. The archive ends at the last substantive turn *before* the log request. (This overrides "include all turns" — the rule is complete capture of everything above the log command, hard stop at it.)

If the conversation is very long and truncation is unavoidable due to context limits, note at the bottom: `[Note: transcript truncated — view full conversation at link above]`

## Step 5: Create the Notion Page

Use `notion-create-pages` with:

```json
{
  "parent": { "data_source_id": "e8afa155-b41a-4aa2-8e9d-3d4365a11dfb" },
  "pages": [{
    "properties": {
      "Name": "<title from Step 2>",
      "Opportunity": "<opportunity page URL if found, else omit>",
      "⭐️": "__NO__"
    },
    "content": "🔗 [Claude conversation](<url>)\n---\n**Frameworks**\n\n<frameworks>\n\n---\n\n**Letter**\n\n<transcript>"
  }]
}
```

## Step 6: Set Claude Icon

Read the shared reference at `/Users/tomseo/.claude/skills/shared-references/claude-note-icon.md` and follow its instructions to set the custom Claude logo emoji as the page icon on the newly created note. This is required for all Claude-generated notes — do not skip.

## Step 7: Confirm to User

After successful creation, respond with one line:

> ✓ Saved to Notes: **[Note Title]** → [Notion page URL]

## Error Handling

- **Notion MCP unavailable:** Inform the user and suggest they log the conversation manually.
- **Title unclear:** Default to "Claude Thread: General" rather than asking.


---

## Final Step: Classify the Note

After the Notion page has been created and the Opportunity relation (if any) has been set, run the `note-classifier` skill to assign the correct `Category` field.

Read the skill at `/Users/tomseo/.claude/skills/note-classifier/SKILL.md` and follow its classification logic against the note just created. Do not skip this step — every note created by this skill must have a Category set.
