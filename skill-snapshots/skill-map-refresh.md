---
name: skill-map-refresh
description: >
  Scan for newly added or modified skills in the skills directory, update both the Quick Reference
  and Platform Map visuals in Notion, render PNGs of the visuals, generate responsive HTML for the
  /stack page that preserves the PNG's exact aspect ratio, and push the HTML to GitHub
  (t-seaux/invertedcap-stack repo). Runs daily at 7:30 AM ET. Trigger phrases include: "refresh skill
  map", "update stack page", "rebuild platform map", "skill map refresh", "update the visuals",
  "regenerate stack visuals", or any variant requesting an update to the platform map or quick
  reference visuals.
---

# Skill Map Refresh

Scan the skills directory for changes, regenerate both the **Platform Map** (stack map) and **Quick Reference** visuals, render PNGs, and push responsive HTML to GitHub. This skill is the single source of truth for the implementation -- the `skill-map-refresh` local LaunchAgent invokes it daily.

---

## Architecture Overview

The skill produces two artifacts per visual (Platform Map + Quick Reference):

1. **PNG render** — A screenshot of the HTML artifact, used as the reference image and for aspect-ratio measurement.
2. **Responsive HTML** — A single `stack-page.html` file pushed to the `t-seaux/invertedcap-stack` GitHub repo, embedding both visuals inline with full responsive scaling. GitHub is the canonical surface; the live site at `invertedcap.com/stack` renders directly from this file.

> **Retired surface**: Earlier revisions of this skill also mirrored both visuals into two Notion host pages ("Skill Map — Stack Map" and "Skill Map — Quick Reference"). Those pages were retired on 2026-04-20 after drifting silently out-of-sync across multiple scheduled runs. Do not re-introduce a Notion write step — GitHub is the single source of truth.

---

## Step 0: Discover Current Skills

Scan the skills directory to build a complete inventory.

> **CRITICAL -- Anti-hallucination guardrail**: Every skill name that appears in the Platform Map, Quick Reference, or any output artifact MUST come from the actual `Glob` results in step 1 below. Do NOT invent, guess, or infer skill names. If a skill doesn't appear in the glob results, it doesn't exist. The canonical function mapping table below is the ONLY authority for which skills belong to which function. If a skill appears in the glob results but is NOT listed in the canonical mapping, flag it for Tom -- do NOT place it in a function or invent a function for it. Violating this guardrail produces hallucinated output that degrades what's live in production.

1. Use `Glob` with pattern `**/skills/*/SKILL.md` to find all skill files.
2. For each SKILL.md, read the YAML frontmatter to extract:
   - `name` (the skill identifier)
   - `description` (first sentence only, for the Quick Reference one-liner)
