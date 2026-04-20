---
name: add-to-contacts
description: >-
  Create a new entry in the Notion People database from a LinkedIn profile or other contact information.
  Trigger when the user says "add to contacts", "new contact", "save this contact", "add this person",
  or provides a LinkedIn URL with intent to save someone's info. Also trigger when the user provides a
  person's name alongside contextual details (company, role, email, etc.) and wants them logged in the
  People database. Even if the user doesn't say "contacts" explicitly, trigger if the context clearly
  involves saving a person's information for future reference — not a deal opportunity (that's add-to-crm),
  but the person themselves.
---

# Add to Contacts

Create a new entry in the Notion People database from a LinkedIn profile or other contact source.

## Overview

Tom maintains a People database in Notion that serves as his contact book. When he encounters someone
worth tracking — a potential intro target, a fellow investor, a founder, a service provider — he wants
their key details captured quickly without manual data entry. The input is usually a LinkedIn URL, but
could also be a name with enough context to populate the record, a screenshot of a conversation, or a
forwarded email containing someone's details.

## Notion Target

- **Database:** 🤝 People
- **Data Source ID:** `1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`

## Fields to Populate

| Field | Type | Source Priority | Notes |
|-------|------|----------------|-------|
| Name | title | Profile name | Full name, e.g. "Matt Heiman" |
| Email | email | ContactOut MCP (`contactout_enrich_person`) → ContactOut browser sidebar → LinkedIn native Contact Info → blank | See email extraction below |
| LI | url | The LinkedIn profile URL | Always the full canonical URL |
| Company | text | Current primary position | Company name only, e.g. "Mercury". See primary role rules below. |
| Role | text | Current primary position | Title only, e.g. "Head of Product". See primary role rules below. |
| Category | select | Inferred from profile | See category inference rules below |
| City | select | Profile location | Map to metro area (see city rules below) |
| State | text | Profile location | 2-letter US state abbreviation; leave blank for non-US |

All other fields (relations, rollups, formulas, checkboxes, buttons) should be left untouched — they
are either computed or populated by other workflows.

## Workflow

### Step 1: Determine Input Type and Extract Data

**LinkedIn URL provided:**
1. Use Claude in Chrome browser tools to navigate to the LinkedIn profile URL.
2. Take a screenshot to read the profile — extract name, headline (company + role), and location.
3. **Email lookup — use ContactOut MCP first:** Call `contactout_enrich_person` with the LinkedIn
   URL. This is the preferred method — faster and more reliable than the browser sidebar. If the
   MCP returns a work email and/or personal email, use those. Prioritize the work email (company
   domain) unless this is a `-1` entry (see add-to-crm rules), in which case prioritize personal email.
4. **Profile photo extraction:** If ContactOut enrichment was performed via `contactout_enrich_linkedin_profile`
   or `contactout_enrich_person`, check for a `profile_picture_url` in the response. Save this URL
   for use as the Notion page icon in Step 4. If no ContactOut profile photo is available, skip —
   do not set a page icon (no emoji fallback for People entries).
5. **Fallback — ContactOut browser sidebar:** If the MCP returns no email, check for a ContactOut
   extension sidebar on the right side of the LinkedIn page. If present, read the email addresses
   it displays and apply the same work/personal priority logic.
6. **Fallback — LinkedIn native:** If neither ContactOut method yields an email, click "Contact info"
   on the LinkedIn profile to open the native contact info modal. Read any email addresses displayed
   there. If no email is available anywhere, leave the Email field blank.

**Name + contextual info (no LinkedIn URL):**
The user might say something like "add Matt Heiman from Mercury to contacts — angel investor, NYC."
Extract whatever is provided directly. If a LinkedIn URL is not given but enough context exists to
search for one, offer to look the person up on LinkedIn to fill in gaps — but don't block on it.
Create the entry with whatever information is available.

**Screenshot or forwarded email:**
Parse the image or email text for name, company, role, email, location. If a LinkedIn URL is visible,
navigate to it for additional data.

### Identifying the Primary Role

Many people on LinkedIn list multiple positions — a full-time job, an angel investing side activity,
a venture scout role, advisory positions, board seats, etc. The Company and Role fields should
reflect the person's **primary, full-time role** — the thing they actually do day-to-day — not
secondary activities.

Concretely: if someone is VP of Engineering at Stripe and also an angel investor on the side, their
Company is "Stripe" and their Role is "VP of Engineering." The angel investing is secondary. Same
logic applies to venture scout roles, advisory positions, board memberships, and similar affiliations
that are typically part-time or informal.

