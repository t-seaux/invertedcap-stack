---
name: lp-portal-allowlist
description: >-
  Master LP contact list + allowlist gate for BOTH gated surfaces: portal.invertedcap.com (investor
  portal / LP portal / SOI) and lpac.invertedcap.com (the LPAC deck). ONE list, scoped per surface by
  the LP Directory's Category. Mirrors the 🤝 LP Directory Notion DB via two-way sync (notion_sync.py:
  push on JSON edit, pull on Notion webhook). Two-row model: entity rows (Closed + Pending Close + no
  email) and contact rows (email + Parent item linking to entity). Trigger: "add <email> to the portal",
  "give <person> portal access", "give <person> deck access", "give <person> temp access",
  "remove <email>", or "{mode: webhook_pull}" from notion-webhook. Edits ~/code/lp-portal/worker/
  allowlist.json then runs notion_sync.py push + bash deploy.sh.
---

# LP Allowlist (portal / SOI + LPAC deck)

One allowlist gates two surfaces:

- **`portal.invertedcap.com`** — investor portal / SOI. Clerk login (legacy email-OTP behind
  `?auth=otp`); the Worker checks the allowlist AFTER auth. Reads `allowlist.json`, bundled at deploy.
- **`lpac.invertedcap.com`** — the LPAC deck. SOFT gate: enter an allowlisted email, no ownership
  verification. Reads the shared **KV** key (below), so **access changes need no deck redeploy**.

## The surface model — READ THIS BEFORE CHANGING ACCESS

Notion `Category` decides which surface a contact gets. `SURFACES_BY_BUCKET` in `notion_sync.py` is
the one place this is expressed:

| Category | portal / SOI | LPAC deck |
|---|---|---|
| `LP (Active)` — closed LPs | ✅ | ✅ |
| `Other` | ❌ | ✅ |
| `LP (Relationship)` — prospects | ❌ | ✅ |
| `Temporary` (expiring) | ✅ | ✅ |
| *blank / unrecognized* (`DEFAULT_SURFACES`) | ❌ | ✅ |

**Only closed LPs see the portal** — it exposes fund NAV, returns and portfolio marks, and is granted
ONLY by an explicit, recognized Category. **Everyone in the directory sees the deck**, including rows
with no Category yet: the deck is a relationship asset, the SOI is not.

That asymmetry is deliberate — an unrecognized Category defaults OPEN on the deck and CLOSED on the
portal, so a half-entered row is harmless. Never add `"portal"` to `DEFAULT_SURFACES`; it previously
fell through to `active_lp`, which would have handed a blank-Category row the SOI on the next webhook.
Ungated rows are reported by name in the `pull` output.

Category names are load-bearing. `pull` once compared against `"Active LP"` while Notion said
`"LP (Active)"`, matched nothing, and rewrote `allowlist.json` down to the two `Other` rows —
silently locking every LP out of the portal. `CATEGORY_TO_BUCKET` now accepts both spellings, an
unmapped Category is parked with NO access and loudly reported, and `pull` **refuses to write an
empty portal allowlist**. Don't remove those guards.

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
- **`pull`** — Notion → **both surfaces**. Queries the LP Directory, rewrites `allowlist.json` (with
  a `surfaces` array on every contact row), writes the deck's shared KV key, then `bash deploy.sh`.
  Flags: `--no-deploy`, `--no-kv`. Triggered by the notion-webhook on any LP Directory change.

`pull` is the **single writer** of both surfaces. The whole path:

```
Notion LP Directory edit
  → notion-webhook Worker (~/.claude/cloudflare-workers/notion-webhook — routes LP_DIRECTORY_DB)
  → claude-job-queue
  → this skill {mode: webhook_pull}
  → notion_sync.py pull
      ├─ allowlist.json + deploy.sh  → portal.invertedcap.com
      └─ KV allowlist_v1             → lpac.invertedcap.com (no redeploy)
```

Shared KV: namespace `3b9503d7a97040549de9788eeac019d8` (`LPAC_ALLOWLIST`), key `allowlist_v1` —
a JSON array of deck-entitled emails, already scoped and expiry-filtered. The deck Worker binds it
as `ALLOWLIST_KV` and is a **reader only**; it holds a stale bundled list purely as break-glass.

The deck Worker used to run its own Notion webhook + Notion client and write this same key with the
raw unscoped `Master Email` column. That was removed — two writers on one key, and its copy had no
notion of surfaces, so it would have handed prospect LPs deck access on every Notion edit. Don't
reintroduce a second writer.

Notion API token via SOPS-encrypted `~/code/notion-backup/.notion-token.enc.txt`. The KV write shells
out to the locally-authed `wrangler` CLI (same auth `deploy.sh` uses) — no separate CF token.

## Mode B (webhook pull) — invoked by notion-webhook

