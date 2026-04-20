---
name: feedback-outreach-drafter
description: >
  Draft feedback outreach notes (also known as backchannel notes) — diligence feedback request emails to expert contacts in Tom's network. Two modes: (1) Manual — Tom names specific people and a company; (2) Scheduled scan — checks the Pending Feedback relation for new entries and drafts outreach notes. Trigger on "draft feedback outreach notes", "draft backchannel notes", "get [NAME]'s feedback on [OPPORTUNITY]", "send diligence questions to [name(s)] about [company]", "reach out to my network for feedback on [company]", "write backchannel emails for [company]", "feedback outreach note for [company]", or any variant asking to email contacts for diligence input on a deal. Always trigger inline — no confirmation needed.
---

# Feedback Outreach Drafter (Backchannel Notes)

Draft feedback outreach notes (also known as backchannel notes) — diligence feedback request emails to expert contacts in Tom's network, using materials from the Notion opportunity to populate the company blurb and tailor the questions. Then log each recipient on the opportunity's `📣 Pending Feedback` relation field.

## Email Format (from screenshot reference)

### Subject line template
```
Thoughts on [Company Name] ([one-line company descriptor])?
```
Example: `Thoughts on Clusia (AI for financial compliance and payment operations)?`

### Body structure
```
Hey [First Name],

Hope you're well (and hoping this isn't too much of an out-of-the-blue note)! I'm digging into a [Stage] opportunity and figured you'd have a gut take. Company blurb below.

No worries if this isn't interesting or a priority atm – that in and of itself is a helpful market signal – but would you happen to have any quick reactions to the questions below? Happy to jump on a call (or text thread) to discuss live if easier:

* [Question 1]

* [Question 2]

* [Question 3]

Best,
Tom

--

About [Company Name] ([website or "website N/A"])

**[Company] is building [core product description — one bold sentence.]** [Supporting context: who the customers are, what problem they face, how data/workflow is fragmented.]

**[Company] orchestrates this work.** [How the product works: what it ingests, what it automates, what the customer doesn't need to do.]

[Team line: "The company is co-founded by [Founder 1], [brief background], and [Founder 2], [brief background]."]
```

### Tone and style notes
- Warm but not sycophantic. The opener is casual ("Hope you're well") but businesslike.
- "That in and of itself is a helpful market signal" is a fixture — keep it.
- Questions should be sharp and specific, not open-ended. Each question should own a distinct dimension of the thesis (e.g., problem severity, AI substitution viability, GTM/category dynamics). Avoid repeating the same frame across questions.
- The company blurb lives below the signature, separated by `--`. It reads as an appendix, not part of the letter itself.
- The blurb should be 3–5 sentences max. Bold the core value proposition sentence and the product mechanism sentence. End with a one-line team note.

---

## Modes

### Manual Mode
Tom provides names and a company explicitly. Skip to Step 1 with the provided names.

### Scheduled Scan Mode
Triggered by the Diligence Agent on a recurring schedule. No names are provided — the skill discovers them from Notion.

**Step 0: Scan for new Feedback entries**

Query the Opportunities DB for all opportunities whose `📣 Pending Feedback` relation field was updated in the past 30 hours. For each opportunity:

> **Why 30 hours:** The agent runs on a ~10-hour cadence (8am and 6pm ET). A 30-hour lookback creates a 20-hour overlap between consecutive runs, so any contact added near a run boundary — or during a run's own execution window — is always caught by the next run. The three-state dedup check below prevents double-drafting regardless of overlap.

1. Fetch the opportunity page and read the current `📣 Pending Feedback` array.
2. For each person in the array, run a **three-state check**:

   **State 1 — Sent mail exists:**
   Search `subject:"Thoughts on [Company]" in:sent to:[person email]`
   → If found, skip. Already reached out.

   **State 2 — Draft exists but not yet sent:**
   Call `gmail_list_drafts` and scan for any draft whose subject matches `Thoughts on [Company]` addressed to this person.
   → If found, skip. Draft is pending send — do not recreate.

   **State 3 — Neither sent nor drafted:**
   → Add to the drafting queue for this opportunity.

3. If the queue is empty after checking all opportunities, return a summary noting nothing to draft and exit.

If names are in the queue, proceed to Step 1 for each person, using the opportunity already resolved from the scan.

## Workflow

### Step 1: Resolve recipient email(s) and People DB entry

For each name provided by Tom:

