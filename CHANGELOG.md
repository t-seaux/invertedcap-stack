# Changelog

All notable changes to the Inverted Cap Stack platform are documented here.

## [2026-04-23] (Week of 2026-04-20) — Event-Driven Reclassification + Microstep Consolidation

**Added:** None
**Removed:** None
**Modified:**
- add-to-companies — canonical enrichment spec extracted to a shared reference; behavior now reads from `shared-references/companies-enrichment-spec.md` so every caller uses the same field list and source priority.
- add-to-contacts — added a workspace-search dedup rule (AI search was silently missing exact title matches) and a no-auto-create rule blocking side-effect workflows from polluting the People DB.
- add-to-crm — now enriches HQ, contact, website, logo, and description from founder LinkedIn before page creation, with a cost-aware source priority (source → public LinkedIn → company website → paid API fallback).
- founder-outreach — reads two voice corpora (accumulated edit patterns and canonical from-scratch examples) before drafting and writes a draft snapshot to Drive at creation time for the headless feedback loop.
- investor-update — mandatory attachment probe before case-routing; Gmail plaintext hides attachments behind a placeholder glyph, so the attachment saver must run unconditionally.
- materials-handler — mirrors the mandatory attachment probe rule and promotes the Chrome-automation path for writing external URLs to Files properties to primary (the public API rejects external URL writes).
- neg1-enricher — delegates company + school resolution to add-to-companies as a subroutine, removing duplicated enrichment logic and inheriting dedup, backfill, and momentum from the canonical spec.
- pass-note-drafter — same dual voice-corpus read step before drafting and writes a Drive snapshot for every draft created.
- pipeline-agent — added scheduler-bot fallback (Blockit, Calendly, Google Calendar, Sidekick, SavvyCal, etc.) so meeting confirmations from bots resolve back to the founder via subject line and quoted thread; explicit rule that Close Date only fires on invest/pass terminal states.
- pre-mortem — added a rendering rule pointing long-form artifacts at the canonical PDF spec (reportlab + Helvetica + black-and-white) rather than the PNG-first design-language guide.

**Total skills:** 23
**Functions:** Intro Management consolidates to 1 skill (was 5) — `intro-outreach-agent`, `intro-resolution-agent`, `intro-draft-agent`, and `log-intro` now absorb into `intro-agent` as microsteps of one end-to-end value chain. Diligence Management consolidates to 8 skills (was 9) — `feedback-outreach-drafter` and `feedback-outreach-scanner` absorb into `feedback-outreach` as microsteps of one "get feedback on an Opp" value chain.

Notes: Introduces an **Event-Driven** third badge alongside Scheduled and Ad Hoc. Top-of-funnel detection and end-to-end lifecycle chains that are now primarily driven by Gmail webhook handlers (rather than cron sweeps) are reclassified: `pipeline-agent`, `intro-agent`, `investor-update`, `feedback-outreach`, and `draft-feedback` move to Event. `pass-note-drafter` moves from Scheduled to Ad Hoc — Tom initiates, the webhook is passive completion detection. The Stack Map legend gains a third swatch. Residual scheduled sweeps remain as fallbacks while webhook coverage is validated.

---

## [2026-04-22] (Week of 2026-04-20) — Headless Feedback Engine + Decision Retros

**Added:** draft-feedback (Pipeline Management, Scheduled), decision-retro (Diligence Management, Ad Hoc)
**Removed:** None
**Modified:**
- founder-outreach — now reads two voice corpora (accumulated edits from prior sends + canonical from-scratch examples) before drafting; writes a draft snapshot to a synced Drive folder at draft creation time, replacing the prior in-database draft writeback.
- pass-note-drafter — same dual voice-corpus read step before drafting; writes a Drive snapshot for every draft created.

**Total skills:** 28
**Functions:** Pipeline Management now lists 7 skills (was 6). Diligence Management now lists 9 skills (was 8). No new functions.

Notes: First-class headless feedback loop. Outreach and pass-note drafts are now snapshotted at creation time to a synced Drive folder; a Gmail webhook detects sends, matches snapshots by persistent message ID, and a local processor extracts voice patterns into skill-local files that the drafters re-read on every run. The loop is invisible — no manual triggers, no database fields, no inbox clutter. From-scratch sends matching the canonical outreach or pass-note subject lines are also captured as canonical voice examples. Decision-retro captures free-form post-decision reasoning on every invest, pass, or cold dismissal and routes thematic nuggets into the founder signal framework corpus that drives ongoing rubric evolution.

---

## [2026-04-22] (Week of 2026-04-20)

