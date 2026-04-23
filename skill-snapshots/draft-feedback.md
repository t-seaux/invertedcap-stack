---
name: draft-feedback
description: >-
  Headless feedback engine for outreach + pass-note drafters. Captures sent-vs-draft
  diffs (when Tom edits Claude's draft) AND from-scratch sends (when Tom writes the
  email himself), extracts voice patterns, and writes them to skill-local
  EDIT_PATTERNS.md / VOICE_EXAMPLES.md so future drafts converge on Tom's voice.

  Runs invisibly via the Gmail Watch webhook (FRAMEWORK_PRD.md §13). Apps Script
  detects sends, queues jobs to Drive; a local launchd processor drains the queue
  every 5 minutes, calls `claude --print` to extract patterns, and appends results.
  No Notion fields, no labels, no manual triggers. Drafters write a draft snapshot
  to Drive at draft-creation time; the webhook matches sent → snapshot by Gmail's
  persistent message ID (which survives subject/recipient/body edits).

  Trigger phrases for manual invocation (rare): "run draft feedback", "process
  draft queue", "drain feedback queue", "show recent edit patterns".

  Distinct from `writing-style` (long-form prose voice guide, separate corpus),
  from `decision-retro` (captures decision reasoning, not writing mechanics), and
  from `founder-outreach` / `pass-note-drafter` (the drafters themselves — this
  skill is their feedback layer).
---

# Draft Feedback

Headless feedback engine. Captures voice signal from every outbound outreach and pass note — both edited drafts and from-scratch sends — and feeds patterns back into the drafters so they converge on Tom's voice over time.

Part of the Feedback Engines described in FRAMEWORK_PRD.md §12.2 + §13.

## Architecture (headless, event-driven)

```
Drafter (founder-outreach / pass-note-drafter)
        │
        │ on draft creation: write snapshot
        ▼
Drive: draft-snapshots/<msg_id>.json     ← syncs via Drive Desktop
        │
        │ Tom edits + sends in Gmail
        ▼
Gmail Watch → Pub/Sub → Apps Script doPost
        │
        │ on SENT event:
        │   1. Look up snapshot by message ID (diff mode), OR
        │   2. Match subject heuristic (from-scratch mode)
        ▼
Drive: draft-feedback-queue/<msg_id>.json
        │
        │ launchd fires every 5 min
        ▼
processor.py
        │
        │ calls `claude --print` to extract patterns
        ▼
EDIT_PATTERNS.md (diff mode) OR VOICE_EXAMPLES.md (from-scratch mode)
        │
        │ next drafter run reads both files
        ▼
Drafter applies patterns as priors when drafting next email
```

Two operational modes, one shared infra:

| Mode | Trigger | Output |
|------|---------|--------|
| **diff** | Drafter created a draft, Tom edited & sent | Append delta patterns to `EDIT_PATTERNS.md` |
| **from-scratch** | No snapshot, but subject matches outreach/pass-note pattern | Append voice analysis to `VOICE_EXAMPLES.md` |

## Components

### 1. Drafter snapshot writes

Each drafter, after creating a Gmail draft, writes a JSON snapshot to Drive:

```
~/Library/CloudStorage/GoogleDrive-tom@invertedcap.com/My Drive/draft-snapshots/<hex_msg_id>.json
```

Schema:

```json
{
  "skill": "founder-outreach" | "pass-note-drafter",
  "messageId": "<gmail persistent hex id>",
  "threadId": "<gmail thread id>",
  "recipient": "<email>",
  "subject": "<exact subject>",
  "draftText": "<plain-text body, signature block excluded>",
  "createdAt": "<ISO 8601>"
}
```

Drive Desktop syncs to the cloud within seconds. See each drafter's SKILL.md (Step 9 for `founder-outreach`, Step 6 for `pass-note-drafter`) for exact instructions.

### 2. Apps Script webhook (the router)

`~/.claude/skills/shared-references/gmail-webhook-code.gs` — deployed in the gmail-webhook Apps Script project. On every Gmail SENT event:

**Track 1 — diff mode (snapshot exists).** Look up `draft-snapshots/<msg.id>.json`. If found, queue:

```json
{
  "mode": "diff",
  "sent_message_id": "...",
  "sent_thread_id": "...",
  "sent_text": "<extracted plain text>",
  "sent_subject": "...",
  "snapshot": { ...full snapshot data... },
  "snapshot_file_id": "..."
}
```

