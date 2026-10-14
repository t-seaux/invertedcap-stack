---
name: code-first
description: Internal. Queue-only worker for the weekly code-first sweep. It moves one prose rule into a script + harness test, or adds tests for a skill change that shipped untested, then ships it live on passing tests with no text (the weekly 🧱 digest lists it with undo). Not user-triggered.
model: opus
---

# Code-First – convert one rule, live on passing tests

Tom 2026-10-11: *"having me review and respond with something is a bit more friction than I'd like"* → approved
**"do it, tell you, let you undo"**, the skill_change model. Memory: `feedback_code_first_skill_building` (the
judgment/mechanics line and the performance bar) and `feedback_live_checkout_is_production`.

**As code:** `scripts/pipeline.py` (weekly pick / sensitive scope / queue / digest / `undo` / `convert` replies),
`scripts/convert.py` (this job's side effects), `fast.py` (pregate: `finalize` and duplicate / sensitive converts
never start a session). This file covers only the judgment part.

## Args

```json
{"mode": "convert", "kind": "rule" | "harden", "id": "…", "slug": "code-first-…", "skill": "…", "repo": "invertedcap-skills",
 "file": "…", "rule_text": "…", "context": "…",                   // rule
 "commit": "…", "subject": "…", "harden_kind": "none|auto-sync|bypass", "code_files": […], "files": […],   // harden
 "source": "code-first-sweep", "batch": "YYYY-MM-DD", "tom_ok": {…}?, "reply_thread": {…}?}
```
`mode: "finalize"` is handled by `fast.py`. If a session gets it anyway, run
`python3 ~/.claude/skills/code-first/scripts/pipeline.py finalize --batch <batch> --attempt <attempt>` and stop.

## Steps (mode convert)

1. **Prep.** `python3 ~/.claude/skills/code-first/scripts/convert.py prep '<args as JSON>'`. Exit 3 or 5 means it's
   refused or already done, and it's recorded: stop. Exit 0 prints `path` (your change copy), the skill's real-corpus
   `fixtures` and existing `tests`. Work only in `path`. Never edit the live folder, and never use `--no-verify`.
2. **Read before deciding.** Read the skill's SKILL.md and the code around the item.
   - **rule:** read `context`, the whole paragraph. Decide whether it is DETERMINISTIC (a gate, set membership, a
     threshold, a format or filename, an ordering of side effects, a dedup) or JUDGMENT (intent, identity, tone, fit).
     For judgment, run `convert.py fail --id <id> --reason "it's a judgment call, not a checkable rule"` and stop.
   - **harden:** run `git -C <path> show <commit>` and read the change. For `auto-sync`, first review it for
     correctness: broken references, half-edits, a rule that contradicts another, code that can't run. If you can't
     validate it (unclear intent, or a bug whose fix is a judgment call), run
     `convert.py look --id <id> --reason "<one plain line>"` and stop.
3. **Build it as code.**
   - **rule:** put the rule in a script in the skill's dir, or in `shared-references/` when more than one skill uses
     it. The script takes judgment as a required input and refuses with a non-zero exit when the input is missing.
     Add a harness test (`<skill>/tests/test_*.py`). Write it red first, then green, built from the incident the line
     names. Use the listed real-corpus `fixtures` whenever the rule parses real data. Then shrink the prose to
     "run X; on exit N do Y".
   - **harden:** add edge-case tests for the changed code: blank, duplicate and out-of-scope inputs, the real shapes
     in the fixtures, plus the case the commit fixed. Fix a real bug only when the fix is obvious and mechanical.
     Otherwise use `look`.
   - **Performance bar:** the code must do at least as well as the prose did on real cases. Replay them. Anything
     worse is not done; use `fail` with the reason.
   - **Scope:** only the item's skill (plus a shared helper if needed). Don't touch other skills, hooks, guards,
     send / payment / delete paths, or LP-facing output. `prep` already refused those.
4. **Commit.** In `path`, run `python3 scripts/precheck.py --message-file <msg>` until it shows ALL CLEAR. The commit
   message needs a `Hardening: <skill>: <edge cases; what moved to code>` line, any `Lane-parity:` trailer precheck
   asks for, and ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Stage only your files.
5. **Ship.** Run `python3 ~/.claude/skills/code-first/scripts/convert.py ship --id <id> --summary "<one plain line,
   ≤160 chars: what is now enforced in code, from Tom's side; no file names>"`. That makes it live on passing tests,
   sends no text, and records it for the digest. A non-zero exit means nothing shipped and the reason is recorded;
   stop there.
6. Stop. Send no alert, text or Slack post. The weekly digest reports it, and `convert.py` replies in-thread for
   `convert <n>` jobs. If anything breaks mid-way, run `convert.py fail --id <id> --reason "<plain>"` before you exit.
   Budget: about 30 minutes. If it's bigger than that, `fail` with "too big for an automatic conversion".
