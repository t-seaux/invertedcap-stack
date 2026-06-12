---
name: batch-add-to-crm
description: Fan out parallel add-to-crm runs when one source (a single email, message, doc, or pasted blurb) mentions MULTIPLE companies that should all land in the Notion Opportunities DB. Trigger when Tom says "batch add to crm", "add all of these to crm", "log all of these deals", "crm all of these", "split this into separate opps", "run parallel add-to-crm on these", or shares a multi-company list (e.g. David Talpalar's "A Few Interesting Ones" digest, a partner intro batch, a curated deal sheet) and asks to log them. Each company spawns its own independent add-to-crm sub-agent, all sharing the same Source. Manual mode only — webhook entry is via `inbound-deal-detect` which always handles single companies.
---

# Batch Add to CRM

Orchestrator that fans out `add-to-crm` runs across multiple companies surfaced from a single source, all sharing one Source person.

## When to invoke

The single-company `add-to-crm` skill is the right answer for one inbound pitch. Use this skill instead when:

- A single email/message/doc names ≥2 distinct companies
- Tom wants them all logged as separate Opportunities (not as variants of one)
- They all share the same Source person (the sender of the surfacing email/message)

If the source is a multi-deal Chris-Oh-style market scan with no opt-in intent ("here are 8 companies I saw this week"), that's a **deal-digest** ingest, not batch-add-to-crm. The discriminator: are these companies Tom should actively consider as deals? If yes → this skill. If they're context/intel → `deal-digest`.

## Architecture

Each company runs as an **independent sub-agent** via `Agent` with `subagent_type: "general-purpose"`. Sub-agents fire in **parallel** — they have no ordering dependency (different Opportunities, different Notion writes). The orchestrator collects each sub-agent's brief return summary and presents one consolidated report at the end.

Single source of truth: each sub-agent reads `add-to-crm/SKILL.md` itself. This skill does NOT inline add-to-crm logic — any update to add-to-crm propagates automatically.

## Step 1: Identify the Source person

The Source is the human who surfaced this batch. Resolve them once, up front, and pass to every sub-agent so all Opps land with the same Source(s) relation.

1. From the source material (email From:, message sender, or Tom's narration), pull the source person's name + email + LinkedIn URL if present.
2. Confirm the resolution with Tom in one sentence before fanning out: "Source = David Talpalar (david@shirlawncap.com). Spawning N parallel add-to-crm runs."
   - Skip the confirm when Tom has already named the source in his invocation ("source is David", "from David's email").

Pass the source as a `sourceDirective` object: `{ email: "<email>", name: "<full name>", li_url: "<linkedin>" }`. Each sub-agent feeds this through add-to-crm's normal People-DB resolution (no auto-create — surface the gap in its return summary if missing).

## Step 2: Enumerate companies

Walk the source material end-to-end and extract a list of `{company, website, founder_names[], founder_li_urls[], round_details, blurb}` objects — one per company. Keep raw text + URLs verbatim — sub-agents will re-enrich, this is just to scope each sub-agent's task.

Show Tom the enumeration before fanning out:

> Found 4 companies: Lucius, Simple Product, Iridium Credit, Factor Labs. Spawning 4 parallel add-to-crm runs.

If Tom replies with a correction (drop one, merge two, add one), reconcile before spawning.

## Step 3: Fan out

Spawn one sub-agent per company **in parallel** — single message, multiple `Agent` tool calls. Each sub-agent gets a prompt that:

1. Identifies itself as a delegated add-to-crm run from `batch-add-to-crm`
2. Inlines the per-company extraction (company, website, founders+LI, round, blurb) so the sub-agent doesn't have to re-parse the source
3. Inlines the **gmailMessageUrl** / source-material pointer so add-to-crm Step 1B can still pull the original for context (and Step 6 can link materials if any)
4. Inlines the `sourceDirective` resolved in Step 1
5. Restates Tom's standing rules that don't transit to spawned sub-agents automatically:
   - **No permission prompts** for prescribed writes (Notion creates, People-DB resolution, Slack alert suppression). Tom invoked this end-to-end — blanket auth.
   - **No People DB writes** without explicit permission — if the source person isn't found, surface the gap and continue without backfilling.
   - **No "want me to..?" trailing offers** — finish and stop.
   - **Source commentary VERBATIM in Opp body** — every sub-agent prompt must inline the source person's blurb about that specific company (the exact words from the email/message/doc) AND instruct the sub-agent to write it into the Opp page body as a `> ` blockquote attributed to the source. Format: `> "<verbatim blurb>" — <Source name>, <date>`. This is the most valuable signal in a batched intro — the Source's read on each company is why Tom asked to log them. Do NOT paraphrase, summarize, or filter (including passes — "wasn't for me" / "too rich" stays in verbatim). Log under a clear heading like `### David Talpalar's note` so it's findable.
   - **Hosted memo / deck links → PDF, not URL** — when the source blurb references a hosted memo, deck, one-pager, or write-up (e.g. `simpleproduct.dev/share/...`, Notion page, public Google Doc), instruct the sub-agent to render it to PDF via materials-handler's Chrome headless path AND upload to Drive AND write the Drive file into the Diligence Materials property. URL-only linking (add-to-crm's Step 3E fallback) is NOT acceptable for batch-add-to-crm — Tom wants an offline-readable PDF artifact. If WebFetch times out (common for AI-styled marketing pages), fall through to Chrome headless — do NOT settle for the URL-only path. Surface explicitly in the return summary if the render fails.
6. Suppresses individual Slack alerts — the orchestrator composes ONE consolidated alert at the end (mirroring `run-all`'s pattern).

Use `model: "sonnet"` for sub-agents. add-to-crm is a scan-and-write workflow; Opus is overkill.

## Step 4: Collect + report

Wait for all sub-agents to return. Each sub-agent's summary should contain:
- Opp page ID + URL
- Founder relations created/resolved
- Status assigned
- Any dedup hits (Protected Status Guard fires) or missing-data flags

Compose ONE consolidated report to Tom inline:

```
Batch add-to-crm complete. Source: David Talpalar.

✅ Lucius — Opp <url>, Founder: Ryan Egralia + Andre Elias, Status: Qualified
✅ Simple Product — Opp <url>, Founder: Andrew Pitre, Status: Qualified  [memo attached]
🛡️ Iridium Credit — DUP (existing Opp <url>), no new Opp created
✅ Factor Labs — Opp <url>, Founder: Max Wolff, Status: Qualified
```

Then send ONE Slack alert via `send-alert` summarizing the batch (one bullet per company, same format).

## Notes

- Sub-agents have isolated context budgets — large batches (10+) won't overflow the parent.
- If a sub-agent fails mid-run, report the failure inline; do NOT auto-retry. Tom decides.
- If the source itself is a forwarded email Tom received from a referrer, the Source = referrer (not the founders, not Tom). add-to-crm's Protected Status Guard inside each sub-agent still runs against the union of all harvested signals (inner Forwarded blocks, etc.).
