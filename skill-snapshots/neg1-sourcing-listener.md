---
name: neg1-sourcing-listener
description: >-
  Processes Tom's replies to candidate cards in #neg1-sourcing — the Slack go / no-go surface for -1 (pre-founder)
  sourcing. Verb grammar on card threads: "draft" (flip row to Draft Requested → notion-webhook fires founder-outreach,
  Gmail draft lands in ~2 min), "pass <why>" (Status Passed + decision-ledger row + reason logged as retro nuggets),
  "track" / "snooze" (sets Re-surface date, default one quarter; card re-posts when it arrives), "more" (posts the full Eval Summary +
  Breakdown into the thread). (A) Scheduled sweep — daily reconciliation over recent card threads, catches replies
  the webhook missed. (B) Webhook — invoked via claude-job-queue when the slack-retro-webhook Worker routes a
  #neg1-sourcing thread reply. Never sends email — drafting happens via the existing Draft Requested pathway, and
  Tom sends by hand from Gmail. Not user-facing in webhook mode. Distinct from decision-retro-listener
  (#decision-retros) and claude-alerts-listener (#claude-alerts).
---

# -1 Sourcing Listener

Turns `#neg1-sourcing` into the decision surface for pre-founder sourcing: one Slack reply = the decision + the automation trigger + the feedback capture. Notion stays the database; Tom never has to open it to make a call.

## Setup requirements (one-time — check before processing, exit gracefully if missing)

1. `#neg1-sourcing` channel exists (private, ID `C0BHT40PTEZ`, created 2026-07-16) and the `claude` Slack app is a member. ✓ DONE
2. ✓ DONE — Channel ID saved at `~/.claude/skills/neg1-sourcing/.sourcing_channel_id` (used by the card writers: pipeline-agent Task 6 step 2b + neg1-sourcing Step 4 — they post via md_to_blocks.py bot-token mode with `~/.claude/skills/claude-alerts-listener/.bot_token`; no incoming webhook needed).
3. Channel ID added to `SKILL_BY_CHANNEL` in the `slack-retro-webhook` Worker (`"<CHANNEL_ID>": "neg1-sourcing-listener"`) and the Worker redeployed. Until then only Mode A (sweep) fires.

## v2 HEADLESS FLOW (2026-07-16 — CURRENT)

Candidates live in the candidate store (`python3 ~/.claude/scripts/decision-ledger/candidates.py`), not the -1 Scanner. **The `draft` reply is the moment of CRM birth** — nothing exists in Notion before it. v2 verb behavior (the Notion-flip instructions below are RETIRED — DB deleted 2026-07-16; ALL cards resolve via the candidate store):

**Resolve (Step 1 v2):** card fingerprints are LinkedIn vanity slugs — `[neg1:{slug}]`. Resolve via `get --li https://linkedin.com/in/{slug}` (the store canonicalizes hosts — www./regional prefixes are equivalent); if empty, `get --name` from the card header; verify name matches. **Verb handling uses `set-state` ONLY — NEVER `upsert` (a verb on an unresolvable card posts an error reply; it must not mint a new row — caused the 2026-07-17 Jinsub duplicate).** Legacy short-id fingerprints (pre-2026-07-16 cards) resolve via the archive at ~/.claude/data/neg1_scanner_archive.json.

**draft [<why>] (v2):** anything after the verb is Tom's POSITIVE taste signal — why this person earned the email. Optional, but when present it is first-class feedback:
- Save it: `set-state --draft-why "<why verbatim>"`.
- Log a nugget to `~/.claude/skills/founder-taste/DECISION_RETROS.md` under `## Founder signals`: `- **YYYY-MM-DD · {Name} · -1: Drafted** — {why}` + Source line (LI URL). Positive texture is as framework-relevant as pass reasons — it is what calibrates what GOOD looks like between invest retros.
- Include it in the eval note (step 3 below) as a `**Why drafted (Tom):**` line.
Then execute:
1. Run `founder-outreach` in **store mode** (inline — read that skill's store-mode block): Gmail draft from the store row, `set-state --gmail-draft-url`.
2. **Create the Opportunity** — invoke `add-to-crm` with the canonical -1 pre-company field table (pipeline-agent Task 7 step 3b): Name `-1 [{Name}]({LI URL})`, Stage `Pre-Seed 💡`, Source Direct, Fund `Inverted 1️⃣`, Status `Outreach`, Description TBD, Contact = email, Website N/A, 🏁 Founder(s) = resolve/create People DB entry. This is when the person's employers get real Companies-DB treatment (add-to-crm handles it).
3. **Create the eval note (keep the Opp body clean — Tom's rule):** a new page in the Notes DB (the database behind the Opp's `✍️ Notes` relation), title `-1 {Name}: Founder Eval` (colon convention), Category `Diligence`, related to the new Opportunity. Body: the Signals line, Eval Summary, full Eval Rationale, and framework connections (which lens the spike sits in, When-gate note if any). The Opportunity body itself gets NOTHING from scoring.
4. `set-state --state drafted --notion-opp-url <opp url>`.
5. Close-loop reply: draft link + Opp link, one line each.

**pass <why> (v2):** `set-state --state passed` + ledger append (scores/rec from store fields, not Notion) + retro logging exactly as below (nuggets to DECISION_RETROS.md; the raw reply is quoted in the ledger `why` — there is no Notion page to hold a Retro block pre-draft, and none is needed). No decision-retro queue registration (no scanner row → the 9:05am scan won't prompt).

**Prefilter promotion (added 2026-07-20 — the top-of-funnel feedback loop):** after logging a pass, classify the reason:
- **Category-level** — Tom excludes a SHAPE, not just this person: "you should not be flagging X", "never show me Y", "stop surfacing Z-shaped people", or a reason that plainly generalizes ("Carta is a stale unicorn; not much signal for folks working there"). → Append a new rule (or extend an existing one, e.g. add a company to PF-3's stale-unicorn list) in `~/.claude/skills/founder-taste/PREFILTERS.md`, with the verbatim quote + date as Source. If the rule is code-expressible, mirror it in `neg1_sourcing.py` (EXCLUDE_ROLES / FUND_NAME_RE / STALE_UNICORNS) in the same change — doctrine-coupled. Then note it in the close-loop reply: `→ promoted to prefilter ({rule id}): {one-clause rule}. Veto by replying here.`
- **Individual-level** — the reason is about this person's particulars. → Retro nugget only (existing behavior), no rule.
When in doubt, nugget — don't rule. A wrongly-minted rule silently starves the funnel; a missed promotion just means Tom repeats himself once more.

**track / snooze (v2):** `set-state --state tracked --resurface {today + duration, default 3mo}`. No Notion.

**more (v2):** post `eval_summary` + `signals_line` from the store; offer the full `eval_rationale` on a second `more`.

---

## Verb grammar

Parsed from Tom's thread reply on a candidate card (case-insensitive, first token wins):

| Verb | Aliases | Action |
|---|---|---|
| `draft [<why>]` | ANY affirmative-pursuit word: yes, pursue, go, reach out, let's do it, send it, in, 👍 as text — the class is open, not a fixed list | Draft + CRM birth (v2 block above); optional `<why>` = positive taste signal, logged like pass reasons |
| `pass <why>` | no, skip | Status → `Passed` + ledger row + `<why>` logged as retro |
| `track <dur>` | snooze, watch, hold, later | Set `Re-surface` date (default 3mo — one quarter; accept `3mo`/`6mo`/`12mo`/`1y`) |
| `more` | details, breakdown | (Courtesy verb, not shown on the card footer) Post full Eval Summary + Signals line into the thread |

Cards carry NO verb legend (dropped 2026-07-16 — the grammar lives in the pinned channel legend); the only footer line is the `[neg1:<short-id>]` fingerprint. The listener parses free text regardless.

**Free-form replies are first-class — Tom will often voice-dump the rationale instead of leading with a verb.** Parse INTENT over the whole reply, not just the first token:
- Clear engage intent ("love this, the Wispr arc is exactly the reluctant-stretch pattern — let's reach out") → `draft`, with the ENTIRE reply as `draft_why`.
- Clear negative read → `pass`, with the entire reply as the reason. No verb word required — and calibrate GENEROUSLY toward pass here: a reply that is a sustained negative evaluation of the candidate's core signal ("Being at Stripe is no longer the signal it used to be... I don't view someone at Stripe as particularly impressive anymore") IS a pass even though it never mentions a decision (Tom confirmed 2026-07-16: "based on my rather negative feedback it should basically be read as a pass"). Reserve clarifying questions for replies with no evaluative content at all ("hmm", "interesting").
- Defer intent ("interesting but he just re-upped, look again next year") → `track`, parsing the duration from the text (default one quarter).
- Questions about the candidate ("does he have any public writing?") → answer from the store row / `more` behavior; no state change.
- The verb can appear anywhere in the reply, not just first ("the arc is great, draft him").
**The only hard rule: never guess a state-changing action from genuinely ambiguous text** ("hmm", "interesting") — post one short clarifying reply instead. When intent is clear, act; the full verbatim reply is ALWAYS the feedback payload (ledger `why` / `draft_why` / retro nuggets), never a paraphrase.

## Mode B: Webhook (per-reply)

Invoked by claude-job-queue with args `{mode: "webhook", channel_id, thread_ts, reply_ts, user, text, files}` — same shape as claude-alerts-listener.

**Step 0 — ack.** Add a 👀 reaction to Tom's reply (quiet confirmation the job picked up; the bot token at `~/.claude/skills/claude-alerts-listener/.bot_token` has `reactions:write`). If the reaction fails, log and continue. Close-loop thread replies throughout this skill use md_to_blocks.py bot-token mode with `SLACK_THREAD_TS={thread_ts}`.

**Step 1 — resolve the card.** `slack_read_thread` on `thread_ts`. The parent message is the candidate card; extract the `[neg1:<short8>]` fingerprint AND the candidate name from the card's first line. Resolve the -1 Scanner row by searching the data source (`collection://32c00bef-f4aa-80a5-923b-000b83921fa3`) by Name, then VERIFY the page UUID starts with `<short8>`. **Never resolve by short-id alone** — 8-char Notion prefixes collide (see decision-retro's Important rules). If name+prefix don't agree, post an error reply and exit.

**Step 2 — parse the verb** (grammar above) from `text`.

**Step 3 — execute:**

- **draft**:
  - If Status is already `Draft Ready`: don't re-request — reply with the existing `Gmail Draft URL` from the row.
  - If Status is `Reached Out` or `Passed`: reply that the row is terminal; take no action.
  - Otherwise flip Status → `Draft Requested` via `notion-update-page` (this is exactly the Request Draft button pathway; founder-outreach webhook mode drafts regardless of Claude Rec — Tom's "draft" IS the override).
  - Close-loop reply: `🫡 Draft requested for {Name} — Gmail link lands via alert in ~2 min.`
  - No ledger row here — the reach-out is logged at SEND time by pipeline-agent Task 7 (a requested draft Tom never sends must not count as a decision).

- **pass <why>**:
  - Flip Status → `Passed`.
  - Ledger: parse per-signal ratings from the row's `Signals` compact line `NL:{n} · Reps:{n} · Rigor:{n|U} · Ant:{n} · Int:{n} · Rng:{n} · rec:{✅|🤔|❌}` (0-10 numbers; U = unobservable; legacy rows may carry H/M/L letters; `rec` = the PRE-gate auto-rec), map the pre-gate `rec` (falling back to `Claude Rec`) → `reach-out|pass`, then:
    ```bash
    python3 ~/.claude/scripts/decision-ledger/append_decision.py \
      --label "{Name}" --decision no-outreach --date {today} --source "-1 scanner" \
      --verdict-raw "Passed (Slack)" --scores '{...}' --rubric-verdict {reach-out|pass} \
      --rubric-version {status version from founder-taste/RUBRIC.md frontmatter} \
      --why "{<why> verbatim}" --retro-ref "{row URL}"
    ```
  - If `<why>` is present, also log it as a retro (this is decision-retro capture mode for scope=neg1, executed inline): append a `## Retro (YYYY-MM-DD)` block with the verbatim reply to the -1 row page, and append founder-signal nuggets to `~/.claude/skills/founder-taste/DECISION_RETROS.md` in the standard entry format (`- **YYYY-MM-DD · {Name} · -1: Passed** — nugget` + Source line). Then register the capture with decision-retro's queue so the 9:05am neg1-retro-scan doesn't re-prompt: run `queue_append.py --opp-id {page_id} --opp-name {Name} --opp-url {row URL} --decision Passed --trigger-source manual`, then set that entry's `status="completed"` / `completed_at=now()` by full page id (python one-liner on `queue.json` — full-id match only).
  - Close-loop reply: `Logged: passed on {Name}` + one line noting whether the reason contradicted or confirmed the rubric's read (e.g. `— rubric had him ✅ on Earned Reps; your reason is timing-shaped. Override recorded.`). This line is the taste-engine surface — keep it one sentence, no lecture.

- **track <dur>**:
  - Set the row's `Re-surface` date property to today + duration (default 3 months — one quarter). Leave Status unchanged.
  - Close-loop reply: `⏳ Tracking {Name} — card re-surfaces {date}.`

- **more**:
  - Post the row's Eval Summary (full) + the `Signals` line as a thread reply; the deep per-signal rationale lives in the page-body "Signals line (archived ...)" / "Eval Rationale" section — include the Notion link rather than pasting it. No state changes.

**Step 4 — audit log.** Append one line per action to `~/.claude/scheduled-tasks/neg1-sourcing-listener/audit-log/YYYY-MM-DD.log` via `mkdir -p` + `tee -a` (per SHARED_SAFETY.md): `[ts] WRITE: {verb} {Name} neg1:{short8} → {action taken}`.

**Idempotency:** record processed `reply_ts` values in `~/.claude/skills/neg1-sourcing-listener/handled.json` (append-only array). Skip any reply already present — the sweep and the webhook must not double-fire. Check BEFORE Step 3, write AFTER successful execution.

## Mode A: Scheduled sweep (daily reconciliation)

Catches replies the webhook missed (Worker down, route not yet added, Slack event dropped).

1. `slack_read_channel` on `#neg1-sourcing`, lookback `$RUN_LOOKBACK_HOURS` (default 24h; 72h Mondays).
2. For each message containing a `[neg1:...]` fingerprint, `slack_read_thread` and collect Tom's replies (user-ID check; ignore bot messages).
3. For each reply whose `ts` is NOT in `handled.json`: run Mode B Steps 1-4 on it.
4. Single-line summary via `send-alert` ONLY if any actions were taken: `📡 -1 sourcing sweep: {N} replies processed ({verbs})`. Silent when idle.

## Hard rules

- **Never send email.** "draft" only requests a draft; Tom sends by hand from Gmail.
- **Never auto-pass on ambiguity.** Unparseable replies get a clarifying reply, not a guessed action.
- **Never demote a terminal row** (`Reached Out` / `Passed`) — reply and exit.
- **Full-UUID matching only** when touching `queue.json` or resolving rows; short-ids collide.
- Status option values are the canonical -1 Scanner set — never invent new ones; `Re-surface` is a date property, not a status.
