---
name: blurb-draft-sync
description: >
  Propagate a freshly updated company blurb (Company Overview) into every ACTIVE
  Gmail draft that carries a blurb dependency for that company — typically the
  `--` / *About [Company]* section of intro-outreach, intro-offer, and deal-share
  drafts. (C) Manual / subroutine only — chained automatically by
  log-company-blurb Step 6 after any blurb version bump, or invoked directly:
  "sync the new blurb to drafts", "update the [company] drafts with the new
  blurb", "refresh blurbs in open drafts", "propagate the blurb". Recreates
  signature-bearing drafts via gmail-create-draft.py (never update_draft),
  trashes the superseded versions, verifies the surviving set, and sends ONE
  consolidated #claude-alerts ping describing exactly what changed. No scheduled
  sweep, no webhook (Modes A/B intentionally absent).
---

# Blurb → Draft Sync

When a company's canonical blurb changes (the 📚 Company Overview callout on its Opportunity page), any open email draft that embeds that blurb is now stale. This skill finds those drafts, swaps in the new blurb, and tells Tom what changed.

## When this runs

- **Chained (primary):** `log-company-blurb` invokes this as its final step whenever it writes a NEW blurb version onto an Opp that previously had one (Case (a) demotion or Case (b) migration with priors). A first-ever blurb (Case (c)) has no drafts depending on it yet — the chain is skipped there.
- **Manual:** Tom asks directly ("update the drafts to Byron and the Peters with the new blurb").

Inputs (from the caller or resolved fresh): company name, Opp page ID, and the new blurb text. If the blurb text isn't passed, fetch the Opp and read the current 📚 callout body — that callout is canonical (see `log-company-blurb/SKILL.md`); never source the blurb from anywhere else.

## Step 1: Find candidate drafts

`list_drafts` with `query: "<company name>"`, `view: DRAFT_VIEW_FULL`. A draft is **in scope** when BOTH hold:

1. It is a live draft (label `DRAFT` — anything else is out of scope; never touch sent mail).
2. It has a **blurb dependency**: an `*About [Company]*` / `<em>About [Company]</em>` section (the standard block under the `--` separator that intro/deal drafters paste per `log-company-blurb`'s "About [Company]" convention), OR a paragraph that is recognizably the PRIOR blurb (≥1 full sentence verbatim or third-personized).

A draft that merely mentions the company in passing (e.g. a scheduling note) has no dependency — leave it alone. Zero in-scope drafts → exit silently; no alert, just report the no-op in the reply if running interactively.

## Step 2: Rewrite the blurb section — and nothing else

For each in-scope draft:

- Replace ONLY the blurb-dependent paragraph(s) with the new blurb. Every other byte of the draft — greeting, pitch paragraphs, closer, signature markup, recipients, subject — is preserved exactly.
- **Match the draft's existing conventions**, not the blurb's: if the draft's About section speaks in third person, third-personize the new blurb ("Our AI" → "Their AI", "We start" → "They start", "Our team" → "The team"); keep the company-name hyperlink and bold treatment the draft already uses; mirror its entity style (`&ndash;`/`&rsquo;` vs literal characters). Person/format adapt; content NEVER rewords.
- **Draft-specific lines survive** (founder bio line, deck link, ask-specific context). Exception: if a surviving line now duplicates a claim the new blurb carries (e.g. both say "finalizing a contract with one of the largest EPCs"), trim the duplicated sentence from the draft-specific line — and flag any detail lost in the trim (a dollar figure, a date) in the Step 4 alert so Tom can re-add it.

## Step 3: Write via recreate-and-supersede — NEVER update_draft

All Tom's outreach drafts carry his Apple Mail signature, and the Gmail connector's `update_draft` flattens it (see `shared-references/gmail-signature.md` and memory `gmail-connector-create-only-no-modify`). The only safe write path:

1. Render the revised HTML body to a temp file (signature block byte-identical from the fetched draft) + a plaintext snapshot file (tags stripped, signature block excluded).
2. Create the replacement via the atomic helper — route `--skill` with `python3 ~/.claude/scripts/email_router.py` (these are usually `intro-outreach`); pass `--no-alert` (Step 4 sends the one consolidated ping):
   ```bash
   python3 ~/.claude/scripts/gmail-create-draft.py \
     --to <recipients, comma-separated> --subject "<same subject>" \
     --html-body-file /tmp/<slug>_body.html \
     --snapshot-text-file /tmp/<slug>_snapshot.txt \
     --skill <routed-label> --no-alert
   ```
   Style-gate warnings on Tom-approved copy are advisory — do not "fix" his settled wording to satisfy the linter.
3. `trash_message` the superseded draft's messageId — standing permission per memory `feedback_supersede_draft_autodelete` (scoped carve-out: only the draft this run just replaced).
4. **Verify:** re-run `list_drafts` for the company and confirm the surviving set is exactly the replacements. A vanished or lingering draft is a failure to report, not to assume away.

Reply drafts (a draft attached to an existing thread) can't be recreated by the helper without losing threading — for those, note the stale draft in the alert instead of touching it, and let Tom decide.

## Step 4: One consolidated Slack alert

Send ONE ping via the `send-alert` skill (GFM on stdin to `send.sh`), regardless of how many drafts were touched:

```
🛠️ <u>**Blurb Sync: [Company]**</u>

[Company]'s Company Overview was refreshed ([date/source]), so [N] open draft(s) got their *About [Company]* section swapped:

- **[Recipient(s)]** ([draft](https://mail.google.com/mail/u/0/#drafts/<messageId>)) — [what changed beyond the swap: trims, lost details, or "blurb replaced; nothing else touched"]

Superseded drafts trashed and verified.
```

Flag every editorial judgment call (trimmed duplication, dropped figure, untouched reply-draft) — Tom audits these alerts.

## Notes

- Model tier: Sonnet (rubric applied + review gate — drafts never send themselves).
- This skill never sends email, never edits sent mail, never touches drafts for other companies, and never rewords blurb content — person and formatting adaptation only.
- If `log-company-blurb` ran but the blurb text is identical to the prior version (no-op version bump), skip everything.
