---
name: remote-session-cleanup
description: >
  Free up claude.ai remote-session capacity by archiving stale remote
  (anthropic_cloud) Claude Code sessions when the account hits its
  remote-session limit. Archive only — never delete; archived sessions
  remain visible on claude.ai/code and can be unarchived. Two triggers:
  (C-auto) SUBROUTINE — any time an attempt to launch a remote session or
  remote agent fails with a session-limit / capacity error ("maximum number
  of remote sessions", "session limit reached", or similar), invoke this
  skill inline, archive the stalest eligible sessions to make room plus
  headroom, then RETRY the original launch. (C-manual) Tom says "clean up
  remote sessions", "archive stale remote sessions", "we hit the remote
  session limit", "free up remote session space", or any variant asking to
  prune claude.ai remote sessions. Always trigger inline — no confirmation
  needed before archiving eligible sessions (reversible op); ineligible
  candidates are surfaced instead.
---

# Remote Session Cleanup

Archives stale remote (`anthropic_cloud`) Claude Code sessions on claude.ai to free capacity under the account's remote-session limit. Sessions with `environment_kind: "bridge"` are local desktop sessions — **never touch them**; they don't count against the remote limit.

**Archive is the only mutation.** It is reversible (unarchive from claude.ai/code), which is why this skill runs without a pre-confirmation gate despite the global confirm-before-destructive rule. Anything that would require a hard delete is out of scope — surface to Tom.

## Step 1 — List

```bash
python3 ~/.claude/skills/remote-session-cleanup/scripts/remote_sessions.py list
```

Returns all `anthropic_cloud` sessions (newest first) with `status` (active/archived), `status_bucket`, `worker_status`, `title`, and `age_hours` (since last event). Only `status: "active"` sessions count toward the limit.

## Step 2 — Pick candidates (the rubric)

From the **active** sessions, rank stalest-first by `age_hours` and select for archiving, applying ALL of:

- **Never archive** a session with `worker_status: "running"` — it's mid-task.
- **Never archive** a session with `age_hours < 24` — too recent to call stale.
- **Prefer** sessions whose `status_bucket` is `completed` and whose title reads as a finished one-off (a review, a single research question) over anything that reads like an ongoing workstream.
- **Read the titles.** If a title matches a live project Tom is actively working on (check recent conversation context), skip it even if old.

How many: enough to clear the limit **plus 2 headroom**. Manual mode with no stated count: archive everything eligible older than 7 days, and list (but don't archive) the 24h–7d ones in the alert.

If nothing is eligible but the limit is still hit (all active sessions are running or <24h old), do NOT force it — post the blocked-state alert (Step 4) and let Tom pick.

## Step 3 — Archive

```bash
python3 ~/.claude/skills/remote-session-cleanup/scripts/remote_sessions.py archive --ids cse_XXX cse_YYY
```

The script archives each ID and re-fetches to verify `status == "archived"`; exit code 1 if any failed. If it reports the endpoint shape failed on both variants (404 on both the partial-update POST and `/archive`), stop and tell Tom the API shape changed — don't improvise other mutations.

In C-auto mode, after `all_ok: true`, retry the original remote launch that triggered the cleanup.

## Step 4 — Report

Via **send-alert** (subroutine trigger from another skill or scheduled context) or directly in chat (interactive):

- What was archived: title, id, age — one line each.
- What was spared and why, if anything borderline was skipped.
- Reminder: "reversible — unarchive at claude.ai/code".
- Blocked state: list the active sessions with ages and ask Tom which to archive.

## Notes

- Auth: OAuth token from macOS keychain (`Claude Code-credentials`), read inside the script — never echo it.
- API: `GET/POST https://api.anthropic.com/v1/code/sessions` with `anthropic-beta: oauth-2025-04-20` (same family as the RemoteTrigger `/v1/code/triggers` API). List + read verified 2026-07-31; the archive POST follows the API's partial-update convention and self-verifies on first live run.
