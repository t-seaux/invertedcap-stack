---
name: investing-style
description: >-
  Refresh Tom's public-facing "Investing Style" Google Doc from the founder-taste corpus. Re-derives the
  Markets / Teams / Products / Models bullets from PILLARS.md, the decision ledger, retros, LP letters and
  investment memos; diffs the result against the last published version; and rebuilds the Doc in the
  diligence-qa visual canon (logo, HEADING_1 title, italic HEADING_2 sections, justified bullets with bold
  "that…" lead-ins). Founder-facing artifact — publication-safety gates are mandatory. Trigger on "refresh
  my investing style", "update the investing style doc", "regenerate investing style", "what's changed in my
  investing style", "bump the investing style doc", or any request to update that doc. Manual only — no
  scheduled or webhook entry point.
---

# Investing Style — refresh

Maintains one artifact: **[Investing Style](https://docs.google.com/document/d/1CegObODNYmHZWHwszxaL8dGG5T37Dl80M__oIHhEOQw/edit)**, a founder-facing statement of what Tom invests in and why. It exists to show how many reps he has run and how opinionated he is — so it must stay current, and it must never leak anything confidential.

**Local files** (`~/.claude/skills/investing-style/`):

| File | Role |
|---|---|
| `manifest.json` | Doc id, version, thesis, provenance template, counts at last refresh |
| `content.json` | **Source of truth for the doc body** — `{section: [[label, text], ...]}` |
| `build_doc.py` | Renders `content.json` into the Doc; diffs; bumps the version |

> ⚠️ **Never hand-edit the Google Doc and expect it to survive.** `build_doc.py` clears everything below the logo and re-renders. Edit `content.json`, or tell Claude the change and let it land there. Same rule as `PILLARS.md` / `pillars.json`.

---

## Workflow

### Step 1: Read current state

```bash
cat ~/.claude/skills/investing-style/manifest.json
python3 ~/.claude/skills/investing-style/build_doc.py --dry-run
```

Note `version`, `last_refreshed`, and the counts. Everything below is scoped to *what has changed since that date*.

### Step 2: Re-derive from the corpus

**First, pull the Drive-sourced material** — it has no scheduled sync by design (Tom's call, 2026-07-27), so it is only as fresh as the last time someone asked:

```bash
python3 ~/.claude/skills/draft-feedback/lp_letter_cache.py sync
python3 ~/Projects/invertedcap-skills/first-pass-diligence/memo_cache.py sync
python3 ~/.claude/skills/draft-feedback/lp_letter_cache.py bridge-memos
python3 ~/.claude/skills/draft-feedback/processor.py --sweep
```

This matters most here: a quarterly refresh that skips it will silently miss the quarter's LP letter, which is the single most likely thing to have changed since the last run.

Then follow `founder-taste/SKILL.md` **Query Mode** — its source table, two-axis weighting, output format, and publication-safety rules all apply. Do not re-derive those rules here; read them there.

Refresh the counts for the provenance line:

```bash
sqlite3 ~/.claude/data/decision_ledger.db \
  "select count(*) from decisions;
   select count(*) from decisions where decision='invested';
   select count(*) from decisions where why is not null and why not like '%BACKFILL%';"
```

Then check each source for movement since `last_refreshed`:

- **`founder-taste/PILLARS.md`** — pass-side Pillars refresh automatically on every sent pass note, so this is the most likely thing to have moved. New or promoted Pillars are candidate new bullets.
- **New investments** in the ledger — invest-side Pillars are a one-time extraction and do NOT self-update. If there are new `invested` rows since the last refresh, their retros need reading directly.
- **New LP letters / investment memos** in `writing-style/letters-and-memos/VOICE_EXAMPLES.md` and the Drive memo folder.
- **`DECISION_RETROS.md`** for reasoning that contradicts an existing bullet. A bullet the corpus no longer supports should be removed, not left to rot — that is the failure this whole system was built to avoid.

### Step 3: Propose the diff — HARD GATE

Write the proposed content to a temp JSON in the same shape as `content.json`, then:

```bash
python3 ~/.claude/skills/investing-style/build_doc.py --content /tmp/proposed.json --dry-run
```

**Surface the diff to Tom and stop.** Show added, removed, and changed bullets with the evidence behind each, and say which source drove it. This artifact is public-facing and opinionated in his name — it does not update unattended. Wait for his call.

### Step 4: Build

```bash
python3 ~/.claude/skills/investing-style/build_doc.py --content /tmp/proposed.json
```

This renders the Doc, writes `content.json`, bumps the minor version, and stamps `last_refreshed`. A no-op rebuild does not inflate the version. Use `--no-bump` for a formatting-only fix.

Report the diff and the doc link.

---

## Quarterly refresh (human-gated)

`com.tomseo.scheduled.investing-style-quarterly` fires the **15th of Jan / Apr /
Jul / Oct at 10:00** — mid-month rather than the 1st so the prior quarter's LP
letter, typically written in the weeks after quarter end, is in the corpus by
the time we read it.

Runner: `~/.claude/scheduled-tasks/investing-style-quarterly/run.sh`, prompt at
`prompt.md` in the same directory. It runs Steps 1-3 only, writes
`proposed.json` + `proposed-summary.md` into the skill directory, and posts the
summary to Slack. **It never publishes.**

That is enforced in code, not by the prompt: the runner exports
`INVESTING_STYLE_PROPOSE_ONLY=1`, and `build_doc.py` refuses any non-dry-run
invocation while it is set (exit 2). A model deviating from the prompt still
cannot publish. To apply an approved proposal, run interactively:

```bash
python3 ~/.claude/skills/investing-style/build_doc.py \
    --content ~/.claude/skills/investing-style/proposed.json
```

A quarter with no material change is a valid outcome — the run says so and
writes no proposal rather than manufacturing movement.

## Gates

- **Publication safety is mandatory** — the rules in `founder-taste/SKILL.md` Query Mode. In short: never attribute a pass to a named company; no fund economics, LP names, round terms, ownership, or valuations; no unflattering reads on named founders. Publish the argument, cut the referent.
- **Quotes from confidential sources.** LP letters and investment memos are marked CONFIDENTIAL. Tom's positions are his to publish; verbatim sentences lifted from those documents are a separate question. Default to stating the position in his voice rather than quoting. His own coinages (*the non-obvious slog*, *the liminal zone*) are fine.
- **No firm layer.** Audience is founders. Cost of capital, AUM dynamics, fund size, ownership targets, graduation math, and portfolio construction stay out — that is the LP argument.
- **Keep the "that…" construction.** Sections are plural nouns (Markets, Teams, Products, Models); every bullet's bold lead-in completes the stem "I look for …". Passes fold into their section as `— a pass` rather than living in a separate rejection list.
- **Formatting is not this skill's to invent.** It comes from `diligence-qa/canonical_spec.py`, which reads the measured profile off Tom's approved reference doc. If the chrome looks wrong, fix it there, not here.

## Known gaps to state when they bear on a refresh

- **All three Pillar sides now self-update.** Pass notes feed on send; retros and -1 decisions feed via DB triggers within seconds; LP letters and memos feed on the on-demand Drive sync in Step 2. What still does NOT self-update is this doc — by design.
- **`decisions.outcome` is empty for all rows** — the doc describes Tom's taste, it cannot yet claim the taste is *right*.
- **The memo corpus never prices** regulatory risk, key-person risk on solo founders, entry valuation, or co-founder split. Relevant if a refresh is tempted to claim comprehensiveness.