1. Search the **People DB** in Notion (`collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`) using `notion-search` with the person's name. If found, fetch the full People page and retrieve the `Email` field.
2. If the email field is blank, use `contactout_enrich_person` with full name as a fallback.
3. **If the person is NOT found in the People DB at all**, create a new People DB entry before proceeding. Follow the field mapping from the `add-to-contacts` skill at `/Users/tomseo/.claude/skills/add-to-contacts/SKILL.md`:
   - Use `contactout_enrich_person` (with name + company/context if available) to get email and profile data
   - Use `contactout_enrich_linkedin_profile` if a LinkedIn URL is available to get full profile data
   - Populate: Name, Email, LI, Company, Role, Category, City, State — per the field rules in add-to-contacts
   - Create via `notion-create-pages` with `data_source_id: 1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`
   - Use the newly created page's ID for the `📣 Pending Feedback` relation in Step 7
4. If ContactOut returns nothing and the person can't be found or created, flag to Tom and skip that recipient.

Note: LinkedIn URLs are NOT needed for recipients — only for founders in the company blurb (see Step 3).

### Step 2: Pull diligence materials from the Notion opportunity

Search the **Opportunities DB** (`fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`) for the company name using `notion-search`. Retrieve the matching opportunity page.

From the opportunity page, gather:
- **Description** (one-liner property) — use as the subject line descriptor
- **Page body** — read the Summary section for product/market context
- **Team** section — extract founder names and backgrounds for the blurb
- **Website** property — include in the blurb header
- Any linked **call notes, transcripts, or memos** — read these to understand the thesis more deeply and inform question quality

If a memo or transcript exists as a linked Notion page or Notes DB entry, fetch and read it before drafting. The goal is to write questions that reflect actual diligence depth, not generic ones.

### Step 3: Draft the company blurb

Using the materials from Step 2, write a 3–5 sentence company blurb following the format above:
- Sentence 1 (bold): "[Company] is building [core product]." — punchy, jargon-minimal.
- Sentence 2: Customer context — who uses this, what's broken in their world today.
- Sentence 3 (bold): "[Company] orchestrates/solves this." — the mechanism.
- Sentence 4: Optional supporting detail (no SQL, no engineering lift, etc.).
- Sentence 5: Team line.

### Step 4: Draft the diligence questions

