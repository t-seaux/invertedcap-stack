---
name: share-skills
description: >
  Regenerate the sanitized, shareable bundle of Tom's skill library against the latest corpus.
  Stages every publicly-mapped SKILL.md (per skill-map-refresh's canonical visible/hidden
  classification), runs a deterministic PII scrub (emails, record IDs, endpoint URLs, paths,
  fund identifiers) plus a semantic name-scrub via parallel subagents (real founder/investor/
  company names → neutral placeholders), verifies zero residuals with a hard grep gate, and
  packages a zip with README at ~/Downloads/inverted-skill-library.zip. Incremental: unchanged
  skills reuse their cached sanitized copy (cache/manifest.json tracks source hashes); only
  changed/new skills re-run the scrub pipeline, and a rules change invalidates the whole cache.
  The verification gate always runs over the full bundle. Manual-only. Trigger
  phrases: "bundle skills", "share skills", "share my skills", "skill bundle", "bundle my
  skills", "refresh the skill bundle", "rebuild the share bundle", "regenerate the sanitized
  skill bundle", "update the skill share zip", "refresh the share bundle", or any variant
  asking to produce a shareable copy of the skill library. Not the skill-map-refresh skill
  (that updates the public /stack visuals) – this produces the full sanitized SKILL.md corpus.
---

# Share Skills

Produce a sanitized snapshot of the skill library that Tom can hand to anyone outside the firm.
The bundle contains only skills classified as **visible** on the skill map, with all compromised
info removed: credentials, endpoint URLs, record IDs, emails, phones, paths, fund back-office
identifiers, and real person/company names from example scenarios. Workflow logic, architecture,
data-source design, and framework content are preserved – that's the point of sharing.

**Canonical classification source**: `~/.claude/skills/skill-map-refresh/SKILL.md`. Never
hardcode the include/exclude lists here – read them live each run so the bundle tracks the map.

## Step 1: Derive the include set

1. Inventory every real SKILL.md. Symlink + vendored-dir gotchas are both live:
   ```
   find -L ~/.claude/skills -name SKILL.md \
     -not -path '*/venv/*' -not -path '*/node_modules/*' -not -path '*/site-packages/*'
   ```
   Many skill dirs are symlinks into `~/Projects/invertedcap-skills/` – plain `find` (no `-L`)
   silently skips them. Vendored Python deps contain junk `SKILL.md`-adjacent matches.
2. Read the classification tables in `skill-map-refresh/SKILL.md` and build the exclude set:
   every skill named in **Hidden Categories** (Fund Ops + Admin), **Excluded duplicates**, and
   **Excluded – not a user-facing skill**. This skill (`share-skills`) is itself Admin-hidden.
3. Include set = inventory minus exclude set. If a skill on disk appears in NO table (visible or
   hidden), STOP and flag it to Tom – same anti-hallucination stance as skill-map-refresh Step 0.
   Do not guess a classification.

## Step 2: Diff against the cache (incremental by default)

The expensive step is the semantic scrub – only run it on skills whose source changed.
`cache/sanitized/<name>.md` holds the last sanitized output per skill; `cache/manifest.json`
maps each skill to the sha256 of its source SKILL.md at last sanitize time, plus hashes of the
sanitization rules themselves.

1. Write the include set (one name per line) to `/tmp/include_skills.txt`.
2. `python3 ~/.claude/skills/share-skills/manifest.py diff /tmp/include_skills.txt` → JSON with
   `rules_changed`, `new`, `changed`, `removed`, `unchanged`.
3. **If `rules_changed` is true** (scrub.py or redaction-spec.md edited since last build), the
   whole cache is stale – old sanitizations miss the new rules. Treat ALL skills as changed.
4. Work set = `new` + `changed`. Report the split to Tom ("3 changed, 46 cached"). If the work
   set is empty and `removed` is too, the bundle is current – skip to Step 5 only if the zip is
   missing, else report "no changes" and stop.

## Step 3: Stage + deterministic scrub (work set only)

Clear `/tmp/skill-bundle/skills/` (`find ... -type f -delete`), then copy ONLY the work set's
source SKILL.md files there as `<name>.md`. Run
`python3 ~/.claude/skills/share-skills/scrub.py` – it edits the staged files in place and
prints a per-file + total hit report. It handles all *structured* compromised data: emails,
Notion/Drive/Docs record IDs, Apps Script `/exec` endpoint URLs, `/Users/<user>` paths, the
Notion workspace slug, launchd labels, fund back-office identifiers (dashfund / Dash N vehicle
names), bank-account patterns, key-shaped tokens, and known personal phone numbers. Review the
totals for anything anomalous (e.g. a new hit category at high volume – may signal a new leak
class worth adding a rule for).

