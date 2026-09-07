---
name: preference-miner
description: >
  Proactive preference learning for the personal agent — reflects over recent SMS/family
  threads and the agent's own action logs, INFERS candidate preferences from behavior
  (repeated corrections, choices, approval patterns), and proposes them to Tom for a
  one-tap confirm. Never auto-adopts. Runs nightly (launchd) and on demand ("run
  preference miner", "learn from recent"). Confirmed candidates graduate into the prefs
  corpus (prefs.py) and, once stable, into the skills themselves.
---

# Preference Miner (v2 — proactive)

Reflect, infer, PROPOSE (never silently adopt). The whole value is turning observed
behavior into durable, confirmed preferences so the agent gets better with use — while a
confirm-gate keeps it from learning wrong lessons from noise.

Unattended when scheduled. Reach the end; if nothing worth proposing, exit quietly.

## 1. Gather evidence (recent window — last ~7 days)
Read, most-recent first:
- `~/.claude/skills/sms-listener/audit-log/*.log` — intents + outcomes of every text.
- `~/.claude/local-agents/sms-agent-daemon/daemon.log` — inbound message bodies + timings.
- `~/.claude/skills/purchase-agent/ledger/*.md` — quotes / confirms / declines / abandons.
- `~/.claude/skills/family-inbox/sweep-log/*.log` — what got flagged vs skipped.
Cap the read (tail the logs) — you want the recent signal, not the whole history.

## 2. Infer CANDIDATE preferences (with evidence)
Look for PATTERNS, not one-offs — each candidate needs ≥2 supporting observations or a
clear repeated correction:
- **Repeated corrections** — Tom fixed the same thing ≥2× → the rule behind the fix.
- **Approval patterns** — what he confirms instantly vs scrutinizes (e.g. groceries vs gifts).
- **Timing/scheduling habits** — events he keeps moving, times he avoids.
- **Phrasing/format nits** — recurring tweaks to how replies read.
- **Scope reinforcement** — what each person actually uses.
Write each as a concise, self-contained rule + the evidence. Assign a domain
(`calendar purchases email people general`).

## 3. Dedup + propose
For each candidate, skip if already covered:
`python3 ~/.claude/skills/sms-listener/prefs.py load --all` (active) and
`... prefs.py candidates` (already pending). Only propose genuinely NEW ones:
```bash
python3 ~/.claude/skills/sms-listener/prefs.py propose <domain> "<rule>" "<evidence>"
```
**Be conservative** — a wrong "learned" preference is worse than none. When unsure, don't
propose. Aim for 0–4 high-signal candidates per run, never a pile of speculation.

## 4. Digest to Tom (1:1, not the group — agent-tuning is his call)
If ≥1 new candidate, text Tom via `~/.claude/skills/sms-listener/send_imessage.sh "+12012567714" "<digest>"`.
Honor the formatting prefs (`prefs.py load core`): Title Case header, blank line, no bold.
```
🧠 Preferences I Noticed

• p3 (purchases): Default to grocery re-orders without re-quoting under $50
   — you approved the last 3 grocery quotes instantly, no changes
• p4 (calendar): Don't schedule you before 10am on Mondays
   — you moved 2 early-Monday events last week

Reply "confirm p3" to save, "reject p3" to drop (or "confirm all").
```
When the digest ALSO carries 📌 graduation flags, tell Tom the blanket confirm covers them:
```
📌 Ready to bake into <skill>: "<pref>"

Reply "confirm p3" to save, "reject p3" to drop. "confirm all" saves + graduates every 📌
above; "graduate all" does just the 📌 flags.
```
Nothing new → send nothing.

## 5. Graduation flag (keep the corpus lean)
If an ACTIVE pref (from `prefs.py load --all`) is stable + general enough to belong in a
skill's core behavior, add one line to the digest: `📌 Ready to bake into <skill>: "<pref>"`
— Tom promotes it and it drops from the corpus. **Don't edit skills HERE** (the miner only
flags). The bake happens when Tom replies: **"confirm all" is a blanket yes that graduates
every 📌 flag in the digest** (bake into the named skill + drop the pref from the corpus) in
addition to confirming the numbered `pN` candidates — see sms-listener step 3. So when the
digest carries 📌 flags, the reply line MUST advertise it (below).

## Notes
- Manual trigger: "run preference miner" / "learn from recent" → run steps 1–5 once.
- Confirmation replies ("confirm p3") are handled by sms-listener, not here.
- `SENDBLUE_API_SECRET` is injected by the wrapper/processor; send helper reads `.sendblue_config`.
