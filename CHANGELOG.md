# Changelog

## [2026-07-22] (Week of 2026-07-21)

**Added:** talent-scan (Portfolio Management), question-bank (Diligence Management)
**Removed:** None
**Modified:** None
**Total skills:** 39
**Functions:** Portfolio Management +1 (talent-scan), Diligence Management +1 (question-bank)

---

## [2026-07-21] (Week of 2026-07-21)

**Added:** neg1-sourcing-listener (Pipeline Management — categorized from pending since 2026-07-17), founder-taste (Diligence Management — categorized from pending since 2026-07-17)
**Removed:** None
**Modified:**
- neg1-enricher -- applies prefilter gate before scoring; now posts individual Slack cards for all verdicts (Reach Out / Second Look / Pass), not just Reach Out; 🧠 callout icon and restructured card format with fixed bullet order
- neg1-sourcing -- added prefilter screen (Step 1.6); all-verdict card posting; new -1 Engine report section in daily alert; re-surface sweep migrated from Notion to local candidate store
- pipeline-agent -- -1 Engine alert section added (cumulative queue + today's activity); candidate store fully replaces deleted Notion -1 Scanner DB; all-candidate card posting aligned with neg1-enricher
- draft-investment-memo -- new canonical formatting exemplar; round overview table now populated via deterministic harness (script output pasted verbatim, never hand-derived); simplified Diligence Materials table
- research-agent -- added two fund families to research universe
**Total skills:** 37
**Functions:** Pipeline Management +1 (neg1-sourcing-listener), Diligence Management +1 (founder-taste)

---

## [2026-07-17] (Week of 2026-07-14)

**Added:** log-deal-share (Pipeline Management — categorized from pending since 2026-07-15)
**Removed:** None
**Modified:**
- neg1-enricher -- migrated to headless candidate-store mode; evaluations write to candidate store and post cards to #neg1-sourcing; Notion -1 Scanner DB retired
- neg1-sourcing -- expanded to 8 cold candidates via rotating archetype recipes; monthly network deep sweep and quarterly departure diff added; immediately enqueues per-candidate enrichment jobs instead of batching
- pipeline-agent -- updated pre-founder pipeline Tasks 6–7 to candidate-store v2 flow; added trashed-draft detection and card-timestamp tracking
- decision-retro -- added structured decision-ledger logging at capture time; neg1-retro-scan sub-skill retired; framework reference paths updated
- founder-outreach -- store mode (v2) added as default invocation; multiple invocation modes formalized; Notion -1 Scanner DB retired
- skill-map-refresh -- added log-deal-share to Pipeline Management canonical mapping
- claude-alerts-listener, decision-retro-listener, draft-feedback, first-pass-diligence, network-scan -- framework reference paths updated following 2026-07-16 knowledge-base consolidation
**Total skills:** 35
**Functions:** Pipeline Management +1 (log-deal-share categorized)

---

## [2026-07-16] (Week of 2026-07-14)

**Added:** None
**Removed:** None
**Modified:**
- add-to-crm -- simplified Opportunity page body to a single Original Email section; removed Summary section since deal metadata lives in Notion properties
- research-agent -- updated research universe: removed three public equity managers, added one new manager
**Total skills:** 35
**Functions:** No changes

---

## [2026-07-15] (Week of 2026-07-14)

**Added:** log-deal-share (uncategorized — pending function assignment by Tom)
**Removed:** None
**Modified:**
- investor-update -- added stripped-attachment guard: detects when iPhone/Apple Mail forwards drop the attachment; skips row creation and alerts when the body has insufficient content to reconstruct the update on its own
**Total skills:** 35
**Functions:** No changes (log-deal-share pending categorization)

---

## [2026-07-14] (Week of 2026-07-14)

**Added:** None
**Removed:** None
**Modified:**
- soi-portfolio-event -- added idempotence preflight gate (soi_preflight.py) that classifies each event as NO-OP / DATE-REPAIR / ENGAGE before any Notion writes or Slack; Holdings conveying convention updated to per-round inline lines (no column-aligned tables, which shatter on Slack mobile); mark command now requires per-round share breakdown
**Total skills:** 35
**Functions:** No changes

---

## [2026-07-13] (Week of 2026-07-07)

**Added:** None
**Removed:** None
**Modified:**
- soi-portfolio-event -- Tier 1 field-driven events now require Tom's in-thread confirm before publishing (was rebuild-only); all drafts include a before→after portal diff with methodology walkthrough; tightened coinvestor selectivity to major institutional funds only
- soi-refresh-inputs -- added step to restate the pinned quarterly record after a financial re-anchor whose as-of date is a quarter-end
- investor-update -- migrated from standalone page-per-update to shared Company Updates row model (one row per company per period, shared with live call content); board materials now append a Board Update Type value alongside Formal via multi-select; row key changed to company–period label format
**Total skills:** 35
**Functions:** No changes

---

## [2026-07-03] (Week of 2026-06-30)

**Added:** None
**Removed:** None
**Modified:**
- feedback-outreach-drafter -- dedup guard expanded from three-state (sent/draft/neither) to inbox-wide search; deleted drafts now treated as a terminal signal, preventing re-creation on subsequent sweeps
**Total skills:** 35
**Functions:** No changes

---

## [2026-06-29] (Week of 2026-06-23)

**Added:** None
**Removed:** None
**Modified:**
- pass-note-drafter -- eligibility filter now spec-driven via shared trigger file; reads `status_allow` and name-exclusion regex from `shared-references/triggers/pass-note-drafter.json`; added exclusion gate for a specific deal-type category
- materials-handler -- deal-closing documents (term sheets, SAFEs, closing binders, etc.) now route to a nested `Deal Docs/` subfolder within the company's Diligence Drive folder
**Total skills:** 35
**Functions:** No changes

---

## [2026-06-26] (Week of 2026-06-23)

**Added:** None
**Removed:** None
**Modified:**
- first-pass-diligence -- added per-job workspace isolation to prevent concurrent-job workspace collisions; scoped all tmp working paths under a per-page-id directory so parallel workers no longer overwrite each other's files mid-run
- update-diligence-priors -- added guardrail to process all company-provided materials regardless of filename deprecation labels ([DEPRECATED], (deprecated), etc.); added classification rule distinguishing company artifacts (Materials) from researcher-authored analysis (Research & Analysis)
- finalize-diligence -- added same deprecated-label guardrail as update-diligence-priors; added classification rule for company vs. researcher-authored artifacts in the Final Assessment materials list
**Total skills:** 35
**Functions:** No changes

---

## [2026-06-24] (Week of 2026-06-23)

**Added:** None
**Removed:** None
**Modified:**
- add-to-crm -- simplified Opportunity page body to two sections (Original Email verbatim + three-paragraph Summary replacing the former multi-section layout); moved material links from body to property fields; added Funding History property to link prior Opps on re-engagements; tightened URL fidelity rules
**Total skills:** 35
**Functions:** No changes

---

## [2026-06-23] (Week of 2026-06-23)

**Added:** None
**Removed:** None
**Modified:**
- soi-portfolio-event -- added coinvestors enrichment step for new portfolio entries; inspects Opp's Coinvestors relation and links matching Companies DB rows before rebuilding the SOI
**Total skills:** 35
**Functions:** No changes

---

## [2026-06-19] (Week of 2026-06-16)

**Added:** None
**Removed:** None
**Modified:** None
**Total skills:** 35
**Functions:** No changes

---

## [2026-06-18] (Week of 2026-06-16)

**Added:** None
**Removed:** None
**Modified:** None
**Total skills:** 35
**Functions:** No changes

---

## [2026-06-17] (Week of 2026-06-16)

**Added:** soi-portfolio-event, soi-refresh-inputs (Portfolio Management — added to canonical mapping and surfaced to Platform Map and Quick Reference)
**Removed:** None
**Modified:**
- soi-portfolio-event -- SOI engine path migrated from `~/.claude/scripts/soi/` to `~/code/lp-portal/`
- soi-refresh-inputs -- SOI engine path migrated from `~/.claude/scripts/soi/` to `~/code/lp-portal/`
**Total skills:** 35
**Functions:** Portfolio Management — soi-portfolio-event and soi-refresh-inputs added to Platform Map and Quick Reference

---

## [2026-06-16] (Week of 2026-06-16)

**Added:** None
**Removed:** None
**Modified:**
- intro-resolution-agent -- on opt-in reply where no intro has been sent, now auto-enqueues intro-draft-agent via enqueue-intro-draft.sh rather than waiting for next scheduled scan
- intro-draft-agent -- added targeted webhook invocation mode (from intro-resolution-agent on confirmed opt-in); mode pill updated to S+W+M
- draft-investment-memo -- renamed Round OS% field to OS% @ Round throughout
**Total skills:** 33
**Functions:** No changes

---

## [2026-06-15] (Week of 2026-06-15)

**Added:** batch-add-to-crm (Pipeline Management — was in mapping since 2026-06-12, now surfaced to Platform Map and Quick Reference)
**Removed:** None
**Modified:**
- add-conversation-to-notion -- added traceability check requiring extracted frameworks to anchor to specific conversation turns
- add-to-crm -- added founder-background vs. pitched-company conflation guardrail; tightened URL fidelity rule to include tool-result sources; added materials re-poll step on missing URLs
- batch-add-to-crm -- added founder-background vs. pitched-company conflation guardrail
- diligence-qa -- added coverage-minimum validation script gate at JSON emission, before doc build
- finalize-diligence -- moved shape-rule lint to run before Notion write; added speaker-attribution script gate; refined Standing Open Questions audit to use forward_looking verdict
- first-pass-diligence -- added speaker-attribution script gate (code-enforcement of speaker attribution rules)
- inbound-deal-detect -- added update-vs-pitch detection (founder updates skip add-to-crm); added material_urls_ambiguous flag; added medium-confidence evidence gate
- intro-agent -- added canonical lifecycle contract reference; added single-thread provenance rule preventing cross-thread candidate-Opp pairings
- intro-draft-agent -- added lifecycle contract reference; split deferral handling: soft deferrals stay in Outreach for recheck; hard deferrals route to resolution scanner
- intro-note-processor -- added lifecycle contract reference; added boundary examples for intro commitment extraction
- intro-outreach-agent -- added lifecycle contract reference
- intro-resolution-agent -- added lifecycle contract reference
- investor-update -- added pre-write grounding gate (deterministic + semantic layers) validating all metrics against source before Notion write
- log-intro -- added lifecycle contract reference
- log-transcript-to-notion -- added traceability check requiring extracted frameworks to anchor to actual transcript content
- materials-handler -- added outcome label for failed runs; clarified processed vs. failed label logic; added credential redaction for demo chip passwords in Slack alerts
- neg1-enricher -- made 4-round online presence search sequential and mandatory; added URL verification post-write; clarified Intentionality handling in Eval Summary
- pipeline-agent -- removed Chrome environment gate; simplified materials delegation (full flow via public Notion API in all cases)
- pre-mortem -- added specificity gate requiring failure modes to anchor to company-specific diligence facts; capped re-prompts at MAX_ITER=2
- product-build-teardown -- added calibration-cite resolution verification script; enhanced killshot requirements to include named competitor evidence
- research-agent -- added cadence annotation for curated firms; deny-listed firms now logged in a digest footer for auditability
- update-diligence-priors -- added speaker-attribution script gate before audit
**Total skills:** 32
**Functions:** Pipeline Management — batch-add-to-crm surfaced to Platform Map and Quick Reference

---


## [2026-06-12] (Week of 2026-06-09)

**Added:** batch-add-to-crm (discovered — pending function assignment, not yet on Platform Map)
**Removed:** None
**Modified:**
- decision-retro -- freshness gate now code-enforced via queue_append.py; sweep callers reject stale opps at the script level; webhook and manual triggers bypass gate
- inbound-deal-detect -- classifier restructured to support multi-company pitch emails; output schema changed from single-company object to array; single-company pitch remains the default
- research-agent -- added one firm to Tier 1 curated list (minor update, no behavioral change)
**Total skills:** 32
**Functions:** No changes

---

## [2026-06-10] (Week of 2026-06-09)

**Added:** None
**Removed:** None
**Modified:**
- pass-note-drafter -- added curated anchor examples as highest-priority voice calibration source; positioned as first read in Step 5 stylebook sequence, outranking rolling voice examples when they disagree
**Total skills:** 32
**Functions:** No changes

---

## [2026-06-09] (Week of 2026-06-09)

**Added:** None
**Removed:** None
**Modified:**
- investor-update -- added dual-PDF mandate for Case A (email body saved as canonical update PDF alongside attachment with descriptive suffix); changed Notion body link from company Drive folder to email-body PDF file; switched dedup logic from title search to Source Email exact match
**Total skills:** 32
**Functions:** No changes

---

## [2026-06-08] (Week of 2026-06-08)

**Added:** diligence-qa (Diligence Management — was mapped on 2026-06-04 but missing from HTML; added to Platform Map and Quick Reference this run)
**Removed:** None
**Modified:**
- finalize-diligence -- added mandatory transcript speaker-labeling via relabel_transcript.py; added speaker attribution guardrail to Final Assessment (prevents investor reframings from being misattributed to founders in the synthesis block)
- first-pass-diligence -- added mandatory transcript speaker-labeling step; made --labeled-transcripts-dir flag required in first_pass_lint.py for deterministic attribution checks; added Common Deviation #28 documenting the speaker misattribution failure mode
- update-diligence-priors -- added mandatory speaker-labeling for new transcripts; added speaker attribution guardrail consistent with finalize-diligence and first-pass-diligence
**Total skills:** 32
**Functions:** Diligence Management +1 (diligence-qa)

---

## [2026-06-04] (Week of 2026-06-02)

**Added:** None
**Removed:** None
**Modified:**
- intro-agent -- added Pattern 4 (Founder Greenlight): when Tom proposes intro candidates to a portfolio founder and the founder approves a subset, greenlit names are written to Qualified; includes Step 1.5 thread-pair reconciliation and a third Gmail sent query to catch outbound proposals lacking intro keyword
- intro-outreach-agent -- added Gate 8 peer-ask suppression: classifies Tom's outbound intent as intro-offer vs peer-ask; suppresses false Outreach writes when Tom is asking peers for deal opinions rather than facilitating a connection
- log-investor-letter-to-notion -- added Step 1.5 Drive upload for provided PDFs so source links point at canonical artifacts; added hard guard preventing self-referential source links
- first-pass-diligence -- added Step 5c PDF Format Guard: re-extracts rendered PDF with PyMuPDF and verifies typography (font/size/color) against measured reference profile before Drive upload; catches heading size regressions, colored hyperlinks, and superscript citation bugs (F1-F10 checks)
- update-diligence-priors -- added PDF Format Guard gate (same F1-F10 checks as first-pass-diligence) before Drive upload; mandatory stop on typography failure
**Total skills:** 31
**Functions:** No changes

---

## [2026-06-03] (Week of 2026-06-02)

**Added:** None
**Removed:** None
**Modified:**
- finalize-diligence -- added PDF format guard gate (Gate 3) between PDF build and Drive upload; re-extracts the rendered PDF with PyMuPDF and validates typography, citation format, heading structure, and page dimensions against a measured reference profile; catches formatting regressions (wrong heading sizes, colored hyperlinks, citations rendered as literal body text) that the header check gate misses
**Total skills:** 31
**Functions:** No changes

---

## [2026-05-29] (Week of 2026-05-26)

**Added:** finalize-diligence, draft-investment-memo, product-build-teardown, log-pass-note-guidance (Diligence Management — four skills previously tracked in canonical mapping now rendered on Platform Map and Quick Reference)
**Removed:** None
**Modified:**
- first-pass-diligence — added memo text cache step for analog-grounding lint gate; added mandatory inner H1 anchor rule for Master Diligence Doc structure; added architectural overlay guardrail requiring analog characterizations to trace verbatim to source; added operational specificity and deictic completeness writing rules
- finalize-diligence — added three intermediate publish-progress alerts during the publish phase (Final Assessment prepend, PDF upload, Diligence Materials property update)
- update-diligence-priors — added three intermediate publish-progress alerts during the publish phase (Update section prepend, PDF upload, Materials link step)
**Total skills:** 31
**Functions:** Diligence Management now lists 12 skills (was 8). Four skills surfaced: finalize-diligence, draft-investment-memo, product-build-teardown, log-pass-note-guidance.

---

## [2026-05-27] (Week of 2026-05-26)

**Added:** None
**Removed:** None
**Modified:**
- investor-update — expanded content-type routing for board materials and additional material formats; updated freshness rules for sweep alert scoping
- decision-retro — expanded dual-scanner pattern to capture invest, pass, and pre-founder decision retros; routes extracted insights into searchable feedback corpus
- inbound-deal-detect — added forwarded-message routing (direct vs referrer-sourced); added material URL gate to prevent classifier false positives on deck links
- intro-outreach-agent — expanded pre-write guard from 4 to 7 gates; added Portfolio Fundraise Forward scanner sub-routine for fundraise-context outreach
**Total skills:** 27
**Functions:** No changes

---

All notable changes to the Inverted Cap Stack platform are documented here.

## [2026-06-02] (Week of 2026-06-02)

**Added:** None
**Removed:** None
**Modified:**
- add-to-crm -- added webhook mode with classifierHints / statusDirective / sourceDirective passthrough from upstream classifier; LinkedIn enrichment switched to Chrome osascript (WebFetch 401s on LinkedIn); added Step 8 Slack alert for webhook-mode completions
- decision-retro -- added freshness gate: skip Opps with Close Date more than 60 days before today during scheduled sweep; webhook-triggered entries bypass the gate and always process
- first-pass-diligence -- added operational specificity and deictic completeness writing rules
- inbound-deal-detect -- delegates to add-to-crm via enqueue-addcrm.sh follow-on job instead of inline execution
- intro-resolution-agent -- Mode B switched to atomic JS-backed write endpoint (intro-resolution-write.py) for 4-field lifecycle moves; added idempotency pre-check before classification; alert wording composed from observed post-write state
- neg1-enricher -- Type options renamed: Warm ☀️ (was Reconnect 👋) and Cold 🧊 (was Cold ☎️); added mandatory Step 6.5 post-write field validation
- neg1-sourcing -- Type options renamed to match neg1-enricher (Warm ☀️ / Cold 🧊); added Step 1.5 ContactOut verification gate for cold candidates; Slack alert uses GFM links
- pre-mortem -- failure-scenario speculation formally excluded from audit scope
- update-diligence-priors -- multi-child priors (Killshots, Risks) use H4 parent with bold-inline paragraph leaders for children
**Total skills:** 31
**Functions:** No changes

---

## [2026-05-26] (Week of 2026-05-19)

**Added:** neg1-sourcing (Pipeline Management — added to Platform Map and Quick Reference as a standalone skill)
**Removed:** None
**Modified:**
- neg1-sourcing — script path corrected from scheduled-tasks directory to skills directory; no behavioral change
**Total skills:** 27
**Functions:** No changes

---

## [2026-05-25] (Week of 2026-05-19)

**Added:** None
**Removed:** None
**Modified:**
- first-pass-diligence — artifact renamed from "First-Pass Diligence" to "Master Diligence Doc" throughout (Notion page title, PDF filename, PDF header); killshot subsections reformatted to H4 headers; added mandatory Opus-tier drafting rule
- pre-mortem — added mandatory LLM audit gate (Step 2.5) with research-artifact-audit wiring; factual claims in failure-mode analysis must trace to source bundle before delivery
- update-diligence-priors — added mandatory Opus-tier drafting rule; prior subheads reformatted to H4 headers; added mandatory LLM audit gate (Step 4.5) scoped to the new Update block only
**Total skills:** 26
**Functions:** No changes

---

## [2026-05-22] (Week of 2026-05-19)

**Added:** None
**Removed:** None (skill directory entry for neg1-sourcing was removed; it was not rendered on the stack page)
**Modified:**
- update-diligence-priors — Step 5 Notion write tooling split into MCP (initial markdown-to-blocks conversion) and direct REST API calls (title updates, block deletions, large-payload operations); adds timeout handling and block consolidation logic
- inbound-deal-detect — classifier tightened for investment-entity senders: entities explicitly raising venture capital are now always classified as deals regardless of entity type, overriding the peer-firm exclusion
**Total skills:** 26
**Functions:** No changes

## [2026-05-21] (Week of 2026-05-19)

**Added:** None
**Removed:** None
**Modified:**
- `first-pass-diligence` -- expanded §2 Product section with canonical product-build framework (six sections: Product Anatomy, Delivery Mechanism, Build Cost to v1, Path to Production-Grade, Moat Read, Killshots); PDF naming updated to `_v[N].pdf` convention; added retention rule to remove prior diligence-snapshot PDFs after each update run
- `update-diligence-priors` -- added product-section refresh mode aligned to shared build framework; PDF naming updated to `_v[N].pdf` convention matching first-pass convention
**Total skills:** 25
**Functions:** No changes

---

## [2026-05-20] (Week of 2026-05-19)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-contacts` -- Tom-supplied company, role, or email now overrides ContactOut when provided explicitly; email domain mismatch between Tom's input and ContactOut triggers preference for Tom's data
- `first-pass-diligence` -- LLM-judge audit logic extracted into shared audit module; added start-time anchoring for elapsed-time reporting; added citation grouping format rule; added deterministic final-draft file selection (prefer normalized over raw)
- `neg1-enricher` -- Type field behavior updated: manual mode defaults to Cold unless Tom's phrasing indicates a prior relationship (then Reconnect); bulk enrichment mode does not write the field
- `research-agent` -- expanded curated firm list with 7 new targets; updated canonical firm naming for two existing targets
**Total skills:** 25
**Functions:** No changes

---

## [2026-05-19] (Week of 2026-05-19)

**Added:** None
**Removed:** None
**Modified:**
- `skill-map-refresh` -- updated internal scheduled-task configuration to track two additional personal automation LaunchAgents under the Admin hidden category; no visible skill changes
**Total skills:** 25
**Functions:** No changes

---

## [2026-05-18] (Week of 2026-05-18)

**Added:** None
**Removed:** neg1-sourcing (Pipeline Management)
**Modified:**
- `first-pass-diligence` -- added hard rule against bypassing audit script failures with an alert escape hatch; added mandatory Notion property write verification after each Diligence Materials PATCH; added footnote unescaping regex
- `update-diligence-priors` -- added mandatory pre-PDF lint gate (Step 5a) before each PDF build; enhanced page break rule to insert breaks between every update section in multi-update PDFs; added mandatory Notion property write verification
**Total skills:** 25
**Functions:** Pipeline Management -1 (neg1-sourcing removed)

---

## [2026-05-15] (Week of 2026-05-11)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-contacts` -- profile photo extraction now always required; added fallback to `profile_only` ContactOut endpoint when the primary enrichment call returns no photo
- `pass-note-drafter` -- added mandatory Pass Note Guidance check (Step 3f): must scan Opp body for a "Pass Note Guidance" section before drafting in both sweep and webhook modes; guidance overrides call notes and materials when substance conflicts
- `first-pass-diligence` -- added `include_transcript: true` requirement on all Notes-DB fetches (meeting notes were returning only AI summaries without it); added video diligence support: local video files transcribed via Whisper (`small` model), Loom/hosted video via in-page transcript panel extraction
- `update-diligence-priors` -- added `include_transcript: true` requirement on all Notes-DB fetches, mirroring the first-pass-diligence rule so update runs operate on the same data fidelity as original analyses
- `decision-retro` -- clarified `notion-fetch` call: Opp/-1 target pages do not get `include_transcript`, but Lookup-mode fetches of linked Notes-DB meeting notes do
**Total skills:** 26
**Functions:** No changes

---

## [2026-05-14] (Week of 2026-05-11)

**Added:** neg1-promote (Pipeline Management — pipeline-agent composite sub-skill)
**Removed:** None
**Modified:**
- `add-to-crm` -- enhanced dedup: harvests signals from inner forwarded-message headers (From, Subject, Cc) to catch same-pitch-forwarded-by-different-referrers duplicates
- `first-pass-diligence` -- alert body switched to GFM link syntax; Files property writes migrated to public Notion API (internal API + Chrome path retired)
- `intro-agent` -- word-boundary corroboration gate expanded: now mandatory for ALL Opp name matches regardless of name length (removed 6-char size threshold)
- `intro-resolution-agent` -- same corroboration gate tightening as intro-agent; bare word-boundary matches with no corroboration are skipped
- `materials-handler` -- added per-message idempotency gate (Step 2.5) using claude/materials-processed Gmail label to prevent duplicate uploads; Files property writes migrated to public Notion API
- `neg1-enricher` -- Online Presence URL writes migrated from osascript/Chrome to public Notion API via notion_files_property.py; Chrome dependency fully removed
- `research-agent` -- added Artisan Partners to scheduled scan targets; Crescat Capital repositioned in tier list
- `update-diligence-priors` -- strict incremental processing: now assembles "already processed" set from all prior update sections before identifying new materials; each run is additive only
**Total skills:** 26
**Functions:** No changes

---

## [2026-05-13] (Week of 2026-05-11)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-crm` -- thin-body enrichment extended to hyperlinked deck URLs (Drive, DocSend, Brieflink, Pitch, Figma, Canva, raw PDFs) in addition to PDF attachments; verification gate added to confirm Diligence Materials property was populated post-entry
- `feedback-outreach-scanner` -- added Mode B-outbound sub-mode: per-event webhook fires on outbound feedback-ask sends (in addition to existing inbound-reply mode)
- `intro-outreach-agent` -- added 4th pre-write gate: Pending Feedback suppression (backchannel outbound to a feedback contact is not intro outreach)
**Total skills:** 26
**Functions:** No changes

---

## [2026-05-12] (Week of 2026-05-12)

**Added:** neg1-sourcing (Pipeline Management)
**Removed:** None
**Modified:**
- `skill-map-refresh` -- added neg1-sourcing to Pipeline Management function mapping; added decision-retro-scan, neg1-retro-scan, retro-weekly-summary, neg1-sourcing LaunchAgents to scheduling table; added drive-save to hidden Admin category; added intro-note-processor as intro-agent composite sub-skill
**Total skills:** 26
**Functions:** Pipeline Management +1 (neg1-sourcing)

---

## [2026-05-12] (Week of 2026-05-12)

**Added:** None
**Removed:** None
**Modified:**
- `add-to-companies` -- requires hook bypass marker before Notion create call; existing enrichment flow unchanged
- `add-to-contacts` -- requires hook bypass marker before Notion People create call
- `add-to-crm` -- requires hook bypass marker before Notion Opportunities create; custom-domain From: wins over ContactOut for Contact email at creation time; honors explicit status directive passed by caller; third-party-referrer-forward definition expanded to cover additional forward shapes
- `decision-retro` -- neg1 daily prompts now include LinkedIn link in header when the field is populated
- `feedback-outreach-drafter` -- email body switched from `<p>` to `<div>` blocks for iOS Mail / Gmail div-layout compatibility
- `founder-outreach` -- removed numeric Eval Score requirement; now checks Claude Rec + Eval Breakdown (High/Medium/Low ratings) before drafting
- `intro-agent` -- added directionality anti-pattern section (inbound offers to intro Tom are not intro events); added pre-write gates (directionality, terminal-status, common-word Opp disambiguation)
- `intro-outreach-agent` -- removed "want to connect" from Gmail subject search; added pre-write gates (directionality, terminal-status, common-word disambiguation); forward-scanner gate requires From: to match Opp's Contact email
- `intro-resolution-agent` -- added pre-write gates (terminal-status, common-word Opp disambiguation)
- `investor-update` -- removed post-processing email unflag step; step numbering updated accordingly
- `log-intro` -- added terminal-status guard; skips write and surfaces confirmation prompt when Opp is in a terminal status
- `neg1-enricher` -- evaluation verdict now uses High/Medium/Low signal ratings instead of a numeric score; primary employer field renamed CurrentCo (CC); added company cache write-back step after enrichment
- `network-scan` -- revenue-trajectory queries now run csearch + deal-digest intersection pipeline for factual ARR grounding
- `update-diligence-priors` -- deduplicates inline attachments against existing Diligence Materials property before uploading to Drive
**Total skills:** 25
**Functions:** No changes

---

## [2026-05-11] (Week of 2026-05-11)

**Added:** None
**Removed:** None
**Modified:**
- `deal-digest` -- after page update, now also rebuilds the deal digest cache and backfills the Companies DB mentions relation; both steps are idempotent
- `investor-update` -- material and demo URLs (Loom, YouTube, product links) now added to Artifacts via headless helper in addition to page body; sweep alerts restricted to the lookback window per freshness framework
- `materials-handler` -- added hard preconditions for Company Blurb writes (idempotency check, email-body-only source requirement, recognizable blurb opener gate); alert accuracy rule added (only report Company Blurb if written this run); WebFetch scope clarified as Round Details only
- `network-scan` -- added `csearch` (company-side) search primitive for company-trait queries; Step 2 split into 2a (csearch) and 2b (vsearch) with routing logic; csearch returns company groups with per-person tenure data pre-extracted
**Total skills:** 25
**Functions:** No changes

---

## [2026-05-08] (Week of 2026-05-04)

**Added:** None
**Removed:** None
**Modified:** None
**Total skills:** 25
**Functions:** No changes

---

## [2026-05-07] (Week of 2026-05-04)

**Added:** `network-scan` (Intro Management), `company-scan` (Research Management)
**Removed:** None
**Modified:**
- `add-to-companies` -- switched identity enrichment from ContactOut to Exa+LLM extraction; dropped LinkedIn URL and Revenue/Funding Status fields from schema; icon now from Notion picker instead of ContactOut logo
- `add-to-contacts` -- added step to cache raw ContactOut payloads to local file before field extraction; added explicit no-permission-prompts section
- `add-to-crm` -- added name-only collision filter requiring corroborating signal beyond bare name match; removed confirmation gate before page creation (auto-creates); icon changed from company logo to thematic emoji
- `deal-digest` -- added handling for sourceless entries (omit source header when unknown); added write-time discipline section to prevent escaped characters and literal backslash-n in Notion payloads
- `decision-retro` -- added `would_back_again` and `would_back_again_rationale` fields to schema and LLM extraction; updated calibration corpus reference to FOUNDER_EVAL_CASEBOOK
- `first-pass-diligence` -- added skip guard for follow-on rounds; public-presence absence now flagged only when role requires visibility (CEO during fundraise, GTM, marketing); removed leading date stamp from page body
- `founder-outreach` -- replaced multi-step draft creation flow with atomic gmail-create-draft.py helper that creates draft and snapshot in one call
- `intro-agent` -- removed auto-People-entry creation; missing contacts now flagged for manual add instead of auto-enriched
- `investor-update` -- extended scope to include board materials; added Board Meeting title convention; expanded detection for board-related emails and Google Slides share notifications
- `materials-handler` -- switched gmail label application to gmail-label.py helper script; alert format changed from Slack mrkdwn to GFM; added portfolio-set status guard to webhook mode
- `neg1-enricher` -- score tiers renamed from Reach Out/Second Look/Pass to Strong/Moderate/Weak; added ContactOut payload caching step; updated calibration corpus reference to FOUNDER_EVAL_CASEBOOK
- `network-scan` -- expanded scope; now queries People DB in addition to LinkedIn cache; broader use cases (intros, angel identification, advisor discovery)
- `pipeline-agent` -- added rule that calendar-invite-sent (not accepted) triggers Scheduled status; portfolio-set opps excluded from materials scan (Active Portfolio, Follow-On, Exited -- not Committed)
- `pre-mortem` -- added founder-thesis journey-length mismatch as a sub-dimension of Team/Org Fragility Risk
- `research-agent` -- added curation feedback handling: Tom's rejections permanently edit skill tier lists and deny-list; added Diamond Hill Capital to deny-list
- `skill-map-refresh` -- major Platform Map restructure: added Companies as a 6th Notion DB column; reordered DB row to Opportunities → -1 Scanner → Company Updates → People → Companies → Notes; tightened all 6 DB descriptions to 3-line fit; introduced a horizontal "Retrieval Layer (Cache, Enrichment & Embeddings)" annotation strip above the DB row (full-width, dotted halftone fill, white uppercase text matching Agentic Workflows weight). Earlier same-day iterations explored a parallel SQLite cache row (People Cache / Companies Cache / Deals Digest Cache with dashed connectors) before consolidating to the single Retrieval Layer annotation; the cache-row treatment is now documented as "do not re-introduce."
- `update-diligence-priors` -- fixed Notion search prefix from `Claude:` to `[Claude]` format for first-pass diligence page lookup
**Total skills:** 25
**Functions:** Intro Management +1 (network-scan), Research Management +1 (company-scan)

---

## [2026-04-30] (Week of 2026-04-27)

**Added:** None
**Removed:** None
**Modified:**
- `first-pass-diligence` -- added a structured per-founder evaluation section that scores each founder against the Founder Eval Framework, with raw LinkedIn scrape + online presence research as inputs; each founder block contains a headline (Working Description + Claude Rec), per-signal scoring with transcript-grounded rationale + spike summary, and a conflict callouts list surfacing inconsistencies between conversational framing and public record. Runs from scratch independent of any prior -1 Scanner score.
- `research-agent` -- added a Tier 2 deny-list of firms Tom has explicitly rejected; matched discovery results are dropped silently from the digest. Added a CEO / leadership letters tier covering annual letters from large public firms whose voice carries broad market weight.
- `deal-digest` -- tightened the prepend-at-top rule: every new ingest sits above every existing block regardless of date stamp. Log order governs vertical position, not date stamp.
- `pipeline-agent` -- tightened Gmail `in:sent` queries with mandatory `-is:draft` to prevent drafts leaking into sent-mail thread results and misclassifying outreach signals.
- `feedback-outreach-scanner` -- switched note title delimiter from em dash to colon to match the canonical title pattern across feedback notes.
- `draft-feedback`, `founder-outreach` -- relocated draft-snapshots Drive directory under `_system/` to keep system data segregated from user-facing Drive content.
**Total skills:** 23
**Functions:** No changes

---

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

