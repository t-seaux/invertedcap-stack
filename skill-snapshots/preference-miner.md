---
name: preference-miner
description: >
  Proactive preference learning for the personal agent — reflects over recent SMS/family
  threads and the agent's own action logs, INFERS candidate preferences from behavior
  (repeated corrections, choices, approval patterns), and proposes them to Tom for a
  one-tap confirm. Never auto-adopts. Runs nightly (launchd) and on demand ("run
  preference miner", "learn from recent"). Each candidate is proposed WITH a destination
  (corpus tier vs a skill's SKILL.md); a single confirm lands it in that final home — no
  separate graduation step.
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
`... prefs.py candidates` (already pending). Only propose genuinely NEW ones — and tag each
with its DESTINATION at propose-time, so that on confirm it lands in its final home in one
step (no separate graduation):
```bash
# narrow runtime override → lives in the corpus tier:
python3 ~/.claude/skills/sms-listener/prefs.py propose <domain> "<rule>" "<evidence>"
# skill-CORE behavior → compiles into that skill's SKILL.md on confirm (append the skill name):
python3 ~/.claude/skills/sms-listener/prefs.py propose <domain> "<rule>" "<evidence>" <skill>
```
Judge the destination: if the rule changes how a skill fundamentally behaves (e.g. how
family-inbox phrases finance alerts, how sms-listener handles a shared link) → tag the skill.
If it's a narrow domain tweak (a time bound, a format nit) → leave it corpus-bound. When
genuinely unsure, leave it corpus-bound (the cheaper, reversible home).
**Be conservative** — a wrong "learned" preference is worse than none. When unsure, don't
propose. Aim for 0–4 high-signal candidates per run, never a pile of speculation.

## 4. Digest to Tom (1:1, not the group — agent-tuning is his call)
If ≥1 new candidate, text Tom via `~/.claude/skills/sms-listener/send_imessage.sh "+12012567714" "<digest>"`.
Honor the formatting prefs (`prefs.py load core`): Title Case header, blank line, no bold.
Show each candidate's destination so Tom knows where a "confirm" sends it — `→ <skill>` for
skill-core prefs, `(corpus)` for narrow overrides:
```
🧠 Preferences I Noticed

• p3 (purchases, corpus): Default to grocery re-orders without re-quoting under $50
   — you approved the last 3 grocery quotes instantly, no changes
• p4 (family-inbox → skill): Finance alerts — always say automatic vs needs-action upfront
   — you asked twice which payments needed your input

Reply "confirm p4" to save, "reject p4" to drop (or "confirm all").
```
Nothing new → send nothing.

## 5. One-step confirm — no separate graduation
Every candidate is proposed WITH its destination (step 3), so a single "confirm" lands it in
its final home — skill-core prefs compile into the named `SKILL.md`, narrow ones stay in the
corpus tier. There is no "📌 ready to graduate" flag and no "graduate all" reply: a pref Tom
has confirmed is never left loitering in the corpus waiting for a second blessing. **The miner
never edits skills itself** — it only proposes with the right destination; the bake happens on
Tom's confirm, handled by sms-listener step 3 (open the target SKILL.md, behavior-match,
write-if-missing, then `prefs.py graduated pN`).

## Notes
- Manual trigger: "run preference miner" / "learn from recent" → run steps 1–5 once.
- Confirmation replies ("confirm p3") are handled by sms-listener, not here.
- `SENDBLUE_API_SECRET` is injected by the wrapper/processor; send helper reads `.sendblue_config`.