3. Classify each skill with a **primary trigger badge**: one of **Scheduled**, **Ad Hoc**, or **Event**.

   **Scheduled** — the skill runs on a cron via local launchd. Run `launchctl list | grep com.tomseo.scheduled` to fetch active LaunchAgents. Orchestrator LaunchAgents subsume multiple canonical skills — all subsumed skills count as Scheduled even though only the orchestrator itself is the LaunchAgent. Do NOT render sub-skill names as separate scheduled entries beyond what's mapped here.

     | LaunchAgent name | Canonical skill(s) |
     |---|---|
     | `research-agent` | `research-agent` |
     | `diligence-agent` | `diligence-agent`, `feedback-outreach` (absorbs former `feedback-outreach-drafter` + `feedback-outreach-scanner` as microsteps of one "get feedback on an Opp" value chain), `pass-note-drafter`, `first-pass-diligence` |
     | `pipeline-agent` | `pipeline-agent` |
     | `intro-agent` | `intro-agent` (absorbs former sub-skills `intro-outreach-agent`, `intro-draft-agent`, `intro-resolution-agent`, `log-intro`; see Function mapping below) |
     | `intro-agent-enricher` | `intro-agent` (queue processor for `intro-agent/processor.py`; rolls up into parent) |
     | `portfolio-agent` | `investor-update` |
     | `admin-agent` | `note-classifier` |
     | `office-cleaning-expense` | `office-cleaning-expense` |
     | `skill-map-refresh` | `skill-map-refresh` |
     | `draft-feedback-processor` | `draft-feedback` |

   **Excluded LaunchAgents** — infrastructure-only agents that do not correspond to a skill and must NOT be rendered on the Platform Map or Quick Reference. Listed here so the Step 0 guardrail finds them and does not flag them as unclassified.

     | LaunchAgent name | Reason |
     |---|---|
     | `skills-autosync` | Infrastructure — syncs skill files, not a user-facing skill |

   **Event** — the skill's primary trigger is now a Gmail webhook handler (Apps Script `gmail-webhook` project, invoked via Pub/Sub push) rather than a cron sweep. A skill is Event when its primary inbound/outbound detection happens in response to a Gmail lifecycle event. Handlers roll up into their parent skill per the table below; only the parent skill is rendered, not the handler.

     | Webhook handler | Rolls into (parent skill) |
     |---|---|
     | `deal-scanner-inbound` | `pipeline-agent` |
     | `pipeline-sent-detect` | `founder-outreach` |
     | `pass-note-sent-detect` | `pass-note-drafter` |
     | `draft-trash` | `draft-feedback` |
     | `feedback-reply-detect` | `feedback-outreach` |
     | `intro-outreach` | `intro-agent` |
     | `intro-made` | `intro-agent` |
     | `intro-resolution-reply` | `intro-agent` |
     | `intro-agent-inbound` | `intro-agent` |
     | `investor-update-inbound` | `investor-update` |

   **Ad Hoc** — the skill is invoked manually by Tom (trigger phrase in chat, no cron, no webhook).

   **Primary-badge rule when multi-mode**: a skill's badge reflects the trigger that drives the majority of its runs *now*, not historical classification. Redundant scheduled coverage that remains while webhook coverage is being validated does NOT override an Event badge — if the webhook is the primary inflow, the skill is Event. Apply these resolutions:

     | Skill | Badge | Rationale |
     |---|---|---|
     | `pipeline-agent` | Event | Top-of-funnel detection now webhook-driven via `deal-scanner-inbound`; remaining cron sweeps (Qualified Triage, stagnation alerts, Connected Triage) are residual fallbacks |
     | `intro-agent` | Event | Inbound/outbound detection now webhook-driven end-to-end |
     | `investor-update` | Event | Webhook-driven ingestion; scheduled skill retained as fallback during validation |
     | `feedback-outreach` | Event | End-to-end get-feedback-on-an-Opp chain (drafter + reply detection); webhook drives reply tracking, scheduled scan retained as fallback |
     | `draft-feedback` | Event | Webhook triggers enqueue; scheduled processor drains the queue |
     | `founder-outreach` | Ad Hoc | Tom initiates; webhook is passive completion detection |
     | `pass-note-drafter` | Ad Hoc | Same as founder-outreach |

   If a new LaunchAgent appears (`com.tomseo.scheduled.<new-name>` not in this table) or a new webhook handler is registered in Apps Script (check `gmail-webhook` Code.gs `handlers` array), flag it in the summary and skip classification rather than guessing.
4. Group skills into **Functions** using this canonical mapping:

| Function | Skills |
|---|---|
| Pipeline Management | `pipeline-agent`, `add-to-crm`, `neg1-enricher`, `founder-outreach`, `add-to-contacts`, `materials-handler`, `draft-feedback` |
| Intro Management | `intro-agent` (single box — absorbs former `intro-outreach-agent`, `intro-resolution-agent`, `intro-draft-agent`, `log-intro` as microsteps of one end-to-end value chain) |
| Portfolio Management | `investor-update`, `coinvestor-recommender` |
| Diligence Management | `diligence-agent`, `feedback-outreach` (absorbs drafter + scanner), `pass-note-drafter`, `first-pass-diligence`, `update-diligence-priors`, `pre-mortem`, `add-conversation-to-notion`, `decision-retro` |
| Research Management | `research-agent`, `log-transcript-to-notion`, `deal-digest`, `log-investor-letter-to-notion`, `add-to-companies` |

#### Hidden Categories (tracked but NOT rendered on the stack page)

These functions are tracked internally for completeness but do NOT appear in ANY public-facing outputs -- no Platform Map, no Quick Reference, no Changelog entries, and no commit messages. Do not reference hidden skill names, hidden function names, hidden skill counts, or the existence of hidden categories anywhere in the GitHub repo (CHANGELOG.md, commit messages, HTML). The "Total skills" count in the changelog must reflect visible skills only -- no secondary "all incl. hidden" count.

| Function | Skills | Why hidden |
|---|---|---|
| Fund Ops | `mmf-to-lp-calc`, `cpa-report` | Operational fund accounting -- not part of the deal/research workflow |
| Admin | `note-classifier`, `uhc-superbill-filer`, `docsend-to-pdf` | Utility/subroutine skills invoked by other skills or personal automation — no standalone user-facing workflow |

#### Excluded duplicates

These skills exist in the directory but are intentional duplicates of skills already mapped. They are excluded from all outputs (visuals, counts, changelog):

| Skill | Duplicate of | Notes |
|---|---|---|
| `feedback-outreach-drafter-manual` | `feedback-outreach-drafter` | Identical implementation; consolidated into the canonical skill |
| `feedback-outreach-drafter-ad-hoc` | `feedback-outreach-drafter` | Identical implementation; consolidated into the canonical skill |
| `backchannel-drafter` | `feedback-outreach-drafter` | Same skill under a different name; "backchannel" trigger phrases merged into canonical skill |

