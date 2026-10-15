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
(`calendar purchases email people general`) — `prefs.py propose` routes it to the owning skill.

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
Destination: `prefs.py propose` routes it (`prefs.py route <domain> "<rule>"` to preview) – an owned domain is tagged with its owning skill (add-to-calendar, purchase-agent, …); only text-surface rules stay corpus-bound.
**Be conservative** — a wrong "learned" preference is worse than none. When unsure, don't
propose. Aim for 0–4 high-signal candidates per run, never a pile of speculation.

## 4. Digest to Tom (1:1, not the group — agent-tuning is his call)
If ≥1 new candidate, text Tom with ONE command (renders the card with each pref's real destination, sends it to his
1:1 and stages it so a 👍 / 👎 tapback resolves in code – Tom 2026-10-14):
`python3 ~/.claude/skills/sms-listener/prefs.py send-card <pN> [<pN> ...]`. Never hand-write the card.
- Skill/preference-graduation escalation always goes to Tom 1:1, never into the Kenyon-Seo family group thread — even if the triggering pattern was observed in group messages (e.g. Elsie's requests too). Applies to the nightly preference-miner digest and any in-session skill-worthy flag.  (2026-08-31)

What `send-card` prints (`prefs.py card <ids>` previews it):
```
🧠 Preferences Noticed: 2 New

• p3 (→ texts): Links in texts end with ' ↗'
   — seen twice
• p4 (→ purchase-agent): Prefer Prime-eligible listings
   — you picked Prime 3 times

👍 saves all · 👎 drops all · "confirm p3" for just one
```

## 5. One-step confirm — no separate graduation
Every candidate is proposed WITH its destination (step 3), so a single "confirm" lands it in
its final home. A text-surface pref is written by `prefs.py confirm` directly. An owned pref (a
skill, the shared email spec `shared-references/email-formatting.md`, or auto-memory) is marked
confirmed and QUEUED to **compile mode** below. That is the ONE place an owned pref gets written,
whether Tom confirmed by text or in a session (Tom 2026-10-14: "Text should simply be the router").
No surface edits the owner on its own.

## 6. Compile mode (job args `{mode:"compile", id, owner, file, text, evidence}`)
Queued by `prefs.py confirm` / `prefs.py add`. Headless; reach the end.
1. **Behavior-match first.** Open `file` and search for the SPECIFIC behavior the rule describes, not
   shared keywords (2026-09-06: a "check calendar before web-searching" pref was dropped as "already
   present" because of an unrelated dedup rule). Already encoded → quote the exact line, skip to 4.
2. **Write it** with skill_change: `python3 ~/.claude/scripts/skill_change.py start <slug>` (slug =
   the owner; `shared-references` for email-formatting.md). Add ONE concise, self-contained rule in the
   right section, dated, in that file's voice. If it CONFLICTS with an existing rule, amend that rule
   instead of adding a contradicting line. auto-memory owner → one feedback file + its MEMORY.md line,
   or update the file that already covers it.
3. Commit in the worktree, then `skill_change.py propose <slug> --summary "pref <id>: <rule>"` (never
   `--in-session`). Tests gate it; Tom gets the 🛠️ Live text with 👎 to undo. Propose fails → stop,
   leave the candidate confirmed, send-alert `🛠️ Pref Compile: Failed`.
4. `python3 ~/.claude/skills/sms-listener/prefs.py graduated <id>`.

## Notes
- Manual trigger: "run preference miner" / "learn from recent" → run steps 1–5 once.
- Confirmation replies ("confirm p3") are ROUTED by sms-listener (`prefs.py confirm`), then compiled HERE (step 6).
- `SENDBLUE_API_SECRET` is injected by the wrapper/processor; send helper reads `.sendblue_config`.
