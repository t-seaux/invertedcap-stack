---
name: feedback-outreach-drafter
description: >
  Draft feedback outreach notes (also known as backchannel notes) — diligence feedback request emails to expert contacts in Tom's network. Two modes: (1) Manual — Tom names specific people and a company; (2) Scheduled scan — checks the Pending Feedback relation for new entries and drafts outreach notes. Trigger on "draft feedback outreach notes", "draft backchannel notes", "get [NAME]'s feedback on [OPPORTUNITY]", "send diligence questions to [name(s)] about [company]", "reach out to my network for feedback on [company]", "write backchannel emails for [company]", "feedback outreach note for [company]", or any variant asking to email contacts for diligence input on a deal. Always trigger inline — no confirmation needed.
---

# Feedback Outreach Drafter (Backchannel Notes)

Draft feedback outreach notes (also known as backchannel notes) — diligence feedback request emails to expert contacts in Tom's network, using materials from the Notion opportunity to populate the company blurb and tailor the questions. Then log each recipient on the opportunity's `📣 Pending Feedback` relation field.

## Email Format

**Canonical voice + format live in the corpus: `~/.claude/skills/writing-style/feedback-outreach/STYLE.md`.**
Read `STYLE.md` + `EDIT_PATTERNS.md` + `VOICE_EXAMPLES.md` in that folder before finalizing every
draft. The stylebook owns the subject-line form, the full body scaffold (opener → market-signal
fixture → sharp questions → `Best, Tom` → `--` → italic `About [Company]` blurb), the question-craft
rules, the blurb rules, the relationship-based opener variants, and the voice register. Do not
re-specify the format here — the stylebook is the single source. The Step 6 HTML rules below are
drafter-only rendering mechanics.

---

## Modes

### Manual Mode
Tom provides names and a company explicitly. Skip to Step 1 with the provided names.

### Scheduled Scan Mode
Triggered by the Diligence Agent on a recurring schedule. No names are provided — the skill discovers them from Notion.

**Step 0: Scan for new Feedback entries**

Query the Opportunities DB for all opportunities whose `📣 Pending Feedback` relation field was updated in the past 30 hours. For each opportunity:

> **Why 30 hours:** The agent runs on a ~10-hour cadence (8am and 6pm ET). A 30-hour lookback creates a 20-hour overlap between consecutive runs, so any contact added near a run boundary — or during a run's own execution window — is always caught by the next run. The draft-once dedup check below prevents double-drafting regardless of overlap.

1. Fetch the opportunity page and read the current `📣 Pending Feedback` array.
2. For each person in the array, run a **draft-once dedup check** across all of Gmail (including Trash and Spam):

   **Any trace exists (sent, draft, or previously-deleted draft):**
   Search `subject:"Thoughts on [Company]" to:[person email] in:anywhere`
   → If ANY match returns (in sent, drafts, or trash), skip. The outreach has already been handled — either sent, awaiting Tom's send, or Tom deleted the draft as a terminal signal (meaning he reached out through another channel or decided not to pursue this contact).

   **No trace anywhere:**
   → Add to the drafting queue for this opportunity.

   **Why draft-once semantics:** Presence in `📣 Pending Feedback` is sticky — the relation persists after the initial draft. Without checking Trash, a deleted draft would keep getting re-created on every scan. Treating draft deletion as terminal collapses the loop: Tom drafts, decides (send / manual reach-out / skip), the record ends there.

3. If the queue is empty after checking all opportunities, return a summary noting nothing to draft and exit.

If names are in the queue, proceed to Step 1 for each person, using the opportunity already resolved from the scan.

## Workflow

### Step 1: Resolve recipient email(s) and People DB entry

For each name provided by Tom:

1. Search the **People DB** in Notion (`collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`) using `notion-search` with the person's name. If found, fetch the full People page and retrieve the `Email` field.
2. If the email field is blank, use `contactout_enrich_person` with full name as a fallback.
3. **If the person is NOT found in the People DB at all**, create a new People DB entry before proceeding. **✅ Auto-creation is authorized here — do not stop and ask.** Tom explicitly named this person as someone to get feedback from, so the relationship is already established and there is no judgement call left for him (confirmed 2026-07-31). The Slack heads-up below replaces the approval gate rather than preceding it. Follow the field mapping from the `add-to-contacts` skill at `/Users/tomseo/.claude/skills/add-to-contacts/SKILL.md`:
   - **Email IS a strong key** (as of the 2026-07-31 MCP fix). `contactout_email_to_linkedin(email)` returns the LinkedIn URL directly, and `contactout_enrich_person(email=…)` now returns Name / Headline / Company / LinkedIn / Location correctly. Both were broken before that date — `email_to_linkedin` 404'd on every call, and `enrich_person` rendered `Company: [object Object]`. If you see either symptom again, the MCP has regressed.
   - **Best chain:** `contactout_email_to_linkedin(email)` → `contactout_enrich_linkedin_profile(url)` for the full profile (headline, location, seniority, complete experience history). `enrich_person` is fine for a quick identity check.
   - For *current* employer trust the profile's `is_current` experience entry over any cached Notion value — Notion rows go stale (Anthony Chen's row still showed a 2017 Flexport title in July 2026).
   - Use `contactout_enrich_linkedin_profile` if a LinkedIn URL is available to get full profile data
   - Populate: Name, Email, LI, Company, Role, Category, City, State — per the field rules in add-to-contacts. **Never skip the lookup for speed** — that hard rule still binds; what changed is only whether Tom is asked first.
   - Create via `notion-create-pages` with `data_source_id: 1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`
   - Use the newly created page's ID for the `📣 Pending Feedback` relation in Step 7
   - **Post a Slack heads-up via `send-alert` to `#claude-alerts`** for every row created:

     > 👤 Created People DB entry: **[Name]** ([email])
     > Reason: feedback outreach on **[Company]** — not previously in People DB
     > Enriched: [LinkedIn URL | "⚠️ no ContactOut match — needs manual enrichment"]
     > [Company] · [Role] · [City]
     > → [Notion People page URL] · [Opportunity URL]
     > ↩️ Reply with their LinkedIn URL and I'll enrich the row.

     The header MUST start with `👤 Created People DB entry` — `claude-alerts-listener` special
     branch 8 keys on that string to route Tom's LinkedIn-URL reply back into enrichment. The
     `↩️` line is required whenever enrichment came back empty, and harmless otherwise (a reply
     also lets Tom correct a wrong ContactOut match). Always include the People page URL — the
     listener needs it to find the row.

     Batch multiple creations from one run into a single message rather than one alert per person.
     When batching, keep one `→ [People page URL]` line per person so the listener can
     disambiguate which row a reply refers to.
4. If every ContactOut lookup comes back empty, **still create the row** with the name and email you have, and say so explicitly in the Slack heads-up so Tom knows it needs manual enrichment. Only skip the recipient entirely if you have neither a usable name nor an email.

Note: LinkedIn URLs are not needed at all in this skill — not for recipients, and (as of 2026-07-31) not for founders either, since the blurb no longer carries a team line (see Step 3).

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

**No founder / team line.** Do NOT close the blurb with who is behind the business — no names, no
pedigree, no LinkedIn links. The recipient is being asked to judge the *business*, and founder
background invites them to evaluate the bet instead of the merits (and can bias the answer). End the
blurb at the product. Confirmed preference, 2026-07-31 — supersedes the team line shown in the
STYLE.md worked example.

### Step 4: Draft the diligence questions

