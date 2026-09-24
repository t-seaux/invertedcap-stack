---
name: add-missed-to-crm
description: Add a company to the Notion CRM pipeline as a MISSED deal — full add-to-crm enrichment, but Status is hard-set to `NR / Missed`. Trigger when Tom says "add to crm as missed", "add as missed", "log as missed", "missed this one — add it", "crm this as missed", "log this missed deal", "mark as missed and add to crm", or shares an announcement (funding news, launch post, "my next adventure"-style article) for a deal he never got into and wants recorded as missed. Source material takes the same forms as add-to-crm (article/announcement URL, screenshot, forwarded email, LinkedIn URL, pasted text). Manual-only — no webhook or scheduled entry point.
---

# Add Missed to CRM

Log a deal Tom missed — saw too late, never got access to, or watched close without him — as a fully-enriched Opportunities row with Status `NR / Missed`. The row exists for the record (anti-portfolio, decision retros, dedup against future re-surfacing), not as live pipeline.

Single source of truth: this skill does NOT inline add-to-crm logic. Read `add-to-crm/SKILL.md` (and its `references/schema.md`) and execute the full manual-mode workflow — every step, every enrichment cascade, every field rule — with only the deltas below.

## Deltas vs. add-to-crm

1. **Status is a directive, not an inference.** Set `statusDirective: "NR / Missed"` and honor it verbatim — skip Step 5's Status inference entirely. Never let thread state, announcement content, or anything else flip it. `Close Date` still follows the canonical rule (leave blank; it anchors scheduled calls, which a missed deal never has).

2. **Pipeline-entry gate does not apply.** The gate routes unqualified machine rosters away from the CRM; this skill is manual-only and the invocation itself IS Tom's judgment that the row belongs. A terminal-status row also doesn't tax the live board the way an active row does. Skip the gate; go straight to the Protected Status Guard.

3. **Protected Status Guard still runs in full — it is MANDATORY.** Delta is only in the decision table:
   - Candidate in a **protected terminal status** (Active Portfolio / Portfolio: Follow-On / Exited / Committed) → do not create; surface the existing page. Tom didn't miss it.
   - Candidate with a **prior pass** (Pass (Met) / Pass (DNM) / Lost / NR / Missed) → do not create; surface the existing page. It's already terminally logged.
   - Candidate in an **in-progress status** (Qualified / Outreach / Connected / Scheduled / Active / Track / etc.) → do not create a duplicate. Ask Tom whether to flip the EXISTING row to `NR / Missed` instead — "add as missed" on a company already in the pipeline usually means "close out that row," not "make a second one." Flip only on his confirmation.

4. **Source material is usually an announcement, not a pitch.** Funding news, a launch post, a founder's "next adventure" article. Everything add-to-crm says still applies: full enrichment cascade (HQ / Contact / Website / Description — no stubs), founder LinkedIn resolution, People-DB relations, thematic emoji icon, body section named by source type (`**Original Post**`, `**Original Email**`, etc.) with the source verbatim, URL-fidelity rule intact.
   - `Round Details`: the round Tom missed, formatted per `shared-references/round-details-format.md` when terms are disclosed. Because the round has typically already closed, the no-valuation variant reads `Raised $3m` instead of the spec's `Raising` form — the only format deviation this skill permits. No disclosed terms → blank, per canon.
   - Add one line of missed-context at the top of the page body, above the Original-* section, stating how/when it was missed if Tom said so (e.g. `⏱️ Missed — announced 2026-09-12; round closed before contact.`). Keep it to one line; don't editorialize.

5. **Source(s)**: resolve normally. Tom finding the announcement himself = "Direct". If a specific person surfaced it too late, they're still the Source — how Tom heard about it doesn't change because the answer was no-window.

6. **No Slack alert** (Step 8 is webhook-only; this skill is manual-only). Materials handling (Steps 6–7) runs normally if a deck/memo is present — rare for missed deals, but not skipped.

## Known interaction — deal-share-out (intentional)

The notion-webhook's auto deal-share (deal-share-out Mode B2) fires on non-(-1)/non-FO Opps in a terminal pass status — `Pass (Met)` / `Pass (DNM)` / `NR / Missed` / `Lost` — including rows *created* with the status already set, which is exactly how this skill creates them (worker `page.created` path, added 2026-09-15 per Tom). So every row this skill mints should shortly produce a deal-share Gmail draft. That is intended behavior, not a stray — do not suppress it.