> **New skill handling**: If a newly discovered skill doesn't fit an existing function, flag it in the summary output for Tom to manually assign. Do NOT create new functions without Tom's approval.

5. Compare the current inventory against the last-known state. Identify:
   - **Added** skills (present in directory but not in current HTML)
   - **Removed** skills (present in HTML but not in directory)
   - **Modified** skills (anything substantive changed in the SKILL.md beyond whitespace)

   For modification detection, do not rely on description-string comparison alone — meaningful edits often happen in the body. Approach:
   - Pull the previous state of each existing SKILL.md from the GitHub repo `t-seaux/invertedcap-stack` (use a sibling `skill-snapshots/` directory tracked alongside `stack-page.html`, one file per skill containing the SKILL.md content as of the last successful run). On the first run after this snapshot mechanism is introduced, seed the directory with the current SKILL.md contents and treat all skills as unmodified for that run.
   - Diff each current SKILL.md against its snapshot. Ignore pure whitespace/formatting changes.
   - For each modified skill, summarize the change in 1-2 short phrases. Examples of what to capture: new or removed trigger phrases, new sub-steps or workflow stages, changed cadence or scheduling behavior, new outputs/destinations, expanded scope, tightened guardrails, bug-fix-driven behavior changes. **Do NOT** paste raw diff hunks — write a human-readable summary.
   - After the diff pass completes, overwrite the snapshot files with the current SKILL.md contents so the next run diffs against this run's state.

If there are zero changes (no Added, Removed, or Modified), report "No skill changes detected — visuals are current" and stop. Do not regenerate or push anything. (Snapshots are still refreshed only when a push happens, so the next run continues to diff against the last published state.)

---

## Step 1: Build the Platform Map (Stack Map)

The Platform Map is a two-tier visual showing Functions (top row) connected via SVG lines to Databases (bottom row), with an annotation layer in between.

### Layout Specification

- **Design width**: 1200px (fixed internal layout)
- **Font**: `'Courier New', Courier, monospace`

#### Unified Color Scheme

There is ONE color scheme for both the web HTML and PNG renders. The page background on the website is `#0d1117` (a cool blue-gray), and all grays are in that same hue family. For PNG rendering, the only change is setting `body { background: #0d1117 }` (the web HTML uses `transparent` for iframe embedding).

| Element | Color |
|---|---|
| Page / body bg (web) | `transparent` (sits on `#0d1117` page) |
| Page / body bg (PNG) | `#0d1117` |
| Function/DB card bg | `#161b22` |
| Card borders | `#30363d` |
| Ad hoc pill bg | `#0d1117` |
| Ad hoc pill border | `#30363d` |
| Event pill bg | `#e0d6c3` (warm cream) |
| Event pill border | `#e0d6c3` |
| Event pill text | `#0d1117` (dark, for contrast on cream) |
| Eyebrow/label text | `#8b949e` |
| Ad hoc pill text | `#8b949e` |
| Description text | `#b1bac4` |
| Annotation box bg | `#161b22` |
| System tag bg | `#161b22` |
| System tag text | `#8b949e` |
| Divider/row borders | `#30363d` / `#21262d` |
| QR badge (ad hoc) bg | `#161b22` |
| QR badge (ad hoc) text | `#8b949e` |
| QR badge (event) bg | `#e0d6c3` |
| QR badge (event) text | `#0d1117` |
| QR skill-line text | `#b1bac4` |
| QR eyebrow text | `#8b949e` |

Scheduled skill pills and function/database names (`#fffffb`) are high-contrast and hue-neutral.

### Top Row — Functions

A 5-column grid (`grid-template-columns: repeat(5, minmax(0,1fr))`) of function cards. Each card contains:
- Eyebrow label: "Function" (9px, uppercase — see color table above)
- Function name (12px, 500 weight, `#fffffb`)
- Skill pills stacked vertically:
  - **Scheduled** skills: white background (`#f0efea`), dark text (`#2c2c2a`), border `#c3c2bd`
  - **Ad Hoc** skills: background and text per the color table above
  - **Event** skills: cream background (`#e0d6c3`), dark text (`#0d1117`), border `#e0d6c3`

#### Composite pill treatment

Skills that are orchestrator-style composites — one pill on the map that absorbs multiple named sub-skills — get an additional `sm-composite` class producing a stacked-card shadow behind the pill. Visual cue that there's more underneath; the Quick Reference carries the expand-to-reveal UI.