Write 2–3 sharp questions tailored to this specific opportunity. Each question should:
- Own a distinct thesis dimension (problem severity, AI substitution viability, competitive/category dynamics, GTM motion, customer trust, etc.)
- Be specific enough that an expert with relevant experience can give a substantive answer
- Not repeat the same frame as another question (e.g., don't ask about "throwing bodies at the problem" twice)
- End with a clear question mark — avoid run-on compound questions

Draw on the thesis as understood from memos, transcripts, and the opportunity page. The questions should signal that Tom has done work.

### Step 5: No personalization line, no stage

Do NOT add a line referencing where the recipient works or their background (e.g., "given your time at X" or "given your work at Y"). This reads as filler. The opener should simply be:

> "I'm digging into an investment opportunity and figured you'd have a gut take."

**Never name the round or stage** — not "pre-seed", not "seed", not "Series A". The ask is about the
merits of the **commercial** opportunity, not the investment one; financing details are irrelevant to
that question and the Stage field must not be read into the opener. `an investment opportunity` is the
canonical phrasing (`a company` is an acceptable alt). Confirmed preference, 2026-07-31.

The fact that Tom reached out to this specific person implies the relevance — it does not need to be stated.

### Step 6: Create Gmail draft(s)

Create one draft per recipient **AND write the draft-feedback snapshot atomically** via
`~/.claude/scripts/gmail-create-draft.py` (same helper as `founder-outreach` Step 7 — draft +
snapshot in one shot, so Tom's edits feed `writing-style/feedback-outreach/EDIT_PATTERNS.md` via
diff mode). Per recipient, write two scratch files — the HTML body (formatting rules below) and a
plain-text snapshot body (strip tags; end at `Tom` + the `--`/About blurb; no signature block) —
then:

```
~/.claude/scripts/gmail-create-draft.py \
  --to <recipient email from Step 1> \
  --subject "Thoughts on [Company] ([descriptor])?" \
  --html-body-file /tmp/<scratch>.html \
  --snapshot-text-file /tmp/<scratch>.txt \
  --skill feedback-outreach-drafter
```

Exit code 0 = both writes succeeded; non-zero = that recipient's draft failed (don't fall back to a
snapshot-less MCP draft). Create each draft independently.

**Exit code 4 = blocked by the writing-style gate** (`~/.claude/scripts/style_gate.py`, added
2026-07-31). This is a HARD, deterministic gate that runs before the Gmail POST, so nothing is
created — there is no draft to clean up. It encodes this stylebook's mechanical invariants: the
market-signal fixture, the `--` + `About` blurb, en-dash-only prose, no round/stage named, no
founder name or LinkedIn link, and a response menu naming `voice note`. **Fix the draft and re-run
— do not reach for `--force`.** `--force` exists only for a deliberate, explained exception and
records `styleGateBypassed: true` in the snapshot.

The gate exists because "read STYLE.md before finalizing" is an instruction the model can skip, and
did (2026-07-31). It is the executable copy of the stylebook — when STYLE.md changes, update
`style_gate.py` in the same commit.

#### HTML Formatting Rules

Always send as `text/html`. Use **`<div>` blocks, NOT `<p>`** — Gmail's native compose emits `<div>content</div>` with `<div><br></div>` for blank lines, and `<p>` adds excess top margin that visibly inflates spacing compared to Tom's regular emails. Confirmed against rendered diff 2026-05-12.

**Critical rules:**
1. The first `<div>` MUST carry inline `style="margin:0;padding:0"` to override iOS Mail's default body top padding (otherwise renders as a phantom blank line above the greeting).
2. The entire `htmlBody` is a single continuous string with **zero whitespace between tags** (no newlines, no spaces between `</div><div>`).
3. Use `<div><br></div>` for every blank line between paragraphs.
4. No `<p>` elements anywhere.
5. No `<!DOCTYPE>`, `<html>`, or `<body>` wrappers — Gmail strips them anyway.

```html
<div style="margin:0;padding:0">Hey [First Name],</div><div><br></div><div>[Opener paragraph]</div><div><br></div><div>[Personalization paragraph — 2-4 sentences explaining why Tom thought of THIS person]</div><div><br></div><div>[No worries paragraph]</div><div><br></div><div>* [Question 1]</div><div><br></div><div>* [Question 2]</div><div><br></div><div>* [Question 3]</div><div><br></div><div>Best,</div><div>Tom</div><div><br></div><div>--</div><div><br></div><div><em>About [Company] (<a href="[URL]" style="color:#1155CC">[domain]</a>)</em></div><div><br></div><div>[Blurb sentence(s) — use founder's verbatim memo blurb when provided by Tom; otherwise auto-generate per Step 3]</div>
```

Key formatting rules:
- **No founder names and no founder LinkedIn links in the blurb** (Step 3). No Founder(s)-relation or
  ContactOut lookup is needed for this skill at all — the blurb ends at the product.
- The "About [Company]" header is italicized with `<em>`
- Body prose uses en dashes (`–`) only — never em dashes (`—`)
- The two blurb lead sentences are bold (`<strong>`)

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