**Added:** None
**Removed:** None
**Modified:**
- neg1-enricher — absorbed the signal scoring + verdict rubric; now produces a fully-evaluated -1 Scanner row (enrichment + score + recommendation) and adds a `--score-only` mode for rubric drift checks.
- founder-outreach — narrowed to a single-mode drafting primitive; scoring moved upstream to neg1-enricher. Refuses to draft on unscored rows and now captures the initial AI-authored body for future diff-based learning.
- first-pass-diligence — expanded the Path to Next Round section from four to six subsections, splitting Seed vs Series A comps and adding an Adjacent Space Comps view of the competitive capital landscape.

**Total skills:** 26
**Functions:** No changes

Notes: The -1 pipeline refactor consolidates enrichment + evaluation in one skill so Tom sees a verdict the moment a row is enriched; drafting is now purely the last mile. First-pass diligence gets sharper stage benchmarking.

---

## [2026-04-21] (Week of 2026-04-20) — Manual Update

**Added:** add-to-companies
**Removed:** None
**Modified:** None
**Total skills:** 26
**Functions:** Research Management now lists 5 skills (was 4). add-to-companies is the standalone entry point for building the companies knowledge base — enriches a company from LinkedIn URL, domain, or name via ContactOut and related sources.

Notes: Also changed the refresh cadence from weekly to daily (runs every day at 7:30 AM ET) so the map stays in sync with active platform evolution.

---

## [2026-04-20] (Week of 2026-04-20) — Manual Reclassification

**Added:** None
**Removed:** docsend-to-pdf
**Modified:** None
**Total skills:** 25
**Functions:** Pipeline Management now lists 6 skills (was 7). docsend-to-pdf reclassified as a utility subroutine — invoked by other skills (add-to-crm, materials-handler) rather than standing on its own as a user-facing workflow.

Notes: Second same-day manual pass. docsend-to-pdf is mechanical utility glue, not a pipeline primitive, so it no longer appears on the public platform map.

---

## [2026-04-20] (Week of 2026-04-20) — Manual Correction

**Added:** neg1-enricher, founder-outreach
**Removed:** None
**Modified:** None
**Total skills:** 26
**Functions:** Pipeline Management now lists 7 skills (was 5). Two pipeline primitives surfaced on the map after correcting a stale canonical mapping.

Notes: Corrects omission from Scheduled Run #3 earlier today — the canonical function mapping in the refresh skill had drifted from the live skills directory, so two Pipeline Management skills were present on disk but not rendered on the visuals. Canonical mapping updated in tandem with this push.

---

## [2026-04-20] (Week of 2026-04-20) — Scheduled Run #3

**Added:** None
**Removed:** neg1-scanner
**Modified:** None
**Total skills:** 24
**Functions:** No changes to function membership. Pipeline Management now lists 5 skills (was 6).

Notes: First run with the `skill-snapshots/` diff baseline — snapshots seeded this run; behavior changes in individual skills will be detected going forward.

---

## [2026-04-06] (Week of 2026-04-06) — Scheduled Run #2

**Added:** None
**Removed:** None
**Modified:** first-pass-diligence — reclassified from Ad Hoc to Scheduled (first-pass-diligence-scan scheduled task detected via `-scan` suffix convention)
**Total skills:** 25
**Functions:** No changes to function membership

---

## [2026-04-06] - Weekly Skill Map Refresh (Manual Trigger)

### Added
- **coinvestor-recommender** (Ad Hoc) - Recommends qualified coinvestors from Tom's network for a specific deal. Integrated into Portfolio Management function.

### Removed
- **backchannel-drafter** - Consolidated into feedback-outreach-drafter (identical implementation)
- **feedback-outreach-drafter-ad-hoc** - Consolidated into feedback-outreach-drafter (identical implementation)
- **feedback-outreach-drafter-manual** - Consolidated into feedback-outreach-drafter (identical implementation)
- **feedback-outreach-drafter (ad hoc)** - Removed from visual; single feedback-outreach-drafter now handles both scheduled and manual modes

### Modified
- **feedback-outreach-drafter** - Added "backchannel" trigger phrases to description; now serves as the single canonical skill for both scheduled scans and manual drafting. Fixed relation field references to use actual Notion field name (`📣 Pending Feedback`).
- **Platform Map** - Updated visual: 5 functions, coinvestor-recommender added to Portfolio, duplicate feedback skills removed from Diligence
- **Quick Reference** - Regenerated skill inventory reflecting consolidation

### Statistics
- **Total Skills**: 25
- **Scheduled Tasks**: 12
- **Ad Hoc Skills**: 13
- **Functions**: 5
- **Excluded duplicates**: 3 (feedback-outreach-drafter-manual, feedback-outreach-drafter-ad-hoc, backchannel-drafter)

---

## Future Release Notes
(Previous releases will be documented as they occur)