**CSS** (additive):
```css
.sm-sk.sm-composite { box-shadow: 2px 2px 0 0 rgba(0, 0, 0, 0.45), 2px 2px 0 0.5px #30363d; }
.sm-sk.sm-event.sm-composite { box-shadow: 2px 2px 0 0 rgba(71, 64, 51, 0.55), 2px 2px 0 0.5px #978f7c; }
.sm-sk.sm-sched.sm-composite { box-shadow: 2px 2px 0 0 rgba(130, 128, 121, 0.5), 2px 2px 0 0.5px #a8a7a2; }
```

**Composites** (as of 2026-04-23): `intro-agent` (absorbs `intro-agent-inbound`, `intro-outreach-agent`, `intro-draft-agent`, `intro-resolution-agent`, `log-intro`) and `feedback-outreach` (absorbs `feedback-outreach-drafter`, `feedback-outreach-scanner`). A skill is a composite only when it *replaces* what would otherwise be multiple pills on the map — orchestrators whose sub-skills are still separately rendered (e.g., `diligence-agent`) do NOT get the composite treatment.

### Middle Layer — SVG Connections

A 120px-tall SVG region with cubic Bezier curves connecting each function to the databases it reads/writes. Connection mapping:

| Function | Databases | Line Colors |
|---|---|---|
| Pipeline Management | Opportunities, People, -1 Scanner | `#7F77DD`, `#1D9E75`, `#D85A30` |
| Intro Management | Opportunities, People | `#7F77DD`, `#1D9E75` |
| Portfolio Management | Opportunities, Company Updates | `#7F77DD`, `#BA7517` |
| Diligence Management | Opportunities, People, Notes | `#7F77DD`, `#1D9E75`, `#888780` |
| Research Management | Notes | `#888780` |

Annotation box centered in the middle: "Agentic Workflows / Continuous read / write across core databases". Background per the color table above. The annotation box has `z-index: 2` to sit above the SVG lines.

#### SVG Pulse Animation

Each connection line is rendered as three layered paths:

1. **Base line** -- always visible at `opacity: 0.2`, `stroke-width: 1.5`. Provides the static topology.
2. **Glow halo** -- `stroke-width: 2.5`, `filter: url(#glow)` (Gaussian blur, `stdDeviation: 0.6`). Opacity breathes between `0.05` and `0.35` via SVG `<animate>`.
3. **Bright core** -- `stroke-width: 2`, no filter. Opacity breathes between `0.12` and `0.55` via SVG `<animate>`.

Both animated layers use `calcMode="spline"` with `keySplines="0.45 0 0.55 1; 0.45 0 0.55 1"` for smooth ease-in-out breathing. Each of the 11 connection lines gets a staggered cycle duration (`2.8s` to `4.0s`, spread via `i % 4 * 0.4`) and a staggered start delay (`i * 0.4 % 2.2`) so they pulse asynchronously.

The SVG `<defs>` block contains a single `<filter id="glow">` with `feGaussianBlur` + `feMerge` (blur node + SourceGraphic). Filter bounds are `-50%` to `200%` to prevent clipping.

**Key design rule**: The animation is a stationary pulse (breathing glow in place), NOT a directional sweep or marching dashes. No gradients, no dash arrays, no `animateTransform`. Pure opacity animation on `<animate>` for maximum browser compatibility.

### Bottom Row — Databases

A 5-column grid of database cards with colored left borders:

| Database | Border Color | Description |
|---|---|---|
| Opportunities | `#7F77DD` | Pipeline and portfolio — core system of record |
| -1 Scanner | `#D85A30` | Pre-qual layer for founders entering pipeline |
| People | `#1D9E75` | Founders, investors, and other network relations |
| Company Updates | `#BA7517` | Updates from portfolio companies |
| Notes | `#888780` | Transcripts, convos, reports, and letters |

### Legend

Above the function cards, include:
- System label: "System of Action" with a "Claude" tag
- Legend swatches in this order: "Ad Hoc Skill" (dark swatch), "Scheduled Orchestrator" (white swatch), "Event-Based Skill" (cream `#e0d6c3` swatch), "Composite – expand in Quick Reference" (cream swatch with stacked-card shadow). The Composite entry uses an en dash (`–`), not an em dash.

Below the divider:
- System label: "System of Record" with a "Notion" tag

---

## Step 2: Build the Quick Reference

A table-style listing of every skill, grouped by function. Each row contains:

- **Badge**: "SCHEDULED" (white bg), "AD HOC" (dark bg), or "EVENT-BASED" (light mint `#9ca3af` bg, dark text `#0d1117`) — 9px uppercase, fixed-width badge
- **Skill name**: 12px, `#fffffb`
- **One-liner**: A concise (~15-word) description of what the skill does, written in present tense. Use the first sentence of the skill's `description` frontmatter as a starting point, but edit for clarity and brevity. The one-liner should be punchy and self-contained.

