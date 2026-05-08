---
name: claude-job-queue-smoke-test
description: Infrastructure smoke test for the claude-job-queue processor. Not user-facing. Invoked by processor.py to verify the webhook → Drive → launchd → claude-print chain is live. Never trigger manually.
---

# claude-job-queue smoke test

This skill is a no-op smoke test used ONLY by the claude-job-queue-processor
to verify end-to-end health of the webhook-dispatch chain. It takes any
arguments and simply echoes "claude-job-queue smoke test OK" plus the
arguments it received, then exits.

## Steps

1. Print the literal line: `claude-job-queue smoke test OK`
2. Print the arguments you received from the job (should include at least
   `__smoke_test: true` and a `ts` timestamp).
3. Exit. Do not touch Notion, Gmail, Drive, or any external system.

## Output format

```
claude-job-queue smoke test OK
args: {"__smoke_test": true, "ts": "<timestamp>"}
```
