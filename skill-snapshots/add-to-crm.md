---
name: add-to-crm
description: Create a new opportunity in the Notion CRM pipeline. Trigger when the user says "add to crm" or similar phrases like "log this deal", "add this opportunity", "crm this". Also trigger when the user uses the shorthand "-1" followed by a LinkedIn URL or name — this notation means "create a pre-company opportunity" (equivalent to add-to-crm with a -1 title). The user will provide source material in one of these forms - a screenshot (of a text message, LinkedIn DM, or other conversation), a forwarded email, a LinkedIn profile URL, or pasted text. Extract all available deal information, populate the Notion Opportunities database, and attach any deck materials.
---

# Add to CRM

Create a new opportunity in the Notion Opportunities database from varied source material.

Read `references/schema.md` for the full database schema and field formatting rules before proceeding.

## Protected Status Guard

Before creating ANY new opportunity, run the dedup check below. This is MANDATORY — do not skip it. Unattended callers (e.g. `inbound-deal-detect`) must run this guard the same way as manual invocations.

### Dedup procedure (run all three — don't short-circuit on a single miss)

1. **Title match.** Call `notion-search` with `data_source_url: "collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"` and the extracted company name as the query. Check results whose title matches the company name exactly OR with a `(Series X FO)` / `(Seed FO)` / similar follow-on suffix.
2. **Website-domain match.** If a website is extracted (e.g. `tuor.dev`), also query with the bare domain — Notion semantic search indexes page properties, so this surfaces rows where only the `Website` field matches.
3. **Contact-email match.** If a founder email is extracted (e.g. `hardik@tuor.dev`), also query with that email — this catches cases where the company was logged under a different spelling/casing.

Collect the union of matches from all three queries, then fetch each candidate page and read its `Status`.

### Filter name-only collisions