> **Modified skills**: For any skill flagged as Modified in Step 0, regenerate its one-liner from scratch off the current SKILL.md — do NOT carry over the previous one-liner. The Quick Reference must reflect the *current* behavior of every skill, not the snapshotted behavior. Apply the same redaction rules listed in Step 5 (no personal names, no portfolio company names, no internal identifiers).

Layout: two-column grid per row — `280px` for name+badge, `1fr` for description. Rows separated by `0.5px solid #3e3e3c` borders. All rows use `padding-left: 22px` on `.qr-skill-name` so badges and skill-text align at the same x-position regardless of whether a row has a chevron.

#### Composite expand behavior

Composite skills (see Step 1 composite list) render as a top-level row with a ▶ chevron prefix. The chevron is absolute-positioned at the row's left edge (occupying the 22px indent) so it doesn't shift the badge or text. Clicking the row toggles an `.open` class on the wrapping `.qr-composite-wrap`, which expands a `.qr-sub-container` (CSS grid-template-rows transition from `0fr` to `1fr`) to reveal nested `.qr-sub-row` entries. Sub-rows use the same grid layout as parent rows (12px name, 12px line, `#fffffb` / `#b1bac4`) with `padding-left: 22px` on the sub-name so it aligns with the parent's badge column. No arrow glyph preceding the sub-name.

#### Composite breakdown

For each composite skill, the Quick Reference expand reveals these sub-skills with hand-written one-liners. Update this list when a composite's internals shift.

| Composite | Sub-skill | One-liner |
|---|---|---|
| `intro-agent` | `intro-agent-inbound` | Detects intro requests in Gmail and iMessage and creates Qualified entries on the matching Opportunity. |
| `intro-agent` | `intro-outreach-agent` | Scans sent mail for outreach to Qualified contacts and promotes them to the Outreach stage. |
| `intro-agent` | `intro-draft-agent` | Drafts double-opt-in intro emails when a contact replies positively to outreach. |
| `intro-agent` | `intro-resolution-agent` | Resolves outreach into Made, Declined, or NR based on reply detection. |
| `intro-agent` | `log-intro` | Manually logs a qualified intro target when a person and a company are named together. |
| `feedback-outreach` | `feedback-outreach-drafter` | Drafts backchannel diligence emails for new entries on an Opportunity's Pending Feedback relation. |
| `feedback-outreach` | `feedback-outreach-scanner` | Scans sent mail and inbox for replies, logs Notion notes, and clears contacts off Pending Feedback. |

Functions appear in this order: Pipeline Management, Intro Management, Portfolio Management, Diligence Management, Research Management.

---

## Step 3: Render PNGs

Use Claude in Chrome to render both visuals as PNGs. There is NO separate PNG HTML -- use the same web HTML with `body { background: #0d1117 }` injected via JavaScript before screenshotting.

1. Serve or inject the stack-page.html content into a Chrome tab (use `document.open()/write()/close()` on an existing tab if `file://` is unavailable).
2. Set `document.body.style.background = '#0d1117'` via JavaScript to make the transparent background visible.
3. Resize the Chrome window to 1200px width.
4. Screenshot each section separately (stack map, then quick reference) and save as `stack-map.png` and `quick-ref.png`.
5. If Chrome tools are unavailable, skip PNGs and note in the summary -- the HTML is the primary artifact.

---

## Step 4: Generate Responsive HTML (`stack-page.html`)

Produce a single HTML file that embeds both visuals (Platform Map on top, Quick Reference below) with full responsive scaling. This file is served on the `/stack` page of the Inverted Capital website.

> **CRITICAL -- Use the existing HTML as a base**: Fetch the current `stack-page.html` from the GitHub repo using `mcp__github__get_file_contents`. Use it as the structural and stylistic foundation -- do NOT write a new HTML file from scratch. Apply only the specific changes identified in Step 0 (added/removed/modified skills). This ensures color scheme, CSS classes, responsive scaling logic, and SVG connection drawing remain stable across runs. Only regenerate from scratch if the existing file is missing or if a structural change (e.g., new function column, new database row) requires it.

### CRITICAL: Responsive Scaling Architecture

The entire visual MUST scale as a single unit — like an image — at every viewport width. No reflowing, no truncation, no line-wrapping changes. The approach:

```
Container (100% width, max-width: 1500px, margin: 0 auto, min-width: 300px)
  └── Inner (fixed 1200px design width)
      └── All content laid out at 1200px
      └── transform: scale(containerWidth / 1200)
      └── transform-origin: top left
```

#### Scaling Logic (JavaScript)

```javascript
const DESIGN_WIDTH = 1200;

function scaleAndDraw() {
  const wrapper = document.getElementById('responsive-wrapper');
  const inner = document.getElementById('responsive-inner');
  if (!wrapper || !inner) return;

  const scale = wrapper.offsetWidth / DESIGN_WIDTH;
  inner.style.transform = 'scale(' + scale + ')';
  wrapper.style.height = inner.scrollHeight * scale + 'px';

  drawConnections(scale);
}

window.addEventListener('load', scaleAndDraw);
window.addEventListener('resize', scaleAndDraw);
```

