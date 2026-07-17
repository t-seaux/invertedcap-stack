---
name: decision-retro
description: >-
  Capture Tom's free-form retro on any invest/pass decision, including cold dismissals — founder signals, stage,
  market, model, positioning, valuation, anything Tom has texture on. Raw retro stays on the Opportunity page;
  nuggets accumulate in a thematic master log. Three paths: (1) Scheduled scan (daily 9am ET) detects Opp status
  to Committed / Pass Note Pending / Pass (Met) / Pass (DNM) and prompts in #decision-retros Slack; a 6pm
  listener processes replies. (2) Capture mode (manual): "retro on [company]", "log retro on X", "decision retro
  on X", "log this retro". (3) Lookup mode (manual): past-tense questions — "why did I pass on [X]", "why did I
  invest in [X]", "what did I think of [X]", "pull my retro on [X]", "my take on [X]" — returns a structured
  summary, never the raw retro verbatim. Distinct from first-pass-diligence (before the decision) and
  pass-note-drafter (outbound founder comms).
---

# Decision Retro

Captures Tom's post-hoc reasoning on every invest/pass decision and routes the raw retro + extracted nuggets into the feedback corpus that drives framework evolution (per founder-taste/RUBRIC.md §8.6 + SYSTEM.md §14.1).

## Scope

Every Opportunity transition into one of these terminal states triggers a retro prompt:
- **Invest side**: `Committed` (the moment of decision; `Active Portfolio` is post-close operational state — no separate retro)
- **Pass side**: `Pass Note Pending`, `Pass (Met)`, `Pass (DNM)` (includes cold dismissals — yes, one-sentence retros are fine)

Follow-on Opps (`(FO)` / `(Series X FO)` in title) are NOT skipped — each FO is a separate Notion row with its own ID, treated as a separate retro entry by the queue dedup. Each follow-on round is a real new decision; longitudinal retros across rounds are valuable calibration data.

The four trigger statuses act as a priority cascade: the queue dedups by Opp ID, so each Opp gets one retro at its earliest terminal-status transition (e.g., `Pass Note Pending` fires first; subsequent `Pass (Met)` is skipped because the Opp is already in the queue). The other statuses serve as backstops if an Opp ever bypasses an earlier state.

Manual trigger works the same way, independent of Status.

## Architecture (as of 2026-04-23)

**State is headless** — lives in `~/.claude/skills/decision-retro/queue.json`, not in Notion. No `Retro Captured` property or similar on the Opportunity page. This keeps the Opp schema clean and makes the retro flow easy to rip out or rewire.

Two scheduled agents:

- **`decision-retro-scan`** (daily 9am ET) — queries Notion for Opps in terminal status, dedups against `queue.json`, posts new prompts to `#decision-retros` as the `claude` Slack app with fingerprint `[opp:<short>]`, records each prompt's thread `ts` in the queue with `scope="opp"`.
- ~~`neg1-retro-scan`~~ — RETIRED 2026-07-16 (plist in `_disabled-plists/`). The -1 Scanner DB is deleted; -1 pass/draft reasons are captured at decision time in the `#neg1-sourcing` card thread by neg1-sourcing-listener. Historic `scope="neg1"` queue entries remain valid for lookups.
- **`decision-retro-listener`** (daily 6pm ET) — for each `prompted` item in the queue, reads the Slack thread, handles both `[opp:]` and `[neg1:]` fingerprints, treats concatenated Tom replies as the retro, runs the extractor (founder-signal-weighted for `scope="neg1"`), logs to source page + `DECISION_RETROS.md`, marks `completed`. Items prompted >7 days ago with no reply auto-close as `no_retro`. `skip`/`ignore`/etc. replies mark as `skipped` and are excluded from framework feed.
- **`retro-weekly-summary`** (Fri 4pm ET) — rolls up the week, groups by `scope` (Company retros / Founder-signal retros), posts synthesis to `#decision-retros`.

Helper: `send-alert/md_to_blocks.py` posts to the channel webhook at `~/.claude/skills/decision-retro/.webhook_url` (distinct from the DM webhook used by `send-alert` itself).