Multiple unrelated companies can share a name (e.g., wallet "Remi" at `remiwallet.io` sourced by Madi Jacox vs. a different "Remi" at `remi.site` from Ameya Jadhav, intro'd by Jake Kupperman — same title, zero shared signal). For each candidate from the title query, require **at least one corroborating signal** beyond the bare name match:

- Shared **website/domain stem** — candidate's `Website` shares a domain stem with the new lead's website or any extracted email domain
- Shared **Contact email** — candidate's `Contact` (which is a `; `-separated alias list) contains any of the new lead's emails
- Shared **founder identity** — candidate's `🏁 Founder(s)` relation includes a Person whose name matches the new lead's founder
- Shared **Source person** — candidate's `Source(s)` relation includes the inbound sender

If a candidate matches ONLY on title with no corroboration, **drop it from the dedup set** — treat as a distinct company and proceed. Candidates from the website-domain or contact-email queries (steps 2–3 above) are already corroborated by definition and skip this filter.

This filter applies to the same caller scope as the rest of the guard: manual `add-to-crm`, unattended `inbound-deal-detect`, and any webhook that does name-based Opp resolution before flipping status (e.g., `intro-connected-detect`'s Pass 5 `findAnyOppByNameFuzzy` path — not yet code-synced; mirror manually when touching the webhook).

### Decision table

- **Any candidate has Status ∈ {Active Portfolio, Portfolio: Follow-On, Exited, Committed}** — do NOT create a duplicate. Do NOT modify the existing page's Status. In manual mode, alert the user with the existing page URL and ask how to proceed. In unattended mode, log `protected-status-skip` with the existing page ID and exit 0. The caller (e.g. `inbound-deal-detect`) is responsible for posting the `🛡️` Slack alert.
- **Any candidate has Status ∈ {Pass (Met), Pass (DNM), Lost, NR / Missed}** — hard-block. Do NOT create a duplicate. In manual mode, alert the user. In unattended mode, log `prior-pass-skip` with the existing page ID and exit 0.
- **Any candidate is in an in-progress status (Qualified / Outreach / Connected / Scheduled / Active / Track / Exploration / Assigned / Pass Note Pending)** — this is a re-surface of an existing live deal. Do NOT create a duplicate. In manual mode, surface the existing page and ask whether to update it in place. In unattended mode, log `duplicate-in-pipeline-skip` with the existing page ID and exit 0.
- **No candidates found** — safe to proceed with creation.

**Why this matters:** A portfolio founder emailing from a work address that isn't on their People DB row will slip past the webhook's founder-sender gate. Without this guard firing in `add-to-crm`, the result is a duplicate Opportunity row shadowing an Active Portfolio company. This guard is the last line of defense.

## Workflow

1. Determine input type and extract data
2. **Enrich from attachments if email body is thin (Step 1B)**
3. Enrich from founder LinkedIn via ContactOut (HQ, Contact, Website, Description)
4. Present extracted fields to user for confirmation
5. Create the Notion opportunity page (with a thematic emoji as icon)
6. Handle any deck/material uploads

## Step 1: Extract Data from Source

Determine the input type and extract all available data points:

**Screenshot (text message or LinkedIn DM):** Read the image directly (multimodal). Extract: names, company mentions, round details, referrer, links, context.

**Forwarded email:** Parse the email text. Extract: company name, founder info, round details, deck links, referrer (from/cc), website, contact email. The email body often contains a pitch or intro.

**ALWAYS read the full email thread** via `gmail_read_thread`, not just the single message. A forwarded pitch is often only the opening move — subsequent messages in the thread (and in any related threads) frequently contain: Tom's reply expressing interest, a second thread where the referrer makes the actual intro, the founder's acknowledgment, scheduling coordination via Blockit/Calendly, and the call confirmation. All of this changes how the opportunity should be classified (see Step 4). Also run a broad Gmail search on the company name to surface related threads (the initial forward and the follow-up intro are often separate threads with different subject lines — e.g. "Fwd: Company.ai" followed by "Connecting Tom <> Founder (Company)").

**LinkedIn profile URL:** Use `WebFetch` on the URL. If it fails or returns limited data, use Claude in Chrome browser tools (navigate to URL, read page). Extract: name, headline, location, current company, about section. **When the input is a bare LI profile URL (no other context), apply these defaults: Stage → `Pre-Seed 💡`, Source → "Direct", Description → `TBD`.**

**`-1` shorthand:** When the user writes `-1` followed by a LinkedIn URL or name, this is equivalent to "add to CRM" with a pre-company `-1` title format. Treat it identically to a bare LinkedIn profile URL input. For `-1` entries, when looking up email via ContactOut MCP (`contactout_enrich_person`) or browser sidebar: **prioritize personal email** (e.g. Gmail) over work email for the People DB `Email` field, since the person is not yet attached to a specific company from Tom's pipeline perspective. Always try the ContactOut MCP first before falling back to the browser sidebar.

**Pasted text:** Parse directly for all relevant data points.

Target fields to extract (leave blank if not found):
- Company name
- Founder name(s) and LinkedIn URL(s)
- One-liner description (keep to a single sentence)
- Location / HQ
- Website
- Contact email
- Stage (Pre-Seed, Seed, etc.)
- Round details (amount, cap/valuation)
- Who referred / sourced the deal
- Any deck or material links (DocSend, Google Drive, etc.)

## Step 1B: Enrich from Attachments (Thin Email Body)

After extracting data from the source, assess whether the extracted fields are sufficient to create a well-populated CRM entry. An email body is considered **thin** if it is missing 2 or more of these key fields: Description (a real product one-liner, not just round context), Founder identity (named founders in the page body Team section + linked Founder(s) relation), Round Details (specific dollar amounts), HQ, or Website.

**If the email has PDF attachments and the extracted data is thin, you MUST read the attachments before creating the Notion page.** The attachment content takes priority over the email body for all fields — the email body is often just a brief cover note while the real deal data lives in the deck or memo.

### How to read attachments

1. **Save attachments to Drive** via the Gmail Attachment Saver Apps Script (see `/Users/tomseo/.claude/skills/shared-references/gmail-attachment-saver.md`). Use the Diligence root folder (`1QINUouO6CpJ7iZa0HF2LHL6kK8hm612d`) as the target — or create per-company subfolders first via the Drive Upload Apps Script `createFolder` action (see `/Users/tomseo/.claude/skills/shared-references/drive-upload.md`). This works in all environments.

2. **Read the PDFs via Chrome** — navigate to the Drive file URL (`https://drive.google.com/file/d/<fileId>/view`) and use `get_page_text` to extract the content. Google Drive's built-in PDF viewer renders the text, making it accessible without downloading the binary.

3. **If Chrome is unavailable** — fall back to creating the Notion page with whatever data the email body provides, but **mark the page with `⚠️ Incomplete — attachment data not extracted` at the top of the page body** so it is obvious the entry needs manual enrichment. Do NOT echo the thin email text into fields like Description or Round Details as a substitute for real data — leave those fields blank or set to `TBD` rather than populate them with non-substantive content.

4. **Extract fields from the PDF content** and merge with what was already extracted from the email body. PDF-sourced data overrides email-body-sourced data where both exist (e.g., if the email says "raising SPV" but the memo says "$3.5m on $27m post SAFE", use the memo's round details).

### What NOT to do

- Do NOT create a CRM entry with Description set to the email's brief context line (e.g., "Seed extension round. Will raise SPV in May.") when a full investment memo is attached.
- Do NOT skip reading attachments because the email body technically has a company name and a vague round mention — those are insufficient for a useful pipeline entry.
- Do NOT defer attachment reading to Step 6 (Materials Handler) — by that point the Notion page is already created with empty/wrong fields. Materials Handler handles the Drive upload and Notion linking; this step handles content extraction for field population.

### When to skip this step

- The email body already contains rich deal data (founder names, product description, round specifics, etc.) — no need to read attachments for field population (though Step 6 will still handle uploading them).
- The source is not an email (screenshot, LinkedIn URL, pasted text) — attachments are not relevant.
- The email has no attachments.

## Step 2: Enrich from Founder LinkedIn + Web (ContactOut as fallback)

Always enrich `HQ`, `Contact`, `Website`, `Description` **before creating the page** — whether or not the source includes a founder LinkedIn URL. (Icon comes from Notion's logo picker pattern in Step 5 — no enrichment needed.)

**Priority order — ContactOut is a paid API, only hit it if the cheaper sources can't answer:**

1. **Source material** — email signatures, forwarded threads, pasted context often contain the founder's email, company website, and HQ verbatim. Check first.
2. **Public LinkedIn page** — WebFetch (or Chrome if logged-in detail is needed) the founder's LI URL for headline, location, current-company name, and bio. Pulls HQ and a description without burning credits. **If no LI URL was provided in the source, web-search `"<founder name>" "<company>" linkedin` (or `"<founder name>" <distinctive context> linkedin` if the company is generic) to find it before falling back to ContactOut.** Disambiguate by description signals (role, ex-employers, school) when multiple matches exist.
3. **Company website** — WebFetch for HQ (footer/contact page) and description (hero/meta).
4. **ContactOut fallback** — only if 1–3 can't resolve a needed field, call `contactout_enrich_linkedin_profile` with `profile_only=false`. It returns `email`/`personal_email`, `company.headquarter`, `company.website`, and full experience. Use `profile_only=true` if you only need profile data (no email credits).

**Case-by-case source: YC company page.** If (and only if) the founder's LI headline, the source material, or the company website mentions YC (e.g. "CEO @ Pluto (YC W25)", "YC S24"), slot in a fetch of `https://www.ycombinator.com/companies/{slug}` — it gives authoritative HQ, website, and one-liner for YC-backed companies. Skip for everything else (the vast majority of deals are not YC).

If `company.headquarter` from any source isn't conclusive (e.g. LI lists "Stealth YC Startup"), the company website (or YC page if YC-backed) wins over a stealth-placeholder LinkedIn company.

If after all lookups HQ is still unresolvable, use `??? 🌀` — but treat that as a failure to investigate, not an acceptable end-state. Same for `Contact=N/A`, `Website=N/A`, `Description=TBD`.

## Step 3: Resolve Founder Contact Email

Look for the founder's contact email using the following priority order. Stop as soon as a valid email is found:

1. **Source material first (always check before anything else):**
   - Parse the email thread for founder email addresses (From, To, CC fields, email body, signature blocks).
   - Check any attached or linked deck, memo, or pitch document for contact info (often on the last slide or in a cover page).
   - Check the forwarded message chain — the founder's email may appear in an earlier reply or in the original sender field.
   - If the source is a screenshot, look for email addresses visible in the image.

   **Custom-domain wins over personal/free providers.** If the source yields BOTH a custom-domain email (`john@highroad.capital`) AND a personal/free email (`jd4life217@gmail.com`) for the same founder, set Contact to the custom-domain email **alone** — do not alias-merge them. The custom-domain `From:` line in a forwarded thread is the authoritative signal: it's how the founder represents themselves to the world, and it survives gmail-account churn. Free-provider domains include `gmail.com`, `googlemail.com`, `yahoo.com`, `hotmail.com`, `outlook.com`, `live.com`, `icloud.com`, `me.com`, `protonmail.com`, `proton.me`, `aol.com`. The `;`-separated alias-merge pattern (per `feedback_upgrade_opp_contact_to_custom_domain_asap`) is reserved for the resolution/webhook layer when a custom-domain email is discovered AFTER initial creation — at creation time we already know the best one, so write it cleanly.

2. **ContactOut lookup (only if source material yields nothing):**
   - **LinkedIn URL lookup** (preferred): Use `contactout_enrich_person` with the founder's LinkedIn URL. Set `include_work_email=true` and `include_personal_email=true`.
   - **Name + company fallback**: If the LinkedIn lookup returns no email, try `contactout_enrich_person` with `full_name` + `company` array.

3. **Domain inference fallback (last resort):** If ContactOut also returns nothing, infer email from the company domain if known (e.g., `vincent@pantainsure.com`). Flag inferred emails as unverified in the page body.

For `-1` opportunities (pre-company / bare LinkedIn profiles), **prioritize personal email** (e.g. Gmail) over work email, since the person is not yet attached to a specific company from Tom's pipeline perspective.

If ContactOut returns multiple emails, prefer the work email for named-company opportunities and personal email for `-1` entries. **If ContactOut returns a different email than what the source material's `From:` header showed, the source material wins** — the founder's authoritative `From:` is the strongest signal of how they want to be contacted, regardless of what ContactOut surfaces.

Set the `Contact` property on the Notion page to the best email found. If no email is found after all attempts, set `Contact` to `N/A`.

## Step 4: Present Summary (no confirmation gate)

Present extracted data as a concise summary in the response, then proceed directly to Step 5. **Do not ask Tom to confirm before creating the page** — the summary is for after-the-fact course correction, not a yes/no gate. Include:
- Proposed page title (company name or `-1 (Founder Name)` format)
- Description (one-liner)
- Status (inferred from thread state — `Qualified`, `Outreach`, `Connected`, or `Scheduled` — per rules in Step 5), Stage, HQ
- Website, Contact, Founder name(s)
- Round Details (if any)
- Source person (if identified)
- Close Date (if Status is `Scheduled`)
- Any materials/decks to upload

**Exception — DO ask** if something mid-flow is genuinely ambiguous (multiple equally-plausible LinkedIn matches for a founder, two People-DB rows that could be the source, conflicting round details across email + deck). Confirmation isn't banned; the rote post-summary "Ship it?" gate is.

## Step 5: Create the Notion Page

Use `notion-create-pages` with parent `{"data_source_id": "fab5ada3-5ea1-44b0-8eb7-3f1120aadda6"}`.

**Before the `notion-create-pages` call, run `touch /tmp/.addcrm-bypass`** to set the hook bypass marker. A PreToolUse hook at `~/.claude/hooks/gate-opps-creation.sh` blocks all direct writes to the Opportunities data source unless this marker is fresh (≤5 min old). The marker auto-expires, so no cleanup is needed. If the hook denies a call with "Direct creation of Notion Opportunities-DB rows is gated", that's the signal you forgot this step.

### Page Icon

**Always set a thematic emoji as the page icon — never ship a new Opportunity with a blank/default icon.** Pick an emoji from Notion's standard emoji set that gestures at the company's product or category (e.g. memory app → 💭, payments → 💳, biotech → 🧬). Pass it as the `icon` parameter to `notion-create-pages` at creation time (or `notion-update-page` immediately after). Don't hunt for company logos, favicons, or external URLs — Notion's emoji library is rich enough to cover any company.

### Title Conventions

Always hyperlink founder name(s) to their LinkedIn URL(s) in the title using Notion inline markdown link syntax `[Name](url)`. This gives quick access to founder profiles directly from the pipeline board.

- **Company name known:** Title is the company name. Example: `Tuor`
- **No company name (very early stage):** Use `-1 ([First Last](linkedin_url))` format. Multiple founders: `-1 ([First Last](url) & [First Last](url))`. **Any `-1` opportunity always defaults to `Pre-Seed 💡` stage, regardless of what external sources (Crunchbase, etc.) say about the company's fundraising history.** The `-1` designation means the deal is pre-company from Tom's pipeline perspective, so the stage should reflect that.

### Property Rules

- `Status`: Infer from the full thread state. **If the caller (e.g. `inbound-deal-detect`) passed an explicit `status` directive, honor it verbatim — do not re-infer.** Otherwise apply the defaults below:
  - `Qualified` — deal is only **surfaced**: a forward of someone else's content, a LinkedIn URL, a pasted profile, a tip from a third party. No founder contact has occurred yet.
  - **`Connected` — cold inbound founder email** where the founder is the actual sender (deal-scanner webhook path, or `forwardedFromTom: true` AND the inner sender's email domain matches the pitched company). The founder pinged Tom directly and the email landed in his inbox; Tom is in a live thread with the founder from message #1, even before he replies. Do NOT use Qualified for these.
  - **`Connected` — double-opt-in intro thread**: a referrer sends an intro email putting Tom and the founder on the same To/Cc line (e.g. "Connecting Tom <> Founder"), or Tom replies into that intro thread with the founder still on it. The intro thread itself IS the connection — Tom and the founder are now in a live email channel. Set `Connected` from the moment that thread exists, even if the founder hasn't typed a response yet. Do NOT default to `Outreach` here — Outreach is reserved for cold direct outreach where no third party has bridged Tom to the founder.
  - **Third-party referrer forward** — covers BOTH `forwardedFromReferrer: true` (a third party sent Tom a forward of a founder's pitch) AND `forwardedFromTom: true` where the inner sender's domain does NOT match the pitched company (e.g. `lurein@givecard.io` forwarding John Daniels's HighRoad pitch through Tom's `tom@dashfund.co` alias). In both shapes the founder is NOT in the thread with Tom yet — the referrer is offering an intro, not making one. Default to `Qualified` if Tom hasn't replied, or `Outreach` if Tom has replied opting in to the intro. Do NOT use `Connected` — Tom has not been connected to the founder yet. (Once the referrer follows up by sending a separate intro thread that puts Tom + founder together, the rule above takes over and status flips to `Connected`.)
  - `Outreach` — Tom has reached out cold directly to the founder and is waiting for a first response, no third party has bridged them. Also covers the post-`Qualified` state where Tom replies to a referrer opting in to an intro but the referrer hasn't yet looped the founder in.
  - `Connected` — generic fallback: founder has replied to Tom's outreach and the conversation is live but unscheduled.
  - `Scheduled` — a call is actually booked (Blockit/Calendly confirmation, calendar invite, or explicit time agreed in-thread).

  Read the full Gmail thread before deciding — never default to `Qualified` without first checking whether the thread already shows later-stage progression OR whether the founder is the original sender. If the user specifies a status explicitly, honor that instead.

  **Detecting third-party referrer forward**: subject starts with `Fwd:`/`FW:`, AND either the outermost `From:` is not Tom AND not the founder (`forwardedFromReferrer` case), OR the outermost `From:` is Tom but the inner sender is not the founder either (`forwardedFromTom` + 2-level forward case — e.g. Lurein forwarded John's pitch to Tom's `tom@dashfund.co`, which Tom then bounced to `tom@invertedcap.com`). The unifying signal is: **the inner sender's email domain does NOT match the pitched company domain** (e.g. `lurein@givecard.io` doesn't match `highroad.capital`; `madi@dormroomfund.com` doesn't match `connectupskill.com`). When this discriminator fires, treat as referrer forward regardless of which `forwardedFrom*` flag the webhook set.
- `date:Close Date:start`: Set to the scheduled call date when Status is `Scheduled` (this anchors the pipeline agent's close-date logic). Leave blank otherwise — the pipeline agent will manage close dates for earlier stages.
- `Website`: Infer from source material (email body links, founder email domain if company domain). `N/A` if unavailable.
- `Contact`: Set to the founder's email if found (per Step 3 priority order). **Never leave this field empty/blank** — if no email is found after all lookup attempts, always set to `N/A`.
- `Description`: One-liner only. Extra context goes in page body.
- `Round Details`: **Strict format, two valid shapes only.** Either `Raising $Xm` / `Raising $X-Ym` for unfinalized rounds (no terms set), OR `$Xm on $Ym post` / `$Xm on $Ym cap` for rounds with terms set. Lowercase `m`/`k`. **Leave blank (null) if the source doesn't disclose a specific dollar amount or cap/post.** Never write timing-only or qualitative substitutes like `Kicking off seed this week`, `Raising soon`, `Active round`, `Closing this month`, `Seed extension` — those describe round *status* or *timing*, not round *terms*, and they belong in the page body (Source Blurb / Source Context), not the property field. If the email body doesn't disclose terms, **read the attached deck/memo before defaulting to blank** (Step 1B handles this) — round terms are most commonly buried in the deck's fundraise slide. If after reading the deck there's still no $ figure, blank is the correct answer; an inbound-deal-detect upstream hint of `""` is also correct. See `references/schema.md` for additional examples.
- `Followed Up`: Always `__NO__`
- `Fund`: Default `Inverted 1️⃣` unless user specifies otherwise
- `Inv @ Round`: Only set if explicitly known
- `Stage`: Use exact emoji variant from schema.md (e.g. `Pre-Seed 💡`, `Seed 🌾`). Default to `Pre-Seed 💡` for `-1` opportunities. For named companies, infer stage from source material (round details, explicit mentions).
- `Support`: Always set to the "N/A" entry: `https://www.notion.so/18200beff4aa80bc8344fc48c7b0fdb1`
- `Source Thread ID`: Set to the Gmail `threadId` when the source is a Gmail message (e.g. invocation by `inbound-deal-detect`, or any flow where the input is a forwarded email or referral thread). Leave blank for non-email sources (LinkedIn URL, screenshot, pasted text, manual entry). This field powers `outreach-detector` / `outreach-decliner` Path D — when Tom replies in the original referral thread, the webhook flips Status deterministically by threadId match instead of subject/body haystack guessing.

### Page Content Structure

See `references/schema.md` for the canonical page body structure. Key sections: **Summary**, **Source Blurb** (if source material contains a company blurb), **Team**, **Round**, **Diligence Materials** (if any deck/material links exist), **Source Context** (with Gmail deep link if from email).

```
**Summary**
[Paragraph description — richer than the one-liner property, covering product, market, and context.]

**Source Blurb**
[If the source email, message, or document contains a pitch blurb or company description, reproduce it here verbatim. Omit this section if no blurb is present in the source material.]

**Team**
[Founder Name](LinkedIn URL) — [brief bio/background if available].

**Round**
[Round details if known, otherwise omit this section.]

**Diligence Materials**
[Only include if deck/material links exist. List each as a labeled bullet.]
- [Document Title](Drive URL)

**IMPORTANT:** If DocSend materials were converted to PDF and uploaded to Drive as part of this flow, the Diligence Materials section must link ONLY to the Drive PDFs — never to the original `docsend.com/view/...` URLs. DocSend links are transient input; the Drive PDFs are the permanent artifact. This also applies to other sections (e.g. Round) — do not reference DocSend as the source of materials anywhere in the page body.

**URL fidelity — never fabricate URLs in the page body.** Any URL written into the page body (deck links, demo URLs, GitHub repos, founder LinkedIn, company website, social profiles, third-party deck-sharing platforms, etc.) MUST appear as a literal substring of the source material (email body, screenshot OCR, deck text, pasted text). Before writing a link, verify the URL is present in the source — `grep -F "<url>" <source>` mentally. If the URL is not literally in the source, omit the link entirely and reference the artifact by name instead (e.g., `**Deck**: see attached` rather than `[Deck](https://docsend.com/view/<fabricated>)`). This rule overrides "be helpful by filling in plausible URLs" — a missing link is recoverable; a wrong link wastes Tom's time and corrupts the audit trail. Applies equally to: the Diligence Materials body section, the Source Context section, the Team section's founder LI links (if the LI URL wasn't in source, leave the founder name unlinked), and any inline references elsewhere on the page.

**Source Context**
[Raw source material: full email text, transcribed screenshot text, DM content, etc. Include Gmail deep link if from email.]
```

**Source Blurb formatting rules:**
- Use `**Source Blurb**` as the section header — bolded, same style as all other section headers, no italics, no attribution in the header.
- Reproduce the blurb text exactly as written in the source — do not paraphrase or edit.
- This section is for any company description, pitch excerpt, or blurb found in the source material (email body, forwarded intro, etc.). It is separate from Source Context, which holds the full raw message.

### Relation Fields (resolve before creating)

Before creating the page, resolve these relation fields. **All relation lookups for Source(s) and Founder(s) must search the People DB exclusively** — not the workspace at large. Other databases (e.g. LP Directory) may have entries with the same name, but the Opportunities relation fields point to the People DB (`collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`), so linking a page from any other database will silently fail to display.

- **Source(s)**: Identify who referred/sourced the deal from context (email sender, person who texted, person who made the intro, etc.). Search the **People DB only** using `notion-search` with `data_source_url` set to `collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9`. If a matching entry is found, include their page URL in the `Source(s)` property at creation time. If not found, leave blank. **If no source/referrer is identified (e.g. user found the profile directly), default to "Direct": `https://www.notion.so/0fb9a64034fd46f9934768d590e69dc9`.**
- **🏁 Founder(s)**: Search the **People DB only** (same `data_source_url` as above) for the founder's name. If a matching entry is found, include their page URL at creation time. **If no match is found, leave the relation blank — never auto-create a People row from this flow.** Call out the gap in the confirmation summary to Step 4 (e.g. "🏁 Founder(s): not in People DB — link blank, add manually if desired") so Tom can decide whether to add them via `add-to-contacts`. When Tom DOES authorize a People creation, the `add-to-contacts` flow is responsible for populating both `City` AND `State` — never just City.
- **Support**: Always set to `https://www.notion.so/18200beff4aa80bc8344fc48c7b0fdb1` (the "N/A" entry).

## Step 6: Handle Decks and Materials

If the source email, screenshot, or pasted text contains any diligence materials (deck links, DocSend URLs, Google Drive share links, Gmail attachments), delegate the full materials-handling flow to the `materials-handler` skill.

**Note:** If Step 1B already saved attachments to Drive and read them for field extraction, pass the Drive file IDs and URLs from that step to materials-handler so it does not re-upload them. Materials-handler still needs to run for: creating per-company subfolders (if not already done), moving files into subfolders, updating the Notion page body Diligence Materials section, and linking files in the Notion Diligence Materials property field via Chrome.

### How to delegate

Read the `materials-handler` skill at `/Users/tomseo/.claude/skills/materials-handler/SKILL.md` and follow its instructions, passing:

- **Company name**: the opportunity title just created
- **Notion page ID**: the page ID returned by `notion-create-pages`
- **Material URLs**: any deck or material URL extracted from the source — DocSend links, Google Drive links, Dropbox links, direct PDF URLs, AND third-party deck-sharing platforms (`brieflink.com`, `pitch.com`, `decko`, similar). Pass all of them; the materials-handler decides whether to convert-to-PDF or link as-is.
- **Gmail message ID**: if the source was a forwarded email, pass the message ID so materials-handler can find and process attachments
- **Pre-saved file IDs** (if Step 1B ran): pass the Drive file IDs and URLs so materials-handler skips re-uploading

The materials-handler skill will determine whether Chrome is available and execute the appropriate path — full Drive upload flow when Chrome is up, or lightweight deep-link fallback when it's not. File uploads go through the Drive Upload Apps Script either way.

### What materials-handler covers

- Gmail attachments → Apps Script endpoint → saves directly to Diligence folder → link in Notion property + body
- DocSend links → Python PDF conversion → Drive upload → link in Notion
- Direct file URLs (Google Drive, Dropbox, raw PDF) → Drive upload or direct linking
- **Third-party deck-sharing URLs (brieflink.com, pitch.com, etc.)** → no PDF conversion attempted; link the URL as-is in the Notion Diligence Materials property field
- Notion page body Diligence Materials section (append or create)
- Notion Diligence Materials property field (Chrome-only) — populated for ALL deck/material URLs, not just converted PDFs. The property field is Tom's actionable surface; never skip it just because the URL didn't fit a PDF-conversion path.
- Graceful fallback to Gmail deep links + "pending upload" notes when Chrome is unavailable

### What add-to-crm still handles

If **no materials are present** in the source (no attachments, no deck links, no DocSend URLs), skip this step entirely — there's nothing to delegate.

If materials-handler encounters an error or stalls, fall back to adding a **Diligence Materials** section in the page body with whatever links or references are available from the source material, plus a note for manual follow-up.

## Examples

### Email Input

Extracted: Name=Tuor, Description="Human-in-the-loop infrastructure for AI.", Stage=Pre-Seed 💡, HQ=New York, Website=tuor.dev, Contact=hardik.gupta18@gmail.com, Founder=Hardik, Round=$1m.

Properties set accordingly. Page body:
```
**Summary**
Tuor helps enterprises deploy AI with confidence by routing model outputs to domain experts for review before reaching end users. API-first, integrates with existing ML pipelines.

**Source Blurb**
[Verbatim blurb from the intro email or source material, if present.]

**Team**
[Hardik Gupta](https://linkedin.com/in/hardikgupta/) — ML engineer, ex-MaxHome and Arthur AI. MS Data Science from Harvard, dual BS from UPenn.

**Source Context**
[How the deal was sourced, Gmail deep link if available.]
```

### Email Input with Thread Progression

Initial email from Ben Futoriansky forwards Armin Aghaei's Spangler.ai pitch. Reading the full thread reveals Tom replied "please intro", a second thread ("Connecting Tom <> Armin (Spangler)") shows Ben made the intro, Armin acknowledged, and Blockit coordinated a call for Mon Apr 20 at 11:30am EDT.

**Do not default to Qualified.** The thread shows the deal has already progressed through Outreach → Connected → Scheduled in the span of the same day. Correct classification:
- Status: `Scheduled`
- Close Date: 2026-04-20 (call date)

Include in Source Context a brief timeline of the thread progression so the state is auditable on the page itself.

### Email with Thin Body + PDF Attachments

Email from Bobby Kwon says: "Playground: Raising SPV for this now. Creating a much more substantial dataroom. Watchful: Will raise SPV for this in May." Two PDF attachments: "Playground Teaser (Apr 26).pdf" and "Watchful Investment Memo (Seed-Ext).pdf".

**Step 1 extracts:** Company names (Playground, Watchful), vague round context, source (Bobby Kwon). Missing: Description, Founder names, specific round details, HQ, Website. This is a thin email body — triggers Step 1B.

**Step 1B:** Save PDFs to Drive via Apps Script → open in Chrome via Drive viewer → extract full deal data from each memo. Result: Playground has $15m ARR, $6m round at $180m post, founders Daniel/Josh/Sasha Andrews + Sasha Reiss, HQ New York, website tryplayground.com. Watchful has $600k ARR, $3.5m SAFE at $27m post, founder Josh Parsons, HQ New Zealand (→ `??? 🌀`), no website found.

**Proceed to Step 2+ with enriched data.** The Notion pages are created fully populated rather than with placeholder text echoed from the email body.

### LinkedIn Profile URL (no company)

User provides `https://linkedin.com/in/janedoe`. No company name → title: `-1 ([Jane Doe](https://linkedin.com/in/janedoe))`. Extract location for HQ. Description from headline. Website=N/A, Contact=N/A. Stage=Pre-Seed 💡 (default for bare LI profile). Source=Direct. Description=TBD.

### Screenshot of Text Message

Screenshot says: "Check out Acme — AI supply chain tools. Founder John Smith, Chicago. Raising $2m seed on $10m cap."

Extracted: Name=Acme, Stage=Seed 🌾, HQ=Chicago, Round Details=$2m on $10m cap, Founder=John. Source=[text sender if identifiable].

## Behavior Rules

### Pull founder info from deck/website by default — never hallucinate

When running add-to-crm, if the email body doesn't name the founders, round, or other key fields, default to pulling them from the deck, investor memo, or company website before creating the Notion page. This supersedes any "skip attachments if email body has company name" shortcut.

**Why:** The email body is a cover note; the deck/memo/website is the source of truth. When a DocSend deck is attached, reading it is the default — creating a page with unnamed founders is wrong.

**How to apply:**
- Email with DocSend link → convert to PDF (docsend-to-pdf skill) → read → extract founder names, round, HQ, website.
- Email with no deck but company website → open site via Chrome osascript → check /team, /about, footer → extract founders.
- If both sources yield nothing: set fields to `TBD` / `N/A`. **Never invent founder names, roles, or backgrounds.** Hallucinated founder data is worse than blank.
- Skip this enrichment only when the email body itself already names founders with role/background detail.

### Ship every Opportunities row enriched — not as a stub

Every new entry in the Opportunities DB must ship enriched, not as a stub:

- **Icon** — always set. Pick a thematic emoji from Notion's standard emoji set. Never create with a blank/default icon.
- **HQ** — source material first (email sig, forwarded thread), then LinkedIn via WebFetch, then company website footer. ContactOut `company.headquarter` is the *fallback* — only hit it if free sources don't answer, since it burns a credit. YC company page is case-by-case (only when clearly YC-backed per LI headline). Map to existing `HQ` select option; default `??? 🌀` only if genuinely unknown.
- **Contact** — founder email. Source material first (signatures, forwarded threads, deck last slide), then public LinkedIn, then company website contact page. ContactOut (`profile_only=false` for `email`/`personal_email`) is the fallback.
- **Website** — email body links, founder email domain, or LinkedIn current-company block. Not `N/A` unless the company truly has none.
- **Description** — one-line what-they-do: YC one-liner, company site hero/meta, LI headline, or deck. Not `TBD`.
- **🏁 Founder(s)** — if the founder already exists in People DB (dedupe via `workspace_search` on "{first} {last}"), link the relation. If they DON'T exist, **leave the relation blank** — do not auto-create a People row. Note the gap in the response so Tom can decide whether to add them. The page body Team section is where founder names + LinkedIn URLs live; there is no separate scalar property for first names anymore.

**Why:** When a tip arrives with a founder LinkedIn URL, Tom expects the full enrichment cascade (LinkedIn URL → ContactOut → People DB dedupe → Opportunities fields) — not "name + source reference" stubs that create downstream work.

**How to apply:**
1. Whenever a founder LinkedIn URL exists in the source, run the enrichment cascade *before* writing the page (or immediately after, same turn): **source material → WebFetch on LI → YC page (if YC) → company website → ContactOut as fallback**. Don't burn ContactOut credits when the answer is free to find on LI or an email signature.
2. Always set `icon` in the `notion-create-pages` call or in an immediate follow-up `update-page`. Use a thematic emoji from Notion's standard emoji set.
3. If the LinkedIn URL isn't in the source, try a quick WebSearch for `{founder name} {company} founder` and grab the LI URL from the first LinkedIn result before giving up.
4. Treat `??? 🌀`, `TBD`, `N/A`, and blank Contact as failures to investigate — not acceptable end-states — unless the source genuinely offers no way to resolve them.
5. Applies equally to scheduled pipeline-agent runs — the unattended-execution guard is NOT an excuse to skip enrichment when the LinkedIn URL is in the tip.
