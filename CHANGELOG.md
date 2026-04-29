# Changelog

All notable changes to the Inverted Cap Stack platform are documented here.

## [2026-04-29] (Week of 2026-04-27)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-crm` -- restructured the Status defaulting rules to distinguish source types: `Qualified` only for surfaced (forwarded/pasted) deals, `Connected` for cold inbound founder emails (deal-scanner webhook path) where Tom is in a live thread from message #1, and `Outreach`/`Connected`/`Scheduled` per existing rules for non-cold-inbound paths. Expanded the deck-URL passthrough to include third-party deck-sharing platforms (brieflink.com, pitch.com, etc.) — all extracted URLs now flow to materials-handler, which decides convert-vs-link-as-is. Property-field linking is now mandatory for ALL deck/material URLs, not just converted PDFs.
- `founder-outreach` -- replaced the delete-then-create draft pattern with a delete-old-after-Notion-points-to-new sequencing: the new draft is created first, Notion's `Gmail Draft URL` is repointed to the new hex, and only then is the prior draft deleted. The webhook smartening shipped in gmail-webhook v73+ uses Notion's URL match to recognize orphan-cleanup deletes and ignore them, preventing spurious pass detection during workshop iteration. Belt-and-suspenders Status re-assertion added on every invocation.
- `materials-handler` -- promoted the headless Python helper (`add_link_to_files_property.py`) to the canonical path for Notion Files-property writes; it shells out to Notion's internal `saveTransactions` endpoint with a token cookie, runs cleanly in scheduled / unattended / job-queue contexts, and no longer requires Chrome. The osascript+Chrome path is now a fallback used only when the Python helper hits TOKEN_EXPIRED. Added Brieflink to the link-only / non-convertible platform list (alongside Figma, Miro, Loom, Pitch.com, Canva, Notion.site).
- `neg1-enricher` -- restructured the Earned Reps signal to read the highest-fidelity hypergrowth source available, in priority order: (1) the Current Company Momentum rollup on the -1 Scanner row, which pulls Deal Digest revenue traction directly from the primary employer's Companies row; (2) Sales Nav Headcount + HC Commentary; (3) funding-round cadence (Hypergrowth Windows). When tenure overlaps best-in-class peer-tier ramp, score 9/10. Added a sector-difficulty multiplier for tenure earned in hard buyer environments (health systems, public sector, defense, regulated finance).
**Total skills:** 23
**Functions:** No changes

---

## [2026-04-28] (Week of 2026-04-27)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-companies` -- added a Mode B webhook entry for existing Companies rows whose URL property has just been populated; the webhook skips dedup search and runs straight into enrichment. Standardized the Slack alert template (header line + bullets covering category, HQ, funding status, revenue, headcount trend, and digest mentions).
- `add-to-contacts` -- added a Mode B webhook entry for existing People rows whose LinkedIn URL has just been populated; the webhook fetches the row, runs ContactOut enrichment, and updates in place without re-running dedup or creating a new row.
- `decision-retro` -- tightened the Slack retro prompt format from bulleted entity/status lines to bare paragraphs (no `- ` prefix); fingerprint sits immediately under the status line. Compact format only — no inline Description / Founders / Website fetch.
- `draft-feedback` -- after each pass-note send is fully processed, posts a single consolidated Slack DM confirming all three downstream outcomes (Status flipped to Pass (Met), Notes archive entry created, diff or from-scratch voice captured) by reading the webhook's results sidecar.
- `first-pass-diligence` -- added a mandatory grounding guardrail requiring portfolio references in generated memos to come from a live Opportunities DB query (Active Portfolio / Exited / Committed only); flagged historical miscategorizations of analyzed-but-not-portfolio companies. Added a rule to always upload a NEW Drive file rather than overwrite (Notion caches PDF previews on file ID, so overwrites silently shipped stale previews). Switched Diligence Materials property linking to a Python helper as the canonical path. Tightened markdown escape handling for HTML table syntax.
- `investor-update` -- added a forwarded-email normalization step that strips the forward chrome (wrapper line, metadata block, blank lines, zero-width spaces, object-replacement glyphs, Apple Mail quote-prefix `>` characters) before the body is fed into either the PDF render or the Notion page; supports multi-hop forwards. Applies uniformly to sweep, webhook, and manual modes.
- `materials-handler` -- added a Mode B webhook entry that fires when a contact on a pipeline opportunity sends an email with materials. Idempotent via a `claude/materials-processed` Gmail label applied per-message after successful processing. Added a scoped Slack alert format separating Page Body / Diligence Materials / Deal Docs sections, each with click-through links.
- `pass-note-drafter` -- recipient list switched from a single primary founder email to the full `;`-split contact list (every meeting attendee captured by the calendar handler). Greeting auto-pluralizes for multi-recipient sends. Retired the legacy Zapier BCC archive path now that the pass-note-sent gmail-webhook handler does the archive natively. Reformatted Slack alert to consolidator style (entity header + drafted-for line). Moved draft snapshot directory under `_system/`.
- `skill-map-refresh` -- registered `outreach-detector` and `outreach-decliner` webhook handlers as rolling into `pipeline-agent` (both as event-mode sub-rows in the Quick Reference composite expansion + classification table).
**Total skills:** 23
**Functions:** No changes