## Step 4: Semantic name scrub (work set only)

Structured scrub can't catch prose names. Fan out parallel general-purpose subagents (batches of
~7 files each; a small work set may need just one agent), every agent instructed to read and
apply `~/.claude/skills/share-skills/references/redaction-spec.md` – the single canonical
redact/keep spec. Agents edit in place and report the names they redacted per file. Read every
agent's report; if one flags a judgment call (public-figure names, infra brands), decide or
surface to Tom rather than ignoring it.

Then sync the cache: copy the freshly sanitized files into `cache/sanitized/`, delete
`cache/sanitized/<name>.md` for every skill in `removed`, and copy all `unchanged` cached files
into the staging dir so `/tmp/skill-bundle/skills/` holds the complete bundle.

## Step 5: Stamp classifications, then verification gate (hard – any hit blocks packaging)

First run `python3 ~/.claude/skills/share-skills/stamp.py` – it parses the canonical Function
mapping + Composite breakdown tables live from `skill-map-refresh/SKILL.md` and inserts a
`function:` key into each staged file's frontmatter, so the reader sees where every skill sits
on the map. Package-time only: sources and `cache/sanitized/` stay unstamped (classification is
always derived fresh – no second copy to drift). If stamp.py exits non-zero with UNMAPPED
skills, surface them to Tom; do not guess.

Then the gate. Safety is NEVER incremental: run every check against the COMPLETE staged bundle
(cached + fresh files alike); ALL must return empty before Step 6:

```bash
cd /tmp/skill-bundle/skills
grep -rhoE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' . | grep -v 'you@example.com'  # emails
grep -rho 'AKfycb[A-Za-z0-9_-]*' .                                # Apps Script deploy tokens
grep -rhoE '\b[0-9a-f]{32}\b' .                                   # bare Notion/record IDs
grep -rc 'tomseo' . | grep -v ':0'                                # username (any form)
grep -rn 'Dash' .                                                 # fund vehicle names
grep -rhoE '\(?[2-9][0-9]{2}\)?[ .-][0-9]{3}[ .-][0-9]{4}' . | grep -v 555   # real phones
grep -rhoE '(AKIA[0-9A-Z]{8,}|ghp_|xox[bap]-|sk-[A-Za-z0-9]{15,}|age1[a-z0-9]{20,}|-----BEGIN)' .  # keys
grep -rhoE '\b6880[0-9]{6}\b|CHK-[0-9]{3,}' .                     # bank accounts
```

Beware line-wrapped names: a name split across a line break evades single-line grep (caught
2026-07-22: "Dash\nFund 2"). For any high-value token, also grep its first word alone.

## Step 6: Package

1. Write `/tmp/skill-bundle/README.md` from `references/README-template.md` – update the
   `_N skills included._` count and nothing else unless the architecture description has drifted.
2. `cd /tmp/skill-bundle && zip -rq /tmp/inverted-skill-library.zip . -x '.*'`
3. Copy to `~/Downloads/inverted-skill-library.zip` (overwrite is fine – it's a generated
   artifact) and report the file count + size to Tom.
4. **Only after the gate passed and the zip shipped**: refresh the manifest so the next run
   diffs against this build – `python3 ~/.claude/skills/share-skills/manifest.py update
   /tmp/include_skills.txt`. Never update the manifest on a failed or aborted run.
5. Remind Tom the live map link (https://stack.invertedcap.com) pairs with the bundle for the
   overview-level share.

## Maintenance

- New leak class found during Step 5? Add a rule to `scrub.py` AND a check to the Step 5 gate –
  gates live in code, not in memory.
- New judgment-call category (e.g. a new class of public names)? Update
  `references/redaction-spec.md` – it is the single source of truth for the semantic pass;
  never restate its rules inline in agent prompts beyond pointing at the file.
- Editing scrub.py or redaction-spec.md auto-invalidates the cache via the manifest's rules
  hashes – the next run rebuilds everything. That's correct, not a bug; don't bypass it.
- Cache corrupted or in doubt? Delete `cache/` entirely – the next run is a clean full rebuild.