**The processor runs `fast.py` first; you only run when it exited 10.** fast.py (2026-10-08) runs this
exact block in code – `pull`, then NO_CHANGE → silent, else stdout verbatim → send-alert – and exits 0. It
exits 10 on any error (pull non-zero / timeout, empty stdout, send-alert failure) WITHOUT sending, and leaves
`~/.claude/scheduled-tasks/lp-portal-allowlist/logs/fast-<job_id>.json` (`pull_rc`, `body`, `alert_sent`,
`stderr_tail`). Read it first: if its `body` is non-empty and your own pull prints `NO_CHANGE` (the first pull
already wrote the JSON before failing), that `body` is the real alert – pipe it to send-alert verbatim, unless
the failure was the empty-portal refusal (nothing was written; don't post that body). Then handle the error as
below. Harness: `tests/test_fast.py`.

When the skill is invoked with `mode: webhook_pull` (from notion-webhook on any LP Directory row
change), run:

```bash
python3 ~/.claude/skills/lp-portal-allowlist/notion_sync.py pull
```

The script handles the JSON write, the KV write, and the Worker redeploy — and skips all three when
nothing changed. No further skill logic.

### Alerting — follow this exactly, do NOT improvise

The script's stdout tells you what to do. There is no other alert logic.

- **stdout is `NO_CHANGE`** → **post NOTHING.** End the run silently. Most webhook jobs land here
  (rollup fan-out) and an alert per no-op is pure noise. Never "confirm the sync ran" — that IS the
  noise. Unlike the daily digest agents, this one is event-driven: silence is the correct output when
  nobody's access moved.
- **anything else** → stdout IS the finished Slack body, in the house agent format. Pipe it to
  `send-alert` **verbatim**. Do not reformat, re-summarize, reorder, or add a preamble.

```bash
BODY=$(python3 ~/.claude/skills/lp-portal-allowlist/notion_sync.py pull)
[ "$BODY" = "NO_CHANGE" ] || printf '%s' "$BODY" | bash ~/.claude/skills/send-alert/send.sh
```

The script emits GFM (`[Name](notion-url)`); `send-alert` converts to Slack Block Kit. Title, then one
flat bullet per person — no section headers, no surface-count footer, and nothing printed for a
category with nobody in it. Every bullet's SUBJECT is the **person** — bolded, a live link to their
Notion row — followed by their email, then the access delta with the surfaces (`LPAC Deck`,
`LP Portal`, comma-separated) in trailing parens:

```
🔐 **LP Directory: July 12, 2026**
• **[Jane Doe](https://notion.so/39900bef…)** (jane@newlp.com): Access Granted (LP Portal, LPAC Deck)
• **[Erik Ronning](https://notion.so/38300bef…)** (erik@rengoai.com): Access Changed (LPAC Deck → LP Portal, LPAC Deck)
• **[Nick Paldrmic](https://notion.so/39800bef…)** (nick.paldrmic@waycrosse.com): Access Removed (LPAC Deck)
• **[Ted Clark](https://notion.so/37c00bef…)** (ted@fourbridgepartners.com): Email Updated (ted@fourbridgepartners.com, → ted@fourbridgepartners.com; access unchanged: LPAC Deck)
• **[Ravi Patel](https://notion.so/39a00bef…)**: Needs Attention (no Category set — defaulted to LPAC Deck only)
```

An address change on the SAME Notion row is ONE bullet (Email Updated, or Access Changed with
`email was …` if the surfaces moved too) — never a Removed + Granted pair for one person.

**Never** report Notion page ids, job ids, or raw bucket counts (`71 active_lp · 2 other …`) — the
alert used to do exactly that and it told Tom nothing about who gained or lost access. The script
owns the formatting precisely so this can't drift back; your only job is to pipe it.

Idempotency: the notion-webhook keys LP Directory jobs on a 20s time bucket ALONE (not per page), so
a burst collapses to one job. This matters because the DB's rollup/formula props (Email Rollup,
Sub-Item Email Rollup) mean one contact edit also fires `properties_updated` on its parent entities —
each a different page id. Keyed per-page, one edit produced a burst of identical pulls, each
redeploying the portal and each firing its own alert.

## Mode C — as code (2026-10-04)

The JSON edits below are no longer hand-edits. After the Notion step (create the page first / archive it
first), run ONE of:

    notion_sync.py add    --email E --name N --bucket active_lp|other|relationship [--parent ENTITY] --notion-id ID
    notion_sync.py temp   --email E --name N --notion-id ID [--expires YYYY-MM-DD | --days N (default 7)] [--notes "..."]
    notion_sync.py remove --email E --notion-archived
    (all take [--path allowlist.json] [--dry-run]; they edit ONLY the JSON — no Notion, KV or deploy)

then `notion_sync.py push --deploy` as before. They dedupe by email (case-insensitive → `NO-OP`, exit 0), stamp
`surfaces` from `SURFACES_BY_BUCKET` (the Worker gates on it), require the Notion page id (push PATCHes
`/pages/{notion_id}` — this resolves the old "push currently expects a notion_id" gap: page FIRST, then JSON),
refuse a temp row for anyone with a permanent row (active_lp / other / relationship — change their Category
instead), refuse temp→permanent while a temp row exists, refuse an unknown `--parent`, refuse removing a parent
entity with children or the last portal email, and refuse `remove` until `--notion-archived` asserts the Notion
archive happened. **Exit 0 = done or no-op (report which); 1 = refused → tell Tom the printed reason, change
nothing, do NOT push/deploy; 2 = bad args (incl. a missing `--bucket` — it decides portal/NAV access, so it is never defaulted: ask Tom if unclear).** Harness: `tests/test_allowlist_edits.py` (temp copies only). The
steps below stay as the WHY / ordering.

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
- **Commit (as code, real-time, 2026-10-04):** every write of `allowlist.json` commits + pushes it alone right away —
  `add` / `temp` / `remove` at write time, `pull` right after it rewrites the JSON, and `push --deploy` / `pull` after
  the deploy as a catch-all — message naming who was added / removed / updated. Shared helper
  `~/code/lp-portal/git_log_change.py` (path-scoped, git failure warns, never fails the write). After a bare
  `bash deploy.sh`, run `notion_sync.py commit`. Never sweep other dirty lp-portal files into it; the Monday
  weekly-backup is only a backstop that alerts when a flow missed its commit.
- If Tom gives a name without an email, ask — never guess an LP's email.
- Related: [[lp_portal_fixes_deploy_default]], [[soi_notion_generator]].
