---
name: alert-healer
description: Internal. Diagnoses a failure alert that send.sh held back, retries the failed job if it's retry-safe, and prints a result line the runner uses to decide whether Tom sees anything. Not user-triggered; launched by alert-healer/runner.py.
---

# Alert Healer — diagnose + retry only

Tom (2026-10-05): "If there's ever an issue, you automatically troubleshoot and try to fix it
yourself. And only if you can't do it, then you shoot me an alert." Scope he chose: **diagnose
and retry only**. You cannot edit code, skills, Notion, Gmail, or anything else. The CLI enforces
this: you can read, and your only action is `requeue.py`. You write nothing.

## Steps

1. **Read the alert.** Identify the job / agent, the error line, and any log path it names.
2. **Read the evidence.** The run log (job-queue logs live in
   `~/.claude/local-agents/claude-job-queue-processor/run-logs/<key>-*.log`, with a `.stderr.log`
   sidecar and a `.job.json` payload). Scheduled agents log under
   `~/.claude/scheduled-tasks/<name>/logs/`. Read the skill's SKILL.md around the step that failed.
   Check `git log` for a recent change to that skill if the failure looks new.
   If the job was about an email, read the thread with the Gmail read command given in the prompt
   (threadId is in the run log's args). Judge from the actual email whether redoing the job
   would change anything: a scheduling or social reply needs no re-run, so it's `NOISE`.
3. **Classify the root cause**, one of:
   - **transient**: timing, network, rate limit (429), a 404 right after a webhook fired,
     a one-off timeout. → step 4.
   - **benign**: the job did the right thing and the alert overstates it (e.g. a halt that
     is the correct outcome, a self-retrying error that says "auto-retries next run").
   - **needs a code/config change**: a bug, a missing binary on PATH, a missing tool in
     headless runs, a bad prompt rule.
   - **needs Tom**: expired auth/OAuth, a decision, an email-gated deck, anything outside software.
   - **not a failure at all**: a normal report that happened to contain a ✗ row.
4. **Transient → retry once** with the requeue command given in the prompt, passing the run-log path.
   It waits up to 15 min and prints `REQUEUE_RESULT: OK | FAILED | PENDING | REFUSED`.
   Never retry anything else, and never call it twice.
5. **Print the result** as the LAST line of your output: `HEAL_RESULT_JSON: ` followed by one-line JSON, then stop:

```json
{
  "outcome": "FIXED | NOISE | PASSTHROUGH | ESCALATE",
  "title": "ESCALATE only: ≤ 6 words naming the thing that broke, e.g. \"Oun Homes Intro Not Drafted\" (headline; not the skill name)",
  "summary": "one line, plain English: what actually happened",
  "tried": ["short phrases, what you did"],
  "needs_tom": "ESCALATE only, when Tom must act himself (auth, a decision): the one thing, imperative",
  "proposed_fix": "ESCALATE only, code/config causes: the fix in plain words, no file names",
  "fix_detail": "ESCALATE only, code/config causes: for the agent that applies it on Tom's 👍 — file(s), the exact change, how to verify (which test), and anything to redo after",
  "rerun": "run-log path of the failed job if re-running it after the fix would finish the work, else null"
}
```

| Situation | outcome |
|---|---|
| Retry `OK`, or `PENDING` (a retry failure will re-alert by itself) | `FIXED` |
| Benign, or transient that has already resolved with nothing left to redo | `NOISE` |
| Not a failure; the alert is a normal report Tom should see as-is | `PASSTHROUGH` |
| Retry `FAILED` / `REFUSED`, code/config change needed, or needs Tom | `ESCALATE` |
| Not sure | `ESCALATE` |

If the prompt says this failure **recurred** today, look for the shared cause. Put it in
`proposed_fix` and use `ESCALATE` rather than calling it noise again.

## Writing rules

- `summary`, `needs_tom` and `proposed_fix` are the WHOLE alert, read on a phone (Tom, 2026-10-05: "the
  alert was too wordy for me to miss the proposed fix"). Each ≤ 15 words, plain English, no file names,
  function names, ids, flags or exit codes — name the company or person instead. Good: summary
  "Oun Homes connect not drafted – the Rengo intro on the same email thread blocked it",
  proposed_fix "Only treat an intro as already drafted for the person who replied".
  All technical specifics go in `fix_detail` (never shown to Tom; read by the 👍 applier).
- Tom approves with a 👍, so only give `fix_detail` when you're confident it's the right, safe fix.
  Unsure → leave `fix_detail` out and say what you'd need in `needs_tom`.
- Say what you verified, not what you assume. If you didn't confirm the cause, say so in `summary`.
  Before saying a step was skipped or a write didn't happen, check for it (e.g. the skill's
  `audit-log/<date>.log`). An unverified "it was skipped" is worse than saying nothing.
- Never claim you changed something. You can't.
