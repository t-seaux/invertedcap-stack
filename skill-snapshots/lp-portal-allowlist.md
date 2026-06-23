---
name: lp-portal-allowlist
description: >-
  Master LP contact list + allowlist gate for portal.invertedcap.com (investor portal / LP portal /
  SOI page — same Cloudflare Worker, ONE allowlist). Mirrors the 🤝 LP Directory Notion DB via
  two-way sync (notion_sync.py: push on JSON edit, pull on Notion webhook). Two-row model: entity
  rows (Closed + Pending Close + no email) and contact rows (email + Parent item linking to entity).
  Trigger: "add <email> to the portal", "give <person> portal access", "give <person> temp access",
  "remove <email>", or "{mode: webhook_pull}" from notion-webhook. Edits ~/code/lp-portal/worker/
  allowlist.json then runs notion_sync.py push + bash deploy.sh.
---

# LP Portal Allowlist

`portal.invertedcap.com` (investor portal / LP portal / SOI) is gated by an email-OTP login. Only
addresses on the allowlist can request a code. THE source of truth is `allowlist.json` in code; the
🤝 LP Directory Notion DB is the human-editable mirror.

## Source of truth

`~/code/lp-portal/worker/allowlist.json` — three categories. Each row is either an LP **entity**
(has `closed` + maybe `pending_close`, no email or with email if single-row LP) or a **contact**
(has `email` + optional `parent` pointing to an entity by name).

- **`active_lp`** — array of entity + contact rows. Permanent access.
- **`other`** — same shape; non-LP permanent allowlist (currently empty).
- **`temporary`** — `{name, email, expires (YYYY-MM-DD), added, notes, notion_id}`. Auto-denied
  after end-of-day UTC on `expires` (Worker enforces in `isAllowed()`).

Every row carries `notion_id` (the Notion page UUID) so the sync can update existing rows in place.

The Worker (`index.js`) reads only `email` and filters out entity rows (no email). Submitted emails
are lowercased + trimmed.

## Notion mirror — 🤝 LP Directory

Database ID: `21e00bef-f4aa-80a4-815e-000bc981305d`. Schema:
- `Name` (title)
- `Email` (text, blank on entity rows except single-row LPs)
- `Category` (select: Active LP / Other / Temporary)
- `Closed` (number, dollar — the entity's commitment closed)
- `Pending Close` (number, dollar — anticipated additional close)
- `Expires` (date, only Temporary)
- `Parent item` / `Sub-items` (native sub-items relation — entity = parent, contacts = children)

Toggle on "Sub-items" in the DB's ••• menu if the tree view isn't already enabled.

## Sync infrastructure

`~/.claude/skills/lp-portal-allowlist/notion_sync.py` — two commands:

- **`push`** — JSON → Notion. PATCHes every page by `notion_id` with current JSON state. Called
  automatically by manual-mode add/remove/temp flows after JSON edits. Run with `--deploy` to
  also `bash deploy.sh` the lp-portal Worker.
- **`pull`** — Notion → JSON. Queries the LP Directory DB, rewrites `allowlist.json`, and runs
  `bash deploy.sh`. Triggered by the notion-webhook on any LP Directory row create/update/archive.

Notion API token via SOPS-encrypted `~/code/notion-backup/.notion-token.enc.txt`.

## Mode B (webhook pull) — invoked by notion-webhook

When the skill is invoked with `mode: webhook_pull` (from notion-webhook on any LP Directory row
change), run:

```bash
python3 ~/.claude/skills/lp-portal-allowlist/notion_sync.py pull
```

That's it. The script handles JSON write + Worker redeploy. No further skill logic.

Idempotency: notion-webhook buckets job keys to the minute, so a burst of Notion edits collapses
to one pull job. Subsequent edits within the same minute are skipped by claude-job-queue dedup.

## Mode C (manual) — add a row

When Tom says "add <email>" / "give <person> portal access" / similar:

1. Parse address(es). Handle multiple in one request.
2. Classify each:
   - LP in the fund → `active_lp`. If they belong to an existing entity (Cameron Holdings, Level
     Ventures, etc.), set `parent` to the entity's name.
   - Non-LP permanent → `other`.
   - "Temp / temporary / time-limited" → `temporary` (see Mode C-temp below).
   - When unclear, default to `other` and confirm.
3. Read `allowlist.json`. Dedupe by email — skip silently if already present and report.
4. Append the new record (no `notion_id` yet — the push step creates the Notion page and the
   pull will fill in `notion_id` on the next sync).

   For new rows that don't yet have a `notion_id`, the push currently expects one — so the
   simplest path is: append to JSON, then call `notion_sync.py pull` (which discovers any new
   rows you created manually in Notion, or rebuilds JSON from Notion). For brand-new emails not
   yet in Notion, create the page first via `mcp__claude_ai_Notion__notion-create-pages` on
   the LP Directory DB (data_source_id `21e00bef-f4aa-80a4-815e-000bc981305d`), capture the
   returned `id`, then add the record to JSON with that `notion_id`.

5. Run `python3 ~/.claude/skills/lp-portal-allowlist/notion_sync.py push --deploy`.
6. Confirm to Tom: what was added (and to which category/entity), expiry if temp, deploy Version ID.
   Do NOT send a test login code unless Tom asks.

## Mode C (manual) — temp access

Trigger: "give X temp / temporary / time-limited access" (or similar).

- **Default window: 1 week** (today + 7 days). Otherwise compute from Tom's phrasing ("through
  Friday", "until 2026-07-01", "for the day", "2 weeks", etc.).
- Append to `temporary`: `{ "name": "...", "email": "...", "expires": "YYYY-MM-DD", "added":
  "YYYY-MM-DD", "notes": "...", "notion_id": "..." }`. Lowercase email, ISO dates. `notes` is
  optional but useful ("Justin viewing SOI", etc.).
- Create the Notion page first (Category=Temporary, Email set, Expires set), capture `id` into
  JSON, then push.
- If the email already exists in `active_lp` or `other`, flag the conflict — they have
  permanent access already.

## Mode C (manual) — remove

Trigger: "remove <email>" / "revoke X" / etc.

1. Locate the record in JSON (any category).
2. Archive the Notion page: PATCH `/v1/pages/{notion_id}` with `{"archived": true}` (see the
   one-off Python snippet pattern).
3. Delete the JSON record.
4. Run `notion_sync.py push --deploy` (or just `bash deploy.sh` — push is a no-op for the
   removed row).
5. Confirm. Note that any existing session stays valid until it expires (24h).

## Notes

- Explicitly-invoked maintenance task: do NOT ask permission for the edit + `bash deploy.sh` +
  `notion_sync.py push`.
- Deploy uses `bash deploy.sh` from `~/code/lp-portal/worker`.
- If Tom gives a name without an email, ask — never guess an LP's email.
- Related: [[lp_portal_fixes_deploy_default]], [[soi_notion_generator]].
