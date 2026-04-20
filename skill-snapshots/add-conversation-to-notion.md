---
name: add-conversation-to-notion
description: Save the current Claude conversation as a new entry in the Notion Notes database. Trigger whenever the user says "add this thread to Notion", "log this conversation", "save this chat to Notion", "add this to Notes", "save this thread", "add to Notion", "log this", or any variant asking to record/archive the current Claude conversation in Notion. Always trigger inline — no confirmation needed before acting.
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

The goal is to make the Frameworks section a standalone, scannable summary of the intellectual content of the thread — the kind of section Tom can re-read weeks later without needing to re-read the full transcript to reconstruct what was actually being argued.

---

### Letter Section

Use `**Letter**` as a bolded text label (not a Markdown header `##`). Reproduce the conversation in full, alternating between labeled turns. Tom's questions/messages must be rendered in blue using Notion's inline color syntax to make them visually distinct from Claude's responses. Use standard formatting (bullets, bold, line breaks) for Claude's responses.

```
**Tom:** {color="blue"}<message>

**Claude:** <response with full formatting — bullets, bold, line breaks — preserved>
```

Include all turns from the beginning of the thread. Do not summarize or truncate — the goal is a complete, searchable archive.

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
