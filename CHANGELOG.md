# Changelog

All notable changes to the Inverted Cap Stack platform are documented here.

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