## Queue state (`queue.json`)

```json
{
  "items": [
    {
      "opp_id": "notion-page-uuid",
      "opp_name": "Acme Corp",
      "opp_url": "https://notion.so/...",
      "decision": "Pass",
      "scope": "opp",                 // "opp" (Opportunities DB) | "neg1" (-1 Scanner DB). Absent = "opp" for legacy entries.
      "short_id": "abc12345",
      "status": "prompted",           // "prompted" | "completed" | "skipped" | "no_retro"
      "thread_ts": "1745432123.456789",
      "prompted_at": "2026-04-23T09:00:00-07:00",
      "completed_at": null,
      "nugget_count": null,
      "would_back_again": null,             // "yes" | "open" | "no" | "profile_mismatch" | "n/a" | null (pre-completion). FYI annotation only — does NOT enter scoring. "n/a" for first-time cold dismissals where the question doesn't apply.
      "would_back_again_rationale": null    // one-line, populated alongside would_back_again
    }
  ]
}
```

Queue grows over time (acceptable — small file). No pruning required.

## Workflow

### Step 1 — Scan + prompt (9am ET)

1. Load `queue.json` (create if absent).
2. Query Notion Opportunities DB for rows where Status ∈ {`Committed`, `Pass Note Pending`, `Pass (Met)`, `Pass (DNM)`}. Include follow-on rows (titles with `(FO)` / `(Series X FO)`) — they are NOT skipped; each FO is a separate Opp ID treated as a separate retro entry. For each row capture `Close Date`, `Website`, `Description`, and `🏁 Founder(s)` alongside `id/name/url/status`. Resolve each Founder relation → People page for name + `LI` URL.
2a. **Freshness gate (code-enforced).** All new queue entries — sweep AND webhook — MUST be appended via `queue_append.py`, never by hand-editing `queue.json`. The script enforces the gate at write time:
   - Sweep callers (no `--trigger-source`): reject any Opp with `Close Date` empty/null OR more than 60 days before today (the sweep is a reconciliation backstop for recent transitions the notion-webhook missed, NOT a back-catalog mining tool — stale historical passes must not surface as retro prompts).
   - Webhook callers (`--trigger-source webhook:status-change` or similar): bypass the gate — they fire on actual transition events and are by definition fresh.
   - Manual `retro on X` invocations: bypass via `--trigger-source manual`.
   - Duplicates (opp_id already in queue with non-terminal status): rejected.
   - Usage: `python3 ~/.claude/skills/decision-retro/queue_append.py --opp-id <id> --opp-name <name> --opp-url <url> --decision <status> --close-date <YYYY-MM-DD> [--trigger-source <src>] [--thread-ts <ts>]`. Exit 0 = appended (proceed); exit 1 = skipped (don't post to Slack); stdout JSON explains the action.
   - Background: Scout (`30c00bef`, Close Date 2026-03-10, 93d old) leaked through on 2026-06-11 because the gate was prose-only. The script makes the bypass impossible.
3. For each Opp whose `id` is NOT in the queue (the dedup + gate run inside `queue_append.py` at Step 3d below — don't second-guess them here):
   - Compose GFM markdown prompt (`[text](url)` for links, `**text**` for bold). `md_to_blocks.py` converts to Slack Block Kit — do NOT hand-write Slack mrkdwn (`<url|text>` / `*text*`), it ships as literal text / renders italic. Per memory `feedback_scheduled_alert_structure.md`:
     ```
     🏢 <u>**{Opp Name} | [Notion]({Notion url})**</u>
     **Status:** {Status}
     `[opp:<short-opp-id>]`
     ```
     Rules:
     - Compact format (no bullets, locked in 2026-04-27): entity line + status line + fingerprint, no Description / Founders / Website fetch. Tom clicks through to the Notion page when he needs context.
     - Both lines are bare paragraphs (no `- ` prefix). The first character is the emoji (OUTSIDE the wrappers — Tom does not want the emoji underlined); the rest of the entity is wrapped in `<u>**...**</u>` so it renders bold AND underlined with the Notion link live.
     - `md_to_blocks.py` treats `<u>` and `**` as recursive wrappers, so bold + underline + inner link compose correctly.
     - The fingerprint sits immediately under the Status line (no blank line), wrapped in backticks for inline-code-styled rendering. **No blank lines anywhere in the body** — blank lines emit a `\n\n` spacer.
     - **Never include a "Reply in this thread…" line** — the channel description already says this
   3a. POST via `send-alert/md_to_blocks.py` with `WEBHOOK_URL=$(cat ~/.claude/skills/decision-retro/.webhook_url)`.
   3b. Immediately call `mcp__claude_ai_Slack__slack_read_channel` on `#decision-retros` (limit 10) and find the message by grepping for the `[opp:<short-opp-id>]` fingerprint. Capture `ts`.
   3c. **DO NOT hand-edit `queue.json`.** Call `queue_append.py` with the fields above. The script enforces the freshness gate + dedup at write time; the gate is the only thing standing between sweep runs and back-catalog leakage like the Scout 2026-06-11 incident.
   3d. If `queue_append.py` exits 1 (skipped / duplicate), the prompt should NOT have been posted in the first place — abort, log the skip reason, and **do NOT post to Slack**. Reorder if needed so the gate runs before the Slack post on subsequent runs.

### Step 2 — Listen + extract + log (6pm ET)

For each `items[]` entry where `status == "prompted"`:

1. Check age: if `prompted_at > 7 days ago`, set `status="no_retro"`, continue (no processing).
2. Call `mcp__claude_ai_Slack__slack_read_thread` with the stored `thread_ts` on `#decision-retros`.
3. Filter replies to those from Tom (user ID check). If no replies, continue (leave in queue for next run).
4. Concatenate Tom's replies in thread order → raw retro text.
4a. **Skip check**: if the concatenated reply (trimmed, lowercased, punctuation stripped) matches `^(ignore|skip|pass|n/?a|no thanks|nope|nah|nvm|not today)$`, treat as an explicit skip. Set queue item `status="skipped"`, `completed_at=now()`, `nugget_count=0`. Do NOT run extraction, do NOT write to Opp page, do NOT append to `DECISION_RETROS.md`. Send no Slack alert for skips. Continue to next queue item.
5. **Extract**: Claude call with prompt:
   ```
   Given this raw retro on [Opp Name] ([Decision]) and the context below,
   extract nuggets keyed by framework dimension AND infer the
   would_back_again disposition (if applicable — see rules below).
   Return ONLY valid JSON.

   Context: [...Opp page snippet, prior first-pass if any...]

   Raw retro: [...]

   Output shape:
   {
     "founder_signal": [...nuggets...],
     "market": [...],
     "biz_model": [...],
     "positioning": [...],
     "valuation": [...],
     "other": [...],
     "would_back_again": "yes" | "open" | "no" | "profile_mismatch" | "n/a",
     "would_back_again_rationale": "<one-line, verbatim where possible>"
   }

   Rules:
   - Preserve Tom's exact phrasing where possible; empty arrays for dimensions
     Tom didn't touch; don't invent dimensions not in the schema.
   - `would_back_again` is **specifically about re-backing**, applicable only
     when Tom has texture on the founder from a prior decision. **For
     first-time cold dismissals or sourcing-only opps where Tom never met
     the founder, use "n/a" — the question doesn't apply because there's
     no "again."** The field also lands as "n/a" if the retro is purely
     about market/category/biz-model and never touches the founder.
   - Bands (FYI annotation only — does NOT enter scoring):
     - "yes": Tom would back this founder's next thing without hesitation
       (e.g., "absolutely back whatever this guy does next").
     - "open": depends on what the founder does / how they evolve
       (e.g., "open question whether I'd back a founder of this profile").
     - "no": clean no — wouldn't back even if they came back.
     - "profile_mismatch": love the founder, but they've graduated past Tom's
       fund product (e.g., post-exit founder raising $30M+ rounds — see
       Casebook cross-synthesis #21 / Randy archetype).
     - "n/a": question doesn't apply (cold first pass / no founder relationship /
       retro didn't address founder-level future-backing).
   - When in doubt between "open" and "n/a": if Tom has met the founder
     and the retro implies *any* re-back disposition (even "I'd need to see X"),
     use "open". Reserve "n/a" for cases where the founder is genuinely
     a stranger or the retro is non-founder-focused.
   ```
6. **Log to Opp page** — append a `## Retro (YYYY-MM-DD)` block with the raw reply text verbatim. Below the raw text, append a single-line annotation when applicable: `**Would back again:** <yes|open|no|profile_mismatch> — <one-line rationale>`. **Skip the annotation entirely if `would_back_again == "n/a"`** (first-time cold pass / no founder relationship — the line would be noise).
7. **Log to `DECISION_RETROS.md`** — append to each relevant thematic section:
   - `## Founder signals` — from `founder_signal` nuggets
   - `## Market` — from `market`
   - `## Company` — from anything not slotted elsewhere, plus `other`
   - `## Business model` — from `biz_model`
   - `## Positioning / moat` — from `positioning`
   - `## Valuation & terms` — from `valuation`

   Entry format:
   ```
   - **YYYY-MM-DD · [Company] · [Invested/Pass]** — [nugget verbatim or lightly paraphrased]
     - Source: [Opportunity page URL]
   ```
7a. **Log to the decision ledger** (`~/.claude/data/decision_ledger.db`) — one structured row per decision so the rubric can be back-tested against Tom's actual calls:
   - **Decision mapping**: `Committed` → `invested`; `Pass (Met)` / `Pass Note Pending` → `pass-met`; `Pass (DNM)` → `pass-dnm`. For `scope="neg1"`: `Outreach` → `reached-out`; `Passed` → `no-outreach`.
   - **Signal scores**: for `scope="neg1"` rows, parse from the row's `Signals` compact line `NL:{n} · Reps:{n} · Rigor:{n|U} · Ant:{n} · Int:{n} · Rng:{n} · rec:{✅|🤔|❌}` (0-10 numbers; U = unobservable; legacy rows may carry H/M/L letters; `rec` = the PRE-gate auto-rec) and map `Claude Rec` → `--rubric-verdict reach-out|pass`. For `scope="opp"` rows, do a **quick retro-time 6-signal read** from the Opp page context already fetched in Step 5 (founder background, call notes, first-pass block): H/M/L per signal with the founder-taste/RUBRIC.md §5 anchors, omitting signals with no evidence. This is a coarse read, NOT an enrichment — never call ContactOut or run web research here. Omit `--rubric-verdict` for opp rows (no auto-rec existed at decision time).
   - **Call**:
     ```bash
     python3 ~/.claude/scripts/decision-ledger/append_decision.py \
       --label "{Opp/person name}" --decision {mapped} --date {today} \
       --source {pipeline|-1 scanner} --verdict-raw "{raw status}" \
       [--scores '{"Non-Linearity": "High", ...}'] [--rubric-verdict reach-out|pass] \
       --rubric-version {status version from founder-outreach/founder-taste/RUBRIC.md frontmatter} \
       --why "{nuggets joined with ' | '}" --retro-ref "{Opp/row URL}"
     ```
   - The script upserts on (canonical name, decision) — safe to re-run; a Task 7 `reached-out` row for the same person stays separate.
   - **Non-fatal**: if the script errors, log and continue — the ledger append must never block retro capture.

8. Mark queue item `status="completed"`, `completed_at=now()`, `nugget_count=<total>`, `would_back_again=<band>`, `would_back_again_rationale=<one-line>`. The `would_back_again` field on the queue item makes the disposition queryable for future Lookup-mode calls without re-parsing the Opp page.
9. Single-line alert via `send-alert` to DM: `🔁 Retro captured for [Company] ([N] nuggets across [sections])` — append ` — would back: [band]` only when `would_back_again` is not `"n/a"`.

### Capture mode (manual)

When Tom says "retro on X", "log retro on X", or any imperative indicating he's providing new retro text:
1. Locate the Opp in Notion.
2. Ask him for the retro text inline (or use whatever he just said as the retro).
3. Run Steps 2.5–2.9 directly (extract, log, report).
4. If the Opp was already in the queue (`prompted`), update to `completed` so the scheduled listener skips it.

### Webhook-prompt mode (single-page, immediate)

Triggered by the `claude-job-queue` processor with args `{ mode: "webhook-prompt", scope: "neg1"|"opp", neg1_page_id|opp_page_id, trigger_event }`. Fires when an event upstream (currently: Gmail draft-trash detected by `gmail-webhook`) flips a row to a terminal status and wants an immediate retro prompt — ahead of the next scheduled scan.

**Why it exists:** the 9am (`decision-retro-scan`) and 9:05am (`neg1-retro-scan`) scheduled agents are reconciliation — they catch anything that transitioned to terminal status since the last run. For event-driven terminal transitions (draft trash, future webhook sources), latency to prompt is the signal: the retro is most useful captured while the decision is still fresh. This mode posts the prompt immediately instead of waiting up to 24h.

**Behavior (per invocation):**

1. Load `queue.json`. If an entry with matching `{scope, neg1_page_id}` already exists with `status ∈ {"prompted", "completed", "skipped"}`, exit silently (dedup — the scheduled scan may have beaten us, or this is a retry). Log reason.
2. Fetch the target page via `mcp__claude_ai_Notion__notion-fetch`. The target is the Opp or -1 page — no transcript content exists, so do NOT pass `include_transcript: true` here. **Exception for Lookup mode:** if the Lookup ever fetches a linked Notes-DB page that IS a meeting note (e.g., a specific call note Tom asks to revisit), pass `include_transcript: true` on THAT fetch only — the workflow is then summarizing call content.
3. Compose the Slack prompt per scope. For `scope="neg1"` use this template exactly (GFM markdown — `[text](url)` for links, `**text**` for bold). `md_to_blocks.py` converts to Slack Block Kit; Slack mrkdwn (`<url|text>`, `*text*`) ships as literal text / renders italic:

   ```
   - 🧍 <u>**{Name} | [Notion]({Notion page url}) | [LinkedIn]({LI url})**</u>
   - **Status:** {Status}
   `[neg1:<short-id>]`
   ```

   Rules:
   - `<short-id>` = first 8 chars of the Notion page UUID
   - Compact format identical in shape to the Opp version (locked in 2026-04-25). Both lines are bullet items.
   - Emoji OUTSIDE the `<u>**...**</u>` wrapping; name + Notion link + LinkedIn link are all bold AND underlined with both links live.
   - **LinkedIn segment only appears when the `LI` property is populated.** If `LI` is empty/missing, drop the trailing ` | [LinkedIn](...)` — never emit a broken or placeholder link.
   - **No Current / Work History / Education / standalone LI line** — Tom clicks through to the Notion page when he needs the rest of the resume context.
   - Status bullet is just the Status value — no trailing "flipped {date}" clause.
   - **No blank lines anywhere in the body** — fingerprint sits immediately under Status, in backticks.

   For `scope="opp"` use the Opp format from Step 1 of the 9am scan (lines 82–96 above).
4. POST via `send-alert/md_to_blocks.py` using the `#decision-retros` webhook URL at `~/.claude/skills/decision-retro/.webhook_url`.
5. Immediately `slack_read_channel` on `#decision-retros` (limit 10) to find the message by fingerprint; capture `ts`.
6. Append a new entry to `queue.json` with `status="prompted"`, `thread_ts=ts`, `prompted_at=now()`, `scope`, page id, page name, page url, decision. Include a `trigger_source: "webhook:<trigger_event>"` field so the analytics layer can distinguish webhook-triggered from schedule-triggered prompts.
7. **Append the audit-log line** (per SHARED_SAFETY.md). Use a single Bash call with `mkdir -p` + `tee -a` to `~/.claude/scheduled-tasks/decision-retro/audit-log/YYYY-MM-DD.log`:
   ```bash
   mkdir -p ~/.claude/scheduled-tasks/decision-retro/audit-log && \
   echo "[$(date '+%Y-%m-%d %H:%M:%S')] WRITE: slack #decision-retros posted retro prompt opp:<SHORT_ID> name:<NAME> status:<STATUS> ts:<thread_ts>" \
     | tee -a ~/.claude/scheduled-tasks/decision-retro/audit-log/$(date +%Y-%m-%d).log
   ```
   This is a concrete shell action, not a Write tool call — runs cleanly under `--dangerously-skip-permissions`. Do NOT report "audit log blocked" in the job summary; if `tee` exits non-zero, that's the only condition that warrants a warning.
8. The existing `decision-retro-listener` (6pm ET) will process the reply when Tom threads a retro — no change to that path.

**Idempotency:** the upstream webhook must pass `idempotencyKey = "draft-trash-retro-{messageId}"` (or analogous) so `claude-job-queue` dedupes re-triggers. Plus the step-1 queue check inside this skill is a second layer against the scheduled scan firing first.

**Failure:** if the page is no longer in terminal status by the time the job runs (Tom manually flipped it back), exit silently — log reason. Never post a prompt for a non-terminal row.

### Lookup mode (manual)

When Tom asks a past-tense question about a prior decision — "why did I pass on X", "why did I invest in X", "what did I think of X", "pull my retro on X", "show me my feedback on X", "my take on X" — return a structured summary, never the raw retro verbatim.

**Target resolution:**
- If the question names a **company** ("why did I pass on Dorsal"): search the Opportunities DB (`collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`). Use the earliest-fund original investment if multiple matches (e.g. `Rengo` + `Rengo (Seed FO)`) — follow-on rows (`(FO)`, `(Seed FO)`, `(A FO)`, `(SPV)`) deprioritize. Use `scope="opp"`.
- If the question names a **person** ("why did I pass on Elsie Kenyon", "what did I think of [person]", especially when paired with `-1` or a LinkedIn URL): query the CANDIDATE STORE (`python3 ~/.claude/scripts/decision-ledger/candidates.py get --name "<name>"`) + the decision ledger (`SELECT * FROM decisions WHERE label LIKE ...`) + the archive (`~/.claude/data/neg1_scanner_archive.json` for pre-2026-07-16 rows). The -1 Scanner DB is deleted. For -1 targets the output structure changes slightly — see "-1 Lookup variant" below (Eval context now reads from the store fields: signals_line, rec, when_gate).
- If no match, say so and stop.

**Data sources (in priority order):**
1. **Official pass note** — the outbound email archived by `pass-note-drafter`. Locate via the Opp's `✍️ Notes` relation → filter to entries with `Category = Diligence` whose body contains the pass note text. If not in Notes, fall back to Gmail search for sent mail to the founder around the Close Date. Summarize (do not paste verbatim).
2. **Internal feedback** — the Opp page's `## Retro (YYYY-MM-DD)` block(s), plus any matching entries in `~/.claude/skills/founder-taste/DECISION_RETROS.md` sourced to this Opp URL. Distill into dimension-keyed summaries (no raw text quoted).
3. **Diligence context** — the Opp page's first-pass diligence block and/or pre-mortem block, or the linked first-pass page in `✍️ Notes`. One-line summary only.
4. **Framework** — `~/.claude/skills/founder-taste/CASEBOOK.md` + `founder-taste/RUBRIC.md`. Match nuggets to named patterns.

**Output structure** (inline chat response, not Slack):

```
**[Company Name]** — [Decision] · Close Date [YYYY-MM-DD] · Retro logged [YYYY-MM-DD]

**Official pass note feedback** (if applicable)
• [1–3 bullets summarizing what Tom told the founder — tone + substance]

**Internal feedback**
• *Founder signal:* [summary, no verbatim]
• *Market:* [summary]
• *Business model:* [summary]
• *Positioning:* [summary]
• *Valuation:* [summary]
(Omit dimensions with no nuggets. Use bare `~$3-4B`, en dashes only.)

**Framework connections**
• Reinforces: [pattern name from founder-taste/CASEBOOK.md] — [one-line why]
• Challenges: [pattern] — [why]
• Expands: [new pattern hint] — [why]

**Diligence context**
• [One-line summary of pre-decision thinking — first-pass priors or pre-mortem headline]

[Notion](url)
```

**Edge cases:**
- **Queue status `prompted` (reply not yet processed)**: don't wait for the 6pm listener. Do an **inline one-off pass** for this single Opp right now:
  1. Read the Slack thread via `mcp__claude_ai_Slack__slack_read_thread` using the stored `thread_ts`.
  2. Filter to Tom's replies, concatenate → raw retro text.
  3. Run Step 4a skip-check (from the scheduled listener logic above). If it matches skip, mark queue item `status="skipped"` and treat as the skipped edge case below.
  4. Otherwise run Steps 5–8 from the listener flow: extract nuggets via Claude, append `## Retro (YYYY-MM-DD)` to the Opp page, append to `DECISION_RETROS.md`, mark queue item `status="completed"` / `completed_at=now()` / `nugget_count=<total>`.
  5. Use the freshly extracted nuggets in the Lookup output. Tell Tom inline: "Processed your Slack reply just now — logged and summarized below."
  6. If the thread has zero replies from Tom, emit the "No reply yet" fallback: header + "No reply logged in Slack yet — thread is here: <slack link to thread>" + whatever Official pass note + Diligence context is available.
- **No retro logged** (`queue.json` has no entry for this Opp at all): emit the header + "No retro logged" in place of Internal feedback, but still surface Official pass note feedback + Diligence context if available.
- **Queue status `no_retro`** (7-day window closed with no reply): same as "No retro logged" — note that no reply came in before the window closed.
- **Queue status `skipped`**: emit the header + "No internal feedback (marked skipped)" in place of Internal feedback. Still surface Official pass note + Diligence context.
- **Invest decisions**: omit "Official pass note feedback" section entirely. Internal feedback + Framework connections + Diligence context still apply.

Lookup is normally read-only. The only write path is the **inline one-off pass** above, which is functionally equivalent to the 6pm listener running early on that single Opp — not a new write path, just timing acceleration.

**-1 Lookup variant** (for `scope="neg1"` targets):

Output structure is adjusted for a person, not a company:

```
🧍 **[Name](LI URL)** — -1: [Passed|Outreach] · Flipped [YYYY-MM-DD] · Retro logged [YYYY-MM-DD]

**Current**
• [Role] @ [Company]

**Work History** (one-line summary, 2–4 bullets max — lean on Experience Summary on the row)
• [Company] ([years], key signal)
• ...

**Internal feedback**
• *Founder signal:* [summary, no verbatim] — this dimension dominates for -1 retros
• *Market / biz model / positioning:* [only if Tom's retro touched these]

**Framework connections**
• Reinforces: [signal pattern] — [one-line why]
• Challenges / Expands: [...]

**Eval context** (replaces "Diligence context" for -1s)
• Eval Score (Spike): [0–1]
• Signals flagged at enrichment: [list from the Signals multi-select]
• Claude Rec at enrichment: [Reach Out / Pass]

[Notion](url)
```

Omit "Official pass note feedback" — -1 rows don't have outbound pass notes. Everything else (skip handling, no-retro handling, inline one-off pass, prompted-but-no-reply) works the same way.

## Founder retro DM format (compact embedded-link)

For founder retro DMs (Inverted 1, Dash 1+2, future fund retros) and similar minimal-format Slack posts that pair a name with a LinkedIn URL, use the compact embedded-link format — NOT a separate `LinkedIn:` line:

```
**<Retro Type>: [<Name>](<LinkedIn URL>) – <Company><optional ` (Status)`>**
`[founder:<short-id>]`
```

Two lines, no blank line between them. Examples:

```
**Dash 2 Retro: [Sean Monteiro](https://www.linkedin.com/in/sean-monteiro) – Bounce**
`[founder:d12-mont]`
```

Rules:
- En dash (`–`) between Name and Company per Tom's voice rules.
- Append status in parens only for non-Active states (e.g., `(Exited)`, `(Wound Down)`).
- No blank line between title and fingerprint (blank lines render as `\n\n` spacers via `md_to_blocks.py`).
- Does NOT apply to long-form alerts where context is the point — those follow the per-entity row convention in `send-alert/SKILL.md`.

## Important rules

- **Never impose a template** on the raw retro. The point is capturing what Tom wouldn't otherwise write down.
- **Extracted nuggets quote the raw retro where possible** — don't paraphrase away the texture.
- **Cold dismissal retros are fully valid**. A one-sentence "wrong GTM motion for this market" is a legitimate retro. Don't prompt for more.
- **Never prompt twice for the same Opportunity** — idempotency is enforced by queue.json `opp_id` dedup.
- **Never match queue items by `short_id` alone** — Notion 8-char prefixes collide more often than feels intuitive (e.g. Comvex `35100bef-f4aa-8162-...` and Atina `35100bef-f4aa-816d-...` share `35100bef`). When deriving short_id from a Slack fingerprint in `decision-retro-listener` Mode A or B, narrow further before writing — disambiguate by `prompted_at` proximity to `thread_ts`, or by `status == "prompted"` (only one item per short_id should be open at a time). Always update queue rows by full `opp_id`. Same caution applies to any future skill that keys off short_id in this queue.
- **Promote to CALIBRATION manually.** This skill does NOT write to `founder-taste/CASEBOOK.md` directly. Patterns solidify into calibration through Tom's explicit curation call.
- **Status re-transitions are one-shot.** If Tom flips an Opp's status across terminal states (e.g., `Pass Note Pending` → `Pass (Met)`, or `Pass (Met)` → `Committed` — rare), the queue's existing `prompted` / `completed` entry stays — no re-prompt. The cascade fires once per Opp ID at the earliest terminal transition. Edge case; handle manually via Tom saying "retro on X" if he wants another pass.

## Consumption (downstream)

- `neg1-enricher` reads `DECISION_RETROS.md` alongside `founder-taste/CASEBOOK.md` during Step 5 (rubric application). Retros surface as soft priors in the Signals line.
- **Decision ledger** (`~/.claude/data/decision_ledger.db`, written by Step 7a + pipeline-agent Task 7 step 3c) joins the rubric's read at decision time to Tom's actual call. Override rows (`override IS NOT NULL`) are the highest-signal evidence of rubric drift — 3+ same-direction overrides in a quarter is promotion/demotion-grade evidence. Backfill/rebuild: `~/.claude/scripts/decision-ledger/backfill.py` (2026-07-16 backfill: 93 decisions).
- Quarterly `--score-only` drift check (per founder-taste/RUBRIC.md §8.7 + Future State item 14) now starts from ledger queries (rubric precision, override clustering by signal and direction) rather than a hand re-score; accumulated retros remain the qualitative evidence layer.

## Example

Tom flips `Acme Corp` → `Pass` on 2026-04-22. The 9am scan on 2026-04-23 posts to `#decision-retros`:

> 🏢 <u>**Acme Corp | [Notion](https://notion.so/tom/acme-...)**</u>
> **Status:** Pass
> `[opp:abc12345]`

Tom replies in thread (voice-to-text via Wispr Flow): "passed on Acme. founder was smart but the market is fundamentally consumer-subsidized B2B — anyone who's not paying for their employer's version will churn the day they change jobs. saw this exact pattern with X and Y."

6pm listener processes:
- Opp page gets `## Retro (2026-04-23)` block with the raw text.
- `DECISION_RETROS.md`:
  - `## Market` — `**2026-04-23 · Acme · Pass** — Consumer-subsidized B2B where churn is triggered by employer changes is a recurring trap pattern. Compare X and Y.`
  - `## Business model` — `**2026-04-23 · Acme · Pass** — Individual-pay-after-employer-pay GTM motion has hidden churn risk.`
- Slack DM: `🔁 Retro captured for Acme (2 nuggets across Market, Business model).`
