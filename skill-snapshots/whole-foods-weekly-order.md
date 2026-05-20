---
name: whole-foods-weekly-order
description: >
  Compute Tom's weekly Whole Foods + Amazon order from a cadence spec
  (basket.yaml) and an Apple Notes ad-hoc checklist, then post a
  Slack-bound shopping list to #claude-alerts every Saturday 8 AM ET.
  Tom builds the actual cart manually on amazon.com (Whole Foods Market
  filter, "Buy It Again") using the list — no headless checkout, since
  most WF items lack stable ASINs and Amazon's anti-bot makes auto-add
  risky for a real-money flow.

  Runs automatically every Saturday at 8 AM ET via launchd
  (`com.tomseo.scheduled.whole-foods-weekly-order`). Also triggers when
  Tom says "what's this week's whole foods order", "show me this
  week's groceries", "run wf order", "whole foods order", or any
  variant asking what's in this Saturday's order. Manual triggers
  should pass `--for-date <YYYY-MM-DD>` and OMIT `--commit` so the
  state file does not double-advance ahead of the Saturday scheduled run.
---

# Whole Foods Weekly Order

## What it does

Every Saturday 8 AM ET:
1. Reads `basket.yaml` (cadence-driven SKU spec) and `state.json` (rotation tracker).
2. Reads Apple Notes "Whole Foods Running List" — items Tom has checked off during the week.
3. Builds a markdown shopping list grouped by cadence + category (Fruit, Veggies, Bread, Dairy & Eggs, Protein, Pasta, Drinks, Baby food + Biweekly + Triweekly + Monthly + Ad hoc WF + Ad hoc Amazon).
4. Posts the list to Slack `#claude-alerts` via send-alert.
5. Unchecks the ticked items in the Apple Notes list.
6. Advances rotation state (week_idx, alternation indices).

Tom then taps through the list on amazon.com to build the cart (Whole Foods filter for fresh items, separate cart for non-WF Amazon items like Cetaphil, simplehuman, etc.).

## Files

- `basket.yaml` — canonical spec. Edit here when adding/removing SKUs or changing cadences.
- `state.json` — rotation tracker (week_idx, alternations). Written by the scheduled run; do not hand-edit.
- `build_cart.py` — main entry. Reads basket+state, optionally reads Apple Notes, prints markdown.
- `notes_helper.py` — Apple Notes integration (init / read / clear).
- `venv/` — Python venv with pyyaml.

Scheduled-task scaffolding:
- `~/.claude/scheduled-tasks/whole-foods-weekly-order/run.sh` — launchd entrypoint
- `~/Library/LaunchAgents/com.tomseo.scheduled.whole-foods-weekly-order.plist` — Sat 8am ET cron

## Cadence rules

- **weekly** — every Saturday run
- **biweekly** — when `week_idx % 2 == 0`
- **triweekly** — when `week_idx % 3 == 0`
- **monthly** — when `week_idx % 4 == 0`
- **alternates** — items with an `alternates` list rotate through the alternates; advance index each time fired

Cycle week 0 (the very first run) triggers all four cadences. Subsequent weeks subset.

## Apple Notes ad-hoc checklist

A single note titled **"Whole Foods Running List"** holds all ad-hoc items pre-populated as a checklist. Tom ticks items during the week as he notices he's running low. The Saturday run reads the checked items, adds them to the cart, then unchecks them.

**One-time setup at Tom's Mac:**
1. Approve the macOS Automation permission prompt for python/Terminal → Notes.
2. Run `~/.claude/skills/whole-foods-weekly-order/venv/bin/python3 ~/.claude/skills/whole-foods-weekly-order/notes_helper.py init` to create the note pre-populated.
3. Open the note on macOS/iOS and select all → Format menu → Make Checklist (Notes' programmatic API doesn't expose a "create as checklist" path, so this one-time UI conversion is required).

After step 3, the note persists as a checklist and the read/clear flows work normally.

## Manual triggers

```bash
# Dry-run (does not advance state, does not touch Apple Notes)
~/.claude/skills/whole-foods-weekly-order/venv/bin/python3 \
  ~/.claude/skills/whole-foods-weekly-order/build_cart.py --for-date 2026-05-23

# Same but read Apple Notes for ad-hoc items (still no state advance)
~/.claude/skills/whole-foods-weekly-order/venv/bin/python3 \
  ~/.claude/skills/whole-foods-weekly-order/build_cart.py --notes --for-date 2026-05-23

# Live run (the Saturday cron does this) — reads Notes, builds, advances state, unchecks Notes
~/.claude/skills/whole-foods-weekly-order/venv/bin/python3 \
  ~/.claude/skills/whole-foods-weekly-order/build_cart.py --notes --commit
```

## Editing the spec

- Adding/removing items → edit `basket.yaml` directly.
- Adding a new ad-hoc item → edit `basket.yaml`'s `adhoc_whole_foods` or `adhoc_amazon` list AND add a checkbox row to the Apple Note manually (or re-run `notes_helper.py init --overwrite` to regenerate from scratch — destructive, loses any current checks).
- Changing cadence (e.g. weekly → triweekly) → move the item between the top-level YAML keys.

## Why no auto-checkout

Most WF items in this spec are non-ASIN'd (generic names like "Organic Banana, 1 each"). Amazon's "add to cart" deep-link API requires ASINs. Even for ASIN'd items, Amazon's anti-bot defenses make headless cart building risky for a real-money weekly flow. The list-+-Slack-deep-link pattern matches Tom's MadeMeals skill for the same reason.