---

## [2026-04-27] (Week of 2026-04-27)

**Added:** None
**Removed:** None
**Modified:**
- `deal-digest` -- restructured from a per-batch standalone page with synthesized benchmark summary into a single monthly rolling Notes page; every ingest (periodic batches, ad-hoc competitor intel, single traction snippets) now prepends as a dated, source-tagged block, with no cross-ingest merging or benchmark synthesis at log time.
- `decision-retro` -- locked in a compact entity-rendering format for Slack retro prompts (entity bullet + status bullet + fingerprint, with bold + underlined linked title); dropped the inline Description / Founders / Website / Work History / Education fetch in favor of a single click-through to the Notion page.
- `draft-feedback` -- relocated voice-pattern corpora (`EDIT_PATTERNS.md`, `VOICE_EXAMPLES.md`) from per-skill directories into a centralized `writing-style/<type>/` tree, so outreach and pass-note drafters share one canonical stylebook surface mapped via `processor.py`.
- `founder-outreach` -- migrated the cold-email template + voice corpora to the centralized `writing-style/outreach/` stylebook; reading order now pulls Canonical Principles (durable rules) and Recent Edits (append-only log) as separate sections of `EDIT_PATTERNS.md`.
- `intro-agent` -- tightened the auto-create-from-intro path to run the full `add-to-contacts` enrichment (LinkedIn URL lookup, ContactOut person enrichment, Expertise Summary, Working Description, page icon) so auto-created People rows match the depth of manual entries; closes a regression where stub-fielded rows were being created.
- `pass-note-drafter` -- added a webhook mode (Mode B) invoked per single Opp via the notion-webhook claude-job-queue when an Opp's Status flips to Pass Note Pending, with idempotency dedup against existing Gmail drafts and a status-guard re-check before drafting; sweep mode unchanged.
**Total skills:** 23
**Functions:** No changes

Notes: Two new directory entries -- `outreach-decliner` and `outreach-detector` -- showed up in the skills directory but are not yet mapped to a function in the canonical refresh spec. Both look like Gmail-webhook-driven Opportunity status flippers (Qualified -> Pass (DNM) and Qualified -> Outreach respectively); flagged for manual classification before they get rendered on the visuals.

---

## [2026-04-25] (Week of 2026-04-20)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-crm` -- dropped Latest Outreach property tracking from CRM page creation; the dedicated Latest Outreach field is no longer set during triage.
- `decision-retro` -- added a Lookup mode that answers past-tense questions about prior decisions (returns a structured summary: official pass-note feedback, internal nuggets by dimension, framework links) without echoing the raw retro; widened the trigger-status set to include `Pass (DNM)`; added a parallel `-1 Scanner` retro mirror so cold dismissals on -1 candidates get the same prompt-and-listen flow; added a weekly Slack rollup post grouped by scope; added skip / ignore reply handling.
- `investor-update` -- formalized the Slack alert delivery as a two-step Haiku composer plus `send-alert` pipe under the `claude` bot identity; removed the direct Slack-MCP send path that was conflating with messages Tom actually sent.
- `materials-handler` -- added a link-only path for interactive / hosted materials (Figma, Miro, Loom, Pitch.com, Canva, Notion.site) that can't be cleanly downloaded or PDF-rendered; relocated the canonical surface for material links from a page-body bullet section to the Notion Diligence Materials property-field chips, with the body section reserved for exception cases.
- `pipeline-agent` -- added a Haiku sub-agent step (escalating to Sonnet on flagged items) for composing the Slack consolidator alert from the structured per-stage summary; clarified that the webhook handler is the primary path for the Qualified → Outreach transition while the scheduled sweep is a reconciliation fallback; dropped Latest Outreach property tracking across triage stages.
**Total skills:** 23
**Functions:** No changes

