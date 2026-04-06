# Changelog

All notable changes to the Inverted Cap Stack platform are documented here.

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

### Classified (not on visual)
- **Fund Ops** (hidden): mmf-to-lp-calc, cpa-report
- **Other** (hidden): note-classifier
- **Excluded duplicates**: feedback-outreach-drafter-manual, feedback-outreach-drafter-ad-hoc, backchannel-drafter

### Statistics
- **Total Skills (visible)**: 25 (was 25 at baseline; +1 coinvestor-recommender, -2 duplicate feedback skills, +1 backchannel-drafter added then consolidated)
- **Total Skills (all, incl. hidden)**: 28 (excluding 3 duplicates)
- **Scheduled Tasks**: 12
- **Ad Hoc Skills (visible)**: 13
- **Functions (visible)**: 5 (no change)
- **Functions (all)**: 7 (added Fund Ops, Other as hidden categories)
- **Excluded duplicates**: 3 (feedback-outreach-drafter-manual, feedback-outreach-drafter-ad-hoc, backchannel-drafter)

---

## Future Release Notes
(Previous releases will be documented as they occur)