Key requirements:

- **Wrapper height**: Set `wrapper.style.height = inner.scrollHeight * scale + 'px'` so the page flows correctly below the visual. Without this, content below the visual overlaps.
- **SVG coordinate adjustment**: `getBoundingClientRect()` returns visual (scaled) coordinates. SVG paths are drawn in the unscaled 1200px coordinate space. Divide all `getBoundingClientRect()` offsets by `scale` to get correct SVG coordinates:
  ```javascript
  const x1 = (fr.left + fr.width / 2 - mR.left) / scale;
  ```
- **`vw`-based font sizing** as a secondary layer: Apply `font-size` in `vw` units to critical text elements so they scale smoothly with the viewport even before JavaScript executes. Example: `font-size: clamp(8px, 1vw, 12px)`.
- **All widths** inside the inner container use fixed px values or percentages relative to the 1200px design width — never viewport-relative units outside the container.
- **`min-width: 300px`** floor on `.responsive-wrapper` to prevent extreme shrinkage on very narrow viewports.
- **`max-width: 1500px`** cap on `.responsive-wrapper` to prevent the visual from scaling too large on wide monitors. Combined with `margin: 0 auto` to center the visual when the cap is reached.
- The PNG output is the reference for correct aspect ratio. If the rendered PNG is e.g. 1200x1800, set `aspect-ratio: 1200 / 1800` on the wrapper as a CSS fallback for before JS loads.
- **`overflow: hidden`** on `.responsive-wrapper` to prevent horizontal scroll from the 1200px inner element.

#### SVG Connection Line Drawing

The `drawConnections(scale)` function:

1. For each connection pair (function → database), get the bounding rects of both elements.
2. Compute center-x of each element relative to the SVG container.
3. **Divide by scale** to convert from visual coordinates to design coordinates.
4. Draw a cubic Bezier from `(x1, 0)` to `(x2, height)` with control points at 45% and 55% of the height.
5. Apply the connection's color, `stroke-width: 2`, `opacity: 0.6`.

#### HTML Structure

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="robots" content="noindex, nofollow">
  <style>/* All CSS inline */</style>
</head>
<body>
  <div class="responsive-wrapper" id="responsive-wrapper">
    <div class="responsive-inner" id="responsive-inner">
      <div class="page-wrap">
        <div class="page-date">Updated [DATE]</div>
        <!-- Stack Map artifact-frame -->
        <!-- Quick Reference artifact-frame -->
      </div>
    </div>
  </div>
  <script>/* Scaling + SVG drawing */</script>
</body>
</html>
```

#### Full CSS Reference

Use the exact CSS class names from the current `stack-page.html` in the repo. There is ONE color scheme -- consult the unified color table in Step 1.

The key classes are:

- **Page**: `.page-wrap` (padding 40px), `.page-date` (`'Courier New', Courier, monospace`, 10px, uppercase, letter-spacing 0.14em, color `#8b949e`, bottom margin 24px -- matches the `.sm-sys-label` style), `.artifact-frame` (border-radius 10px, transparent bg, no border)
- **Stack Map**: `.sm-wrap`, `.sm-sys-label`, `.sm-sys-tag`, `.sm-al`, `.sm-al-item`, `.sm-al-sw`, `.sm-fns-row`, `.sm-fn-card`, `.sm-eyebrow`, `.sm-fn-name`, `.sm-fn-skills`, `.sm-sk`, `.sm-sk.sm-sched`, `.sm-mid`, `.sm-mid-annotation`, `.sm-ann-title`, `.sm-ann-sub`, `.sm-divider`, `.sm-dbs-row`, `.sm-db-card`, `.sm-db-name`, `.sm-db-desc`
- **Quick Reference**: `.qr-wrap`, `.qr-fn-section`, `.qr-fn-eyebrow`, `.qr-fn-header`, `.qr-skill-row`, `.qr-skill-name`, `.qr-skill-text`, `.qr-skill-line`, `.qr-badge`, `.qr-b-sched`, `.qr-b-adhoc`, `.qr-b-event`

Do NOT deviate from these class names.

#### Shine Hover Effect

All cards (`.sm-fn-card`, `.sm-db-card`), the annotation box (`.sm-mid-annotation`), skill pills (`.sm-sk`), system tags (`.sm-sys-tag`), and QR badges (`.qr-badge`) have a diagonal light-sweep hover effect using `background-image` + `background-position` animation. The effect animates in on hover and snaps back instantly on mouse-leave (no exit transition). This prevents the appearance of "bleed" when moving between adjacent elements under `transform: scale()`.