---

## [2026-04-24] (Week of 2026-04-20)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-companies` -- tightened credit-saving rule: full Steps A–E enrichment pipeline always runs on direct invocation; only skip is when `Last Enriched` is already populated.
- `add-to-crm` -- dedup check is now mandatory for every caller (manual + unattended) with a 3-way lookup (title + website domain + contact email); added a formal decision table for protected / prior-pass / in-progress / safe-to-create paths.
- `decision-retro` -- rewired from synchronous Notion-based capture to a headless queue-based architecture: scan at 09:00 ET posts a prompt to a dedicated Slack channel, listener at 18:00 ET reads thread replies and logs the retro; state now lives in a local `queue.json` rather than on the Opportunity page.
- `feedback-outreach-scanner` -- added webhook mode for per-inbound-reply processing via the `claude-job-queue` primitive; sweep mode retained as reconciliation fallback.
- `founder-outreach` -- auto-invoked as the terminal step of `neg1-enricher` on manual -1 scan requests, regardless of the Claude Rec verdict, so Tom sees the draft alongside the score; batch enrichment via `pipeline-agent` Task 6 remains no-chain.
- `intro-agent` -- clarified the scheduled mode as a reconciliation sweep backing the per-event webhook handlers.
- `intro-resolution-agent` -- added webhook mode for per-reply classification via `claude-job-queue`; now three modes total (sweep, webhook, manual).
- `investor-update` -- added webhook mode for per-message processing via `claude-job-queue`; now three modes total (sweep, webhook, manual).
- `neg1-enricher` -- on manual invocation, auto-chains into `founder-outreach` regardless of Claude Rec verdict; batch mode via `pipeline-agent` stays no-chain; added a `--score-only` re-evaluation mode for rubric drift checks.
- `pass-note-drafter` -- now archives sent pass notes to the Notes DB as a Diligence-category note linked to the Opportunity; added a Step 2.5 backfill pass that catches cases where the webhook archive step failed while the status flip succeeded.
- `pipeline-agent` -- clarified the scheduled mode as a reconciliation sweep backing the per-event webhook handlers.
- `skill-map-refresh` -- collapsed the Scheduled/Event visual binary into a unified Autonomous category; Quick Reference rows now render a fused tri-cell mode pill (Sweep / Webhook / Manual); added `pipeline-agent` and `diligence-agent` to the composite list so their expansion drawers show the full webhook + canonical + manual universe.
**Total skills:** 23
**Functions:** No changes

---

## [2026-04-23] (Visual refresh)

**Added:** None
**Removed:** None
**Modified:** None
**Total skills:** 23
**Functions:** No changes

Notes: Event pill recolored from cool gray to warm cream (`#e0d6c3`) across the Stack Map legend, skill pills, and Quick Reference badges. Introduced composite treatment: skills that absorb multiple sub-skills into a single pill (currently `intro-agent`, `feedback-outreach`) render with a stacked-card shadow on the Stack Map and an expand-on-click interaction in the Quick Reference that reveals nested sub-skill rows with per-sub one-liners. Legend reordered to Ad Hoc / Scheduled / Event-Based / Composite, with a new Composite swatch entry. Quick Reference row layout normalized so badges and skill-text align at the same x-position regardless of whether a row has a chevron.

---

## [2026-04-22] (Visual refresh)

Event pill recolored to neutral gray; label reverted to Event.

---

## [2026-04-23] (Visual refresh)

**Added:** None
**Removed:** None
**Modified:** None
**Total skills:** 23
**Functions:** No changes

Notes: Event pill color lightened from teal to light mint (`#86EFAC`); badge label renamed from "Event" / "Event-Driven" to "Event-Based" across the Stack Map legend and Quick Reference badges. Connection-line and People database border colors unchanged.

---

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