Match key is the **persistent Gmail message ID**, which is stable across subject/recipient/body edits (Gmail just flips DRAFT → SENT labels on send, the message ID doesn't change).

**Track 2 — from-scratch mode (no snapshot).** Apply subject heuristic:

- `Subject: Introducing Inverted Capital` → `skill = founder-outreach`
- `Subject: ... - Inverted follow up` (not a Re: or Fwd:) → `skill = pass-note-drafter`

Match → queue:

```json
{
  "mode": "from-scratch",
  "skill": "...",
  "sent_message_id": "...",
  "sent_text": "...",
  "sent_subject": "...",
  "sent_recipient": "..."
}
```

No match → log `no-snapshot` and exit. Personal emails are not captured.

Both tracks write to `draft-feedback-queue/<sent_msg_id>.json` in Drive.

### 3. Local processor (the worker)

`~/.claude/skills/draft-feedback/processor.py` — runs every 5 minutes via `~/Library/LaunchAgents/com.tomseo.scheduled.draft-feedback-processor.plist`.

For each queue entry:

- **diff mode:** read sent + draft, call `claude --print` with diff prompt, append delta patterns to `~/.claude/skills/<skill>/EDIT_PATTERNS.md`.
- **from-scratch mode:** read sent text only, call `claude --print` with voice analysis prompt, append full sent + analysis to `~/.claude/skills/<skill>/VOICE_EXAMPLES.md`.

After successful processing:
- Archive raw entry to Drive `draft-feedback-archive/<msg_id>.json` for auditing
- Delete queue entry
- Delete corresponding snapshot (consumed)

On failure: leave queue entry in place — next run retries.

### 4. 30-day TTL purge

Apps Script `purgeOldSnapshots` runs daily (manual trigger setup required), trashes any snapshot in `draft-snapshots/` older than 30 days. Catches drafts Tom never sent (forgot, decided not to reach out, etc).

## Drafter consumption

Each drafter (`founder-outreach`, `pass-note-drafter`) reads both pattern files at runtime:

- `EDIT_PATTERNS.md` — accumulated deltas. "What does Tom typically cut, simplify, reorder?" Apply as priors.
- `VOICE_EXAMPLES.md` — canonical from-scratch sends. "What does Tom's voice look like when he writes alone?" Use as ground truth.

Drafters explicitly do this in their respective Step 4 (founder-outreach) / Step 5 (pass-note-drafter).

## Manual invocation

Mostly unnecessary — runs automatically via launchd. But for debugging or processing a backlog:

```bash
python3 ~/.claude/skills/draft-feedback/processor.py
```

To inspect queue state:

```bash
ls "$HOME/Library/CloudStorage/GoogleDrive-tom@invertedcap.com/My Drive/draft-feedback-queue/"
ls "$HOME/Library/CloudStorage/GoogleDrive-tom@invertedcap.com/My Drive/draft-snapshots/"
```

To inspect launchd status:

```bash
launchctl list | grep draft-feedback
tail ~/Library/Logs/draft-feedback-processor.log
```

## Important rules

- **Never send anything.** This skill only reads sent mail and writes local pattern files.
- **Match by persistent Gmail message ID, not by subject/recipient/threadId.** The message ID survives every edit a draft can undergo (subject change, recipient change, wholesale body rewrite). All three corner cases are handled gracefully.
- **From-scratch heuristic is intentionally narrow** (exact outreach subject + pass-note suffix). Better to under-capture than to falsely capture personal emails as voice signal.
- **Extracted patterns are durable, raw archives are bulk.** `EDIT_PATTERNS.md` and `VOICE_EXAMPLES.md` are committed to skills directory; raw archives live in Drive and may rotate.
- **Patterns are observations, not commands.** Drafters read them as "what Tom has done" — apply as priors, not rigid rules. Tom's voice is contextual.
- **Never clean up Tom's writing during extraction.** If his edit introduced a typo or unusual phrasing, capture it as-is. That's signal, not noise.
- **Idempotent.** Apps Script dedupes on historyId + message ID; processor is safe to re-run on the same queue entry; consumed snapshots are deleted.

## Failure modes

| Failure | Behavior |
|---------|----------|
| `claude --print` errors | Queue entry stays, retries next 5-min run |
| Drive Desktop sync lag | Snapshot may not be present yet when SENT fires; queue entry written with `no-snapshot` log line; manual rerun later catches it |
| Snapshot orphaned (Tom never sent) | Auto-purged at 30 days via `purgeOldSnapshots` |
| Subject heuristic false positive | Worst case: a non-outreach email gets logged to `VOICE_EXAMPLES.md`. Manually delete the entry. |
| Webhook missed entirely | Apps Script `last_history_id` advances on every notification; if a notification is somehow lost, the next one's history list will catch the missed message |

## Relationship to `writing-style`

`writing-style` is the long-form voice guide — memos, LP letters, thesis prose. `draft-feedback` is the short-form equivalent, scoped to drafter flows. Parallel corpuses. Over time, patterns learned here may get promoted into `writing-style` if they generalize beyond outreach/pass notes.

## Reference files

- `processor.py` — local launchd worker
- `~/.claude/skills/shared-references/gmail-webhook-code.gs` — Apps Script source
- `~/.claude/skills/shared-references/gmail-webhook-setup.md` — one-time GCP/Apps Script setup guide
- `~/Library/LaunchAgents/com.tomseo.scheduled.draft-feedback-processor.plist` — launchd config
- `~/Library/Logs/draft-feedback-processor.log` — processor run log