**Card and annotation box implementation** (applies to `.sm-fn-card`, `.sm-db-card`, and `.sm-mid-annotation`):
- Default state includes: `background-image: linear-gradient(120deg, transparent 0%, transparent 40%, rgba(255,255,255,0.06) 47%, rgba(255,255,255,0.13) 50%, rgba(255,255,255,0.06) 53%, transparent 60%, transparent 100%); background-size: 300% 100%; background-position: 100% 0; transition: border-color 0.3s ease;`
- Also include `overflow: hidden; isolation: isolate;` on each element. Cards additionally have `position: relative;` (the annotation box uses `position: absolute;` for centering).
- Hover state (`:hover`): `background-position: 0% 0; border-color: #3d444d; transition: background-position 0.8s ease, border-color 0.3s ease;`

**System tag implementation** (`.sm-sys-tag` -- the "Claude" and "Notion" chiclets):
- Same white-highlight gradient as cards: `linear-gradient(120deg, transparent 0%, transparent 40%, rgba(255,255,255,0.06) 47%, rgba(255,255,255,0.13) 50%, rgba(255,255,255,0.06) 53%, transparent 60%, transparent 100%)`
- Default: `background-size: 300% 100%; background-position: 100% 0; transition: none; overflow: hidden; isolation: isolate;`
- Hover: `background-position: 0% 0; border-color: #3d444d; transition: background-position 0.8s ease;`

**Skill pill implementation** (`.sm-sk` -- ad hoc pills with dark bg):
- Same white-highlight gradient as cards (see above). Default: `background-size: 300% 100%; background-position: 100% 0; transition: none; isolation: isolate;`
- Hover: `background-position: 0% 0; border-color: #3d444d; transition: background-position 0.8s ease;`

**Scheduled skill pill implementation** (`.sm-sk.sm-sched` -- white bg pills):
- **Inverted** gradient using dark tones so the sweep is visible against the light background: `linear-gradient(120deg, transparent 0%, transparent 35%, rgba(0,0,0,0.06) 44%, rgba(0,0,0,0.12) 50%, rgba(0,0,0,0.06) 56%, transparent 65%, transparent 100%)`
- Default: `background-size: 300% 100%; background-position: 100% 0;`
- Hover: `background-position: 0% 0; border-color: #a8a7a2; transition: background-position 0.8s ease;`

**QR badge implementation** (`.qr-badge`):
- Default state includes: `background-size: 300% 100%; background-position: 100% 0; transition: none;`
- `.qr-b-sched` uses **inverted** (dark) gradient: `linear-gradient(120deg, transparent 0%, transparent 35%, rgba(0,0,0,0.08) 44%, rgba(0,0,0,0.16) 50%, rgba(0,0,0,0.08) 56%, transparent 65%, transparent 100%)`. Hover also sets `border-color: #a8a7a2`.
- `.qr-b-adhoc` adds gradient: `linear-gradient(120deg, transparent 0%, transparent 35%, rgba(255,255,255,0.12) 47%, rgba(255,255,255,0.22) 50%, rgba(255,255,255,0.12) 53%, transparent 65%, transparent 100%)`
- `.qr-b-event` uses **inverted** (dark) gradient (same pattern as `.qr-b-sched`) since the light mint background needs dark tones to show the sweep: `linear-gradient(120deg, transparent 0%, transparent 35%, rgba(0,0,0,0.06) 44%, rgba(0,0,0,0.12) 50%, rgba(0,0,0,0.06) 56%, transparent 65%, transparent 100%)`. Hover sets `border-color: #1D9E75` (deeper green for contrast on hover).
- Hover state (`:hover`): `background-position: 0% 0; transition: background-position 0.8s ease;`

**Key design rule**: The `background-position` transition is ONLY set on the `:hover` rule. The default (non-hover) state has NO `background-position` transition, causing the shine to reset instantly when the cursor leaves. This is intentional -- it prevents two adjacent elements from showing shine simultaneously during the exit animation when moving between them.

**Inverted shine rule**: Any element with a light/white background (`#f0efea`) uses `rgba(0,0,0,...)` gradients instead of `rgba(255,255,255,...)`. This ensures the sweep reads as a subtle dark shadow passing across the light surface rather than being invisible white-on-white.

---

## Step 5: Build Changelog

Maintain a running `CHANGELOG.md` in the `t-seaux/invertedcap-stack` repo that records what changed each week. This gives Tom a cumulative record of platform evolution without having to dig through commit history.

1. Fetch the existing `CHANGELOG.md` from the repo using `mcp__github__get_file_contents`. If it doesn't exist yet, initialize it with a `# Changelog` header and a one-line description: "All notable changes to the Inverted Cap Stack platform are documented here."
2. Prepend a new entry at the top (below the header) using the diff data from Step 0:

```markdown
## [DATE] (Week of YYYY-MM-DD)

**Added:** skill-a, skill-b (or "None")
**Removed:** skill-x (or "None")
**Modified:**
- skill-y -- [1-2 phrase summary of what changed, e.g. "added scheduled mode; previously ad hoc only"]
- skill-z -- [summary]
(or "None" if no modifications)
**Total skills:** [visible count only]
**Functions:** [list any function membership changes, or "No changes"]
```

3. Every field must be populated -- use "None" when there are no changes in that category. Even if no skills changed (i.e., the run was triggered manually or the no-change gate in Step 0 was bypassed), still log an entry noting "No changes detected" so the changelog doubles as an execution log.
4. **Modified-skill summaries — what to write**: Each modified-skill bullet should describe the *behavior* change in plain language. Good summaries: "added DocSend handling to materials pipeline", "tightened guardrail around hidden-category leakage", "switched from manual to scheduled cadence". Bad summaries: "updated SKILL.md", "various changes", or pasted diff fragments. If a modification is purely cosmetic (whitespace, typo, comment rewording), omit it from the changelog -- the modified-skill list should reflect changes that matter to a reader auditing platform evolution.
5. **Redaction rule**: The changelog is pushed to a public GitHub repo. Before writing any entry, scrub the following from skill names, summaries, and any other field:
   - Hidden category names, hidden skill names, or any reference to the existence of hidden categories (the changelog must read as if hidden skills do not exist; "Total skills" reflects visible skills only).
   - Personal names of contacts, founders, investors, or any third parties (e.g., "added handling for Lupe Hernandez confirmations" → "added handling for office cleaner confirmations").
   - Personal email addresses, phone numbers, account numbers, financial identifiers, dollar amounts tied to specific people, and physical addresses.
   - API keys, OAuth tokens, webhook URLs, Apps Script endpoint URLs, internal Notion database IDs, or any other credential or internal identifier.
   - Specific portfolio company names tied to non-public deal status (e.g., do not write "added pass-note logic for [CompanyX]"). Refer generically: "added pass-note logic for declined opportunities".
   - Any internal-only workflow detail Tom would not want a stranger to read. When in doubt, generalize.
6. Include `CHANGELOG.md` in the GitHub push (Step 6).

---

## Step 6: Push to GitHub

Push the following to the `t-seaux/invertedcap-stack` repository (main branch) in a single commit:

1. `stack-page.html` -- The responsive HTML file
2. `stack-map.png` -- The Platform Map PNG
3. `quick-ref.png` -- The Quick Reference PNG
4. `CHANGELOG.md` -- The running changelog
5. `skill-snapshots/<skill-name>.md` -- One file per visible skill, containing the current SKILL.md content. These are the diff baseline for the next run's modification detection (Step 0). Push only the snapshots that changed plus any new skills' snapshots; do not re-push unchanged snapshots.

Use the `mcp__github__push_files` tool to push all files in a single commit. Commit message format:

```
Update skill map visuals -- [DATE]

[Added/Removed/Modified skill summary -- apply the same redaction rules as the changelog (Step 5.5). Generic descriptions only, no personal names, no portfolio company names, no internal identifiers.]
```

---

## Step 7: Send Alert

Use the `send-alert` skill to notify Tom via Signal Note to Self:

```
🗺️ Skill Map Refresh

Updated Platform Map and Quick Reference.
[Summary of changes: e.g. "+1 skill (backchannel-drafter) in Diligence Management"]
Stack page pushed to GitHub.
```

If running from `run-all`, suppress this alert (the orchestrator handles consolidated notifications).

---

## Error Handling

- **GitHub push fails**: Retry once. If still failing, save the HTML file locally and alert Tom to push manually.
- **PNG render fails**: Skip PNG update but still push HTML. Note the failure in the alert.
- **No changes detected**: Report cleanly and exit — do not regenerate or push stale content.
- **New skill doesn't fit a function**: Flag it in the alert for Tom to assign manually. Do not add it to the visuals.

---

## Scheduled Task Configuration

This skill is invoked by the `com.tomseo.scheduled.skill-map-refresh` local LaunchAgent:

- **Cadence**: Daily at 7:30 AM local time (`StartCalendarInterval` with just `Hour=7 Minute=30` — no `Weekday` key)
- **LaunchAgent plist**: `~/Library/LaunchAgents/com.tomseo.scheduled.skill-map-refresh.plist`
- **Runner**: `/Users/tomseo/.claude/scheduled-tasks/skill-map-refresh/run.sh` — invokes this skill file.
- **No-change gate**: Step 0 exits cleanly without a push when nothing has changed, so daily runs stay cheap on no-op days.

The LaunchAgent does NOT contain implementation logic -- it simply invokes this skill file.
