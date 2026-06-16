---
name: neg1-promote
description: >-
  Promote an existing `-1 (FounderName)` Opportunity to a named company when the founder reveals the real
  company name (post-meeting email, Slack reply to a "no matching Opp" alert, or chat instruction). Renames the
  Opp, merges new content (cofounders, deck, additional emails, post-meeting context) into the existing row, and
  dedups emails into the Contact alias list. Does NOT create a new Opp — this is the merge path when add-to-crm
  dedup hits an existing -1 row or Tom directs the rename. Idempotent. Trigger phrases: "this is -1
  [FounderName], rename to [Company]", "promote -1 [FounderName] to [Company]", "rename -1 [FounderName] to
  [Company]", "roll [thread/intro/email] into [FounderName]'s -1 opp and rename to [Company]", or any variant
  naming an existing -1 row + a company to promote it to. Modes: B (Listener, via claude-alerts-listener on a
  "no matching Opp" reply) and C (Manual). Never creates a People DB row for a new cofounder — surfaces them to
  Tom.
---

# -1 Promote

Take an existing `-1 (FounderName)` Opportunity and rename it to the actual company name once revealed, merging any new content from the triggering source into the existing row.

## Notion Targets

- **Opportunities Database** — Data Source ID: `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`

## Inputs

- `founder_name` (required) — the founder behind the existing `-1` row (e.g. `Vikas Sankhla`)
- `new_company_name` (required) — the actual company name (e.g. `Aerion`)
- `source_thread_id` (optional) — Gmail threadId to pull cofounders / deck / additional emails / post-meeting context from
- `source_alert_ts` (optional, Mode B only) — Slack thread parent ts in `#claude-alerts` for close-loop reply
- `channel_id` (optional, Mode B only) — defaults to `C0B06385BP1` (#claude-alerts)

## Workflow

### Step 1: Resolve the -1 Opp

Search the Opportunities DB for a row whose Name matches `-1 ([FounderName])`:

- Tool: `mcp__claude_ai_Notion__notion-search`
- `data_source_url`: `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
- `query`: the founder name

Filter results by title. The expected match is a page titled `-1 (FounderName)` or `-1 ([FounderName](linkedin-url))` (Notion-flavored markdown link form).

**Failure modes:**
- **Zero matches** — surface to Tom: "No `-1 (FounderName)` row found in Opportunities. Want me to run `add-to-crm` instead?" Exit without writes.
- **Multiple matches** — surface the candidate URLs and ask Tom which one. Don't guess.
- **Match but Status is in `{Pass (Met), Pass (DNM), Lost, NR / Missed, Active Portfolio, Portfolio: Follow-On, Exited, Committed}`** — refuse. The Opp is in a terminal/protected state; renaming would corrupt history. Surface the existing page and the conflict to Tom.

### Step 2: Fetch the page

Fetch the matched page with `mcp__claude_ai_Notion__notion-fetch`. Read:
- Current `Name`
- Current `Contact` (`; `-separated alias list, per memory `feedback_upgrade_opp_contact_to_custom_domain_asap.md`)
- Current `Diligence Materials` property
- Full body content (you'll edit it in Step 5)

### Step 3 (optional): Extract from source thread

If `source_thread_id` is provided, fetch the full thread with `mcp__claude_ai_Gmail__get_thread` (`messageFormat: FULL_CONTENT`).

Scan the latest message(s) from the founder for:

- **Cofounders / team additions** — body phrases like "I have also included [Name], our [role], on this thread" or "[Name] is joining as [role]". Capture name + email (from CC headers) + role. Do NOT create People DB rows for them (per `feedback_no_people_entry_without_permission.md`); they live in the Opp body until Tom explicitly authorizes a People entry.
- **Additional founder/team emails** — any new addresses on the From/To/CC lines that aren't already in the Opp's Contact field. Apply Gmail dot-normalization (`vikassankhla@gmail.com` ≡ `vikas.sankhla@gmail.com`) when deduping.
- **Deck / materials URLs** — Drive links, DocSend links, attachment references. Cross-check against the `Diligence Materials` property values; if a URL from the thread isn't there, flag for Step 6.
- **Post-meeting context** — meeting confirmations, working name reveals, cadence statements ("I'll reach out as updates become significant"), in-person meeting locations.

### Step 4: Update Name + Contact

Single `update_properties` call:

```json
{
  "Name": "<new_company_name>",
  "Contact": "<existing alias list; new emails appended, ; -separated, deduped on Gmail-normalized form>"
}
```

If Contact is already up to date (no new emails after dedup), omit it from the update.

### Step 5: Update body content

Use `update_content` (search-and-replace) with these passes — each pass is idempotent (skip if the new string is already present):

1. **Summary** — if the existing Summary opens with `Pre-company.` or similar pre-reveal phrasing and does NOT already contain the new company name, prepend `Pre-company, working name **<new_company_name>**.` to the existing Summary. If the company is no longer pre-company (e.g. the email signals a founded entity), use `Working name **<new_company_name>**.` Skip if the company name already appears in the Summary.
2. **Team** — if cofounders were extracted in Step 3, append a one-line entry per cofounder to the existing Team section:
   `<Name> – <role>. Email \`<email>\`. Added to thread by <founder_name> on <YYYY-MM-DD>; LinkedIn/background not yet enriched.`
   Skip cofounders whose name already appears in the Team section.
3. **Post-Reveal section** — if `source_thread_id` is provided, append (don't replace) a new section at the end of the body:
   ```
   **Post-Reveal Update (YYYY-MM-DD)**
   <one-paragraph summary of what the source thread/email revealed: working name, cofounders looped in, deck attached, cadence statement, etc.> ([Gmail thread](https://mail.google.com/mail/u/0/#all/<source_thread_id>))
   ```
   If a `**Post-Reveal Update (YYYY-MM-DD)**` section with today's date already exists in the body, skip — don't double-write.

### Step 6: Verify Diligence Materials

For each deck/material URL extracted from the source thread:

- If already present in the Opp's `Diligence Materials` property, skip.
- If missing, surface to Tom: "Deck URL `<url>` from the source thread isn't in the Diligence Materials property — re-invoke `materials-handler`?" Do NOT silently invoke materials-handler from this skill; that's the user's call (or the upstream webhook's job).

Per memory `feedback_deck_url_in_diligence_materials_field.md`: deck URLs belong in the property field, not just body links.

### Step 7: Close-loop (Mode B only)

If `source_alert_ts` was passed (Mode B / listener invocation), post a close-loop reply in the originating Slack thread. Use the listener's helper:

```bash
/Users/tomseo/.claude/skills/claude-alerts-listener/post_close_loop.sh \
  "<channel_id (default C0B06385BP1)>" \
  "<source_alert_ts>" \
  "✅ Rolled into existing `-1 (<founder_name>)` Opp and renamed to **<new_company_name>**. <bullet of what was merged: cofounders / deck / Contact emails / post-reveal section>. <link to updated Opp>"
```

In Mode C (manual / interactive), just summarize the changes back to Tom in chat — no Slack post.

## Idempotency contract

Re-running this skill with the same inputs must be a no-op:

- Name property write: if current Name already equals `new_company_name`, skip.
- Contact write: dedup on Gmail-normalized form before writing; if no new emails, skip.
- Summary edit: skip if `<new_company_name>` already appears in the Summary.
- Team additions: skip cofounders already present by name.
- Post-Reveal section: skip if a section with today's date already exists.

This skill is safe to chain after `add-to-crm`'s `duplicate-in-pipeline-skip` exit when the dup is a `-1` row, and safe to re-invoke from the listener after webhook retries.

## What this skill does NOT do

- Does NOT create People DB rows for cofounders. Per memory `feedback_no_people_entry_without_permission.md`, surface them in the Opp body and flag to Tom; new People entries require explicit permission + ContactOut enrichment + photo.
- Does NOT change the Opp's `Status`. The status transition (Scheduled → Active, Connected → Active, etc.) is handled by the pipeline state machine via webhook handlers — don't double-write here.
- Does NOT invoke `materials-handler`. If a deck URL is missing from the property, surface to Tom (Mode C) or flag in the close-loop (Mode B); the upstream webhook handles material attachment.
- Does NOT re-classify the Opp into the Companies DB. That's `add-to-companies`'s job, invoked separately if Tom wants the company tracked.
- Does NOT promote `-1` rows whose Status is terminal or protected (see Step 1 failure modes).

## Why this exists

The `-1` shorthand creates Opps before the founder has named the company. When the founder later reveals the name (typically post-first-meeting via deck send), the existing row needs renaming — not a duplicate. The intro-agent's "Three-way intro / no matching Opp" alert and `add-to-crm`'s dedup guard both surface the case but neither handles the merge. This skill closes that gap.

Originating incident: 2026-05-12 — Vikas Sankhla (`-1`) emailed Tom post-coffee with the Aerion deck and looped in Chinmay Prabhakar as technical lead. Intro-agent fired a "no matching Opp / Company: Aerion" alert because its three-way intro pre-checks didn't catch the dot-normalized Gmail address match. Tom replied "This is -1 Vikas Sankhla. Roll everything into that opp and rename to Aerion." This skill is the formalization of that workflow.