Write 2–3 sharp questions tailored to this specific opportunity. Each question should:
- Own a distinct thesis dimension (problem severity, AI substitution viability, competitive/category dynamics, GTM motion, customer trust, etc.)
- Be specific enough that an expert with relevant experience can give a substantive answer
- Not repeat the same frame as another question (e.g., don't ask about "throwing bodies at the problem" twice)
- End with a clear question mark — avoid run-on compound questions

Draw on the thesis as understood from memos, transcripts, and the opportunity page. The questions should signal that Tom has done work.

### Step 5: No personalization line

Do NOT add a line referencing where the recipient works or their background (e.g., "given your time at X" or "given your work at Y"). This reads as filler. The opener should simply be:

> "I'm digging into a [stage] opportunity and figured you'd have a gut take."

Where [stage] maps from the opportunity's Stage field as follows:
- Pre-Seed 💡 → "pre-seed"
- Seed 🌾 → "seed"
- Seed+ 🛣️ → "seed"
- Series A 🏎️ → "Series A"
- Series B 📈 → "Series B"
- Growth 🚀 → "growth"
- Any other / unknown → "early-stage"

The fact that Tom reached out to this specific person implies the relevance — it does not need to be stated.

### Step 6: Create Gmail draft(s)

Use the Gmail MCP (`gmail_create_draft`) to create one draft per recipient. Fields:
- **To**: recipient email (from Step 1)
- **From**: `tom@invertedcap.com`
- **Subject**: `Thoughts on [Company] ([descriptor])?`
- **Body**: full email as HTML (see formatting rules below)
- **contentType**: always `text/html`

Create each draft independently.

#### HTML Formatting Rules

Always send as `text/html`. Use inline styles for compatibility with Gmail rendering:

```html
<!DOCTYPE html>
<html>
<body style="font-family: Helvetica, Arial, sans-serif; font-size: 13px; color: #000; line-height: 1.6;">

<p>Hey [First Name],</p>

<p>[Opener paragraph]</p>

<p>[No worries paragraph]</p>

<p>* [Question 1]</p>
<p>* [Question 2]</p>
<p>* [Question 3]</p>

<p>Best,<br>Tom</p>

<p>--</p>

<p><em>About [Company] ([website or "website N/A"])</em></p>

<p><strong>[Company] is building [core product].</strong> [Supporting context sentence.]</p>

<p><strong>[Company] orchestrates this work.</strong> [Mechanism sentence.]</p>

<p>The company is co-founded by <a href="[LinkedIn URL]" style="color: #1155CC;">[Founder 1 Name]</a>, [background], and <a href="[LinkedIn URL]" style="color: #1155CC;">[Founder 2 Name]</a>, [background].</p>

</body>
</html>
```

Key formatting rules:
- The two core blurb sentences are wrapped in `<strong>` tags
- Founder names in the team line are hyperlinked to their LinkedIn URLs using `<a href="..." style="color: #1155CC;">`
- The "About [Company]" header is italicized with `<em>`
- Use `<p>` tags for all paragraphs; `<br>` for line breaks within a paragraph (e.g., "Best,\nTom")
- LinkedIn URLs: retrieve from the opportunity's Founder(s) relation pages in Notion (People DB). If a LinkedIn URL is not found in the DB, use `contactout_enrich_person` with the founder's full name and company to attempt a lookup. If ContactOut also returns nothing, leave the founder name as plain text (no hyperlink) and note to Tom that a LinkedIn URL could not be verified. Never infer or guess a LinkedIn slug.

### Step 7: Update `📣 Pending Feedback` relation on the opportunity

After creating all drafts, update the opportunity page's **`📣 Pending Feedback`** relation field.

The `📣 Pending Feedback` relation field must be updated by passing a **JSON array string** of ALL page URLs — existing entries plus new ones — in a single `notion-update-page` call. Writing a single URL or making sequential calls will overwrite all prior entries.

**Procedure:**
1. Fetch the current Opportunity page and read the existing `📣 Pending Feedback` array (may be empty or already contain entries).
2. Collect the People DB page URLs for all recipients in this batch (from Step 1).
3. Merge existing entries + new entries into a single deduplicated list.
4. Write the full list back in one call using a JSON array string format:



This is the same pattern used by the intro-agent skill for `👓 Intros (Qualified)`.

### Step 8: Create Per-Person Feedback Note in Notion (on response only)

Do NOT create a Notion note when drafts are sent. A note is created only when a specific person replies with feedback.

When a reply is received (either Tom flags it or a Gmail scan surfaces it), create a new Notes DB page for that person's feedback using `notion-create-pages` with `data_source_id: e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`.

**Page title:** `[Company] — [First Name] [Last Name] ([Company Name]) Feedback`
Example: `Clusia — Jeff Green (Verde Tech Ventures) Feedback`

**Page content structure:**
```
<mention-page url="[Notion People DB URL]"/> ([LinkedIn](LI URL)) — [Role], [Company]. [1-2 sentence background].

---

## Response — [Date]

[Verbatim or lightly cleaned response — no editorializing]

---

## Outreach Note — [Date sent]

[The plain-text body of the outreach email sent to this specific person —
opener, questions, and company blurb]
```

Format rules:
- Use `<mention-page url="[Notion People DB URL]"/>` to create a direct mention/link to the person's People DB page — this renders as their name in Notion
- Follow immediately with `([LinkedIn](LI URL))` as a hyperlinked URL placeholder
- Background is 1-2 sentences reflecting the person's current primary role and 1-2 prior relevant roles. Always verify against their LinkedIn profile (via ContactOut enrichment if needed) — do not rely solely on the People DB Company/Role fields as these may be stale. Use the add-to-contacts primary role rules: full-time day job takes precedence over side activities
- No location line
- Response section always comes before the Outreach Note
- Both sections are dated

After creating the page, link it to the opportunity via the `✍️ Notes` relation using the JSON array string pattern — fetch the current `✍️ Notes` array first, append the new page URL, and write the full array back in a single call.

If the same person sends a follow-up reply later, append it to their existing note using `notion-update-page` with `command: update_content` — do not create a second page.

Also update the respondent's email in the People DB if a different address was used in the reply than what was on file.

### Step 9: Confirm to Tom

Summarize what was done:
- Number of Gmail drafts created, with recipient name(s)
- Whether the blurb and questions were drawn from a memo/transcript or from the opportunity page alone (so Tom knows the depth of sourcing)
- Whether any recipients were skipped due to missing email
- Whether each recipient was newly added to `📣 Pending Feedback` or was already present

---

## Edge Cases

- **Multiple recipients, same company**: Run Steps 1–5 per recipient, Steps 6–7 together. The blurb and questions can be identical; personalization in the opener varies.
- **No memo or transcript available**: Draft questions from the opportunity page Summary and Description alone. Note this in the Step 8 summary so Tom knows the questions are less thesis-informed.
- **Company not found in Notion**: Alert Tom before proceeding. Do not draft without a source of truth on the company.
- **Recipient not in People DB and ContactOut returns nothing**: Flag to Tom and skip. Do not guess emails or create a partial entry with no contact info.