The way to identify the primary role is usually straightforward — it's the position listed first on
LinkedIn (the "current" position with the longest tenure or the one that reads as full-time). If the
headline explicitly leads with a side activity (e.g., "Angel Investor | VP Eng @ Stripe"), look at
the Experience section to determine which is the actual day job.

The one exception is people whose primary professional identity genuinely is investing — a full-time
VC partner, a fund GP, a family office principal. In those cases, the investing role *is* the primary
role.

### Role Formatting: Compound Founder/C-Level Titles

When someone's title combines "Co-founder" (or "Co-Founder") with a C-level role (CEO, CTO, COO,
CRO, CPO, etc.), use **only the C-level title** in the Role field. Do not list both.

- "CEO & Co-founder" → Role: **CEO**
- "Co-Founder & CTO" → Role: **CTO**
- "Co-founder / CEO" → Role: **CEO**

If the title is just "Founder" or "Co-founder" with no C-level role attached, use that as-is:

- "Founder" → Role: **Founder**
- "Co-Founder" → Role: **Co-Founder**

### Step 2: Infer Category

Map the person's profile to one of these categories based on what you observe:

- **Corporate** — works at a large established company (Fortune 500, Big 4, major banks, big tech, etc.)
- **Startup** — works at or founded a startup / early-stage company
- **Investor** — angel investor, VC, PE, family office, fund manager
- **Services** — lawyer, accountant, consultant, recruiter, banker (professional services)
- **Legal** — specifically a lawyer or at a law firm
- **Student** — currently enrolled in school, recent grad with no full-time role
- **Other** — doesn't fit neatly into the above
- **N/A** — not enough information to determine

Use judgment. Someone who is both an angel investor and a startup founder should be categorized
based on what appears to be their primary current activity. When in doubt, go with the more
specific category rather than N/A.

### Step 3: Map City and State

**City:** Use the metro area name, not the specific neighborhood or suburb. For example:
- Brooklyn, Queens, Manhattan → **New York**
- Palo Alto, Mountain View, Sunnyvale → **San Francisco** (unless the exact city is in the list)
- Arlington, VA or Bethesda, MD → **Washington** (or **Washington DC** or **Bethesda** if those are better matches)

The City field is a select with pre-existing options. If the person's city matches an existing option,
use it. If not, it's fine to create a new option — Notion will handle that automatically.

**State:** 2-letter US abbreviation (NY, CA, IL, etc.). For international locations, leave blank.

### Step 4: Create the Notion Page

Use `notion-create-pages` with:
```
parent: {"data_source_id": "1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9"}
```

Set the properties as extracted. Only include fields where you have actual data — don't set fields
to empty strings or placeholders.

**Page body: always leave blank.** Do not add any text content to the page body — no bio, no
summary, no notes. Properties only. Tom manages body content separately.

**Page icon — LinkedIn profile photo:** If a `profile_picture_url` was captured in Step 1, set
it as the page `icon` (external image URL) when creating the page. This makes the People database
visually scannable. If no profile photo URL is available, omit the icon field entirely — do not
fall back to an emoji for People entries.

**Do not confirm with the user before creating** — Tom wants this to be fast. If the LinkedIn
profile is clear and unambiguous, just create the entry. If something is genuinely unclear (e.g.,
you can't tell which of two companies is the current one), ask.

### Step 5: Report Back

After creating the entry, provide a brief confirmation with the person's name and a link to the
new Notion page. Keep it concise — one or two lines is fine.

## Example

**User:** `add to contacts https://www.linkedin.com/in/heiman/`

**Claude:**
1. Navigates to the LinkedIn profile in Chrome
2. Reads headline: "Angel Investor" at Mercury — but checks Experience section
3. Experience shows primary role is **Head of Corporate Development & Investor Relations** (full-time, 2+ years).
   "Angel Investor" is a secondary activity, not the day job. Uses the primary role.
4. Calls `contactout_enrich_person` (MCP) → returns mrh2118@gmail.com and mheiman@mercury.com
5. Picks mheiman@mercury.com as the work email (company domain match)
6. Captures `profile_picture_url` from ContactOut response for use as page icon
7. Creates Notion People entry:
   - Name: Matt Heiman
   - Email: mheiman@mercury.com
   - LI: https://www.linkedin.com/in/heiman/
   - Company: Mercury
   - Role: Head of Corporate Development & Investor Relations
   - Category: Startup
   - City: New York
   - State: NY
   - Icon: (ContactOut profile photo URL)
8. Returns: "Added Matt Heiman (Mercury) to People. [link]"
