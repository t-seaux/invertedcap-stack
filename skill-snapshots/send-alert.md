---
name: send-alert
description: >
  Centralized alert delivery skill. Posts a summary message to Tom's Slack
  `#claude-alerts` channel via the `claude` incoming webhook, so scheduled
  skills show up under a distinct bot identity (not as `tom`, which is the
  Slack MCP, and not as `google`, which is the Apps Script bot). All
  scheduled skills and agents reference this skill for alert delivery
  instead of maintaining their own inline config.

  This skill is not triggered directly by the user. It is referenced by other
  skills (pass-note-drafter, investor-update, pipeline-agent, intro-agent,
  research-agent, diligence-agent, portfolio-agent, run-all, etc.) at the
  point where they need to notify Tom. Those skills compose the message
  content — this skill defines HOW it gets delivered.
---

# Send Alert (Centralized)

Every scheduled agent or skill that needs to notify Tom MUST follow this file rather than maintaining its own inline notification config.

---

## Delivery Channel

**Slack `#claude-alerts` channel**, Inverted Capital workspace (team `T0B0B9WK1RN`), posting as the **`claude`** Slack app (distinct from `google`, which is the Apps Script bot, and `tom`, which is the Slack MCP posting as Tom's user).

- **Transport:** Slack Incoming Webhook (POST JSON to `https://hooks.slack.com/services/...`)
- **Webhook URL file:** `~/.claude/skills/send-alert/.webhook_url` (mode 600)
- **Channel:** `#claude-alerts` is fixed at webhook creation time — no `channel_id` needed in the payload.

Why a webhook instead of the Slack MCP: posts show up as the `claude` bot, visually distinguishing Claude's scheduled alerts from both Apps Script automation (`google`) and Claude's interactive MCP messages (`tom`). Also removes the MCP-discovery flakiness that required a subprocess fallback.

---

## How to send

Pipe the GitHub-markdown body on stdin to the helper script:

```bash
cat <<'EOF' | /Users/tomseo/.claude/skills/send-alert/send.sh
📬 AGENT NAME — 2026-04-23

**Section**
- **Company** — [Title](https://example.com) — key point
- Another item
EOF
```

The helper:
1. Reads the webhook URL from `.webhook_url` (sibling file, mode 600).
2. Converts GFM markdown to Slack mrkdwn (`**bold**` → `*bold*`, `[text](url)` → `<url|text>`, `# Heading` → `*Heading*`, `-`/`*` bullets → `•`).
3. POSTs `{"text": "<converted body>"}` to the webhook.
4. On success, prints `ok` to stdout. On failure, prints an error to stderr and exits non-zero.

**One message per skill invocation.** Do not loop; compose the full summary in one body and pipe it once.

---

## Formatting (write GFM — the helper converts to Slack Block Kit)

Calling skills compose their bodies in **standard GitHub-flavored markdown**; `md_to_blocks.py` (invoked by `send.sh`) converts to Slack Block Kit rich_text blocks before POSTing. Do NOT hand-write Slack mrkdwn (`*bold*`, `<url|text>`) — it ships as literal text or renders italic.

Supported transforms:

| GFM input | Slack output |
|---|---|
| `**bold**` or `__bold__` | bold (recursive wrapper — composes with inner links/underline/etc.) |
| `<u>text</u>` | underlined (recursive wrapper — composes with inner bold/links/etc.) |
| `[text](url)` | live link |
| `# H1` / `## H2` / `### H3` (up to `######`) | bolded heading (single line) |
| `- item` or `* item` at line start | indented bullet item (real list) |
| `_italic_` | italic |
| `` `inline code` `` | inline code |
| ` ``` ` fenced code blocks | preformatted block |
| `> blockquote` | blockquote |

**Recursive wrappers (`**...**`, `<u>...</u>`):** these compose with each other and with leaf patterns. `<u>**Acme** [link](url) more text**</u>` renders the entire span as both bold AND underlined with `link` as a live link. Order doesn't matter — `<u>**x**</u>` and `**<u>x</u>**` produce identical output. Updated 2026-04-25 (previously the bold pattern swallowed inner markdown literally).

**Blank lines emit a `\n\n` spacer.** A blank line in the source markdown widens the visible vertical gap because `md_to_blocks.py` inserts a `rich_text_section` containing `\n\n` between blocks. For tight single-line spacing, omit blank lines. Add them only where you genuinely want extra vertical separation.

---

## Per-entity row convention (compact format, bullets removed 2026-04-27)

When an alert lists one or more entities (a company, a person, an opportunity, a deal), every entity is a flat **two-line row** with an optional fingerprint — **no bullet glyphs**:

```
<emoji> <u>**<NAME> | [<LINK_LABEL>](<LINK_URL>)**</u>
**<Status-or-key-label>:** <value>
`[<fingerprint>]`  ← optional, only for entries that need a queue-dedup ID
```

Conventions:

- **No `- ` bullet prefix on entity rows.** Both lines are bare paragraphs; they render as parallel left-aligned lines in Slack/Beeper without the `•` glyph. (Updated 2026-04-27 — Tom finds the bullets visually noisy.)
- **Emoji is OUTSIDE the `<u>...</u>` and `**...**` wrappers** — Tom does not want emojis underlined. Common emojis: 🏢 (company / opportunity), 🧍 (person / -1 founder), 📬 (digest / report), 🔁 (cycle / refresh), 🔔 (alert), 📈 (signal / movement).
- **Everything after the emoji is wrapped in `<u>**...**</u>`** — name + ` | ` + link. Renders bold AND underlined with the link live.
- **No Description, no Founders, no Round details, no domain link**. Tom prefers compact (entity name + Notion link is enough; he clicks through for context). Avoids extra Notion fetches in the scan logic.
- **Labels (other than Status) are bolded** (`**Status:**`, `**Last touch:**`, `**Decision:**`, etc.) — value follows after a colon and one space.
- **Single-line spacing throughout.** NO blank lines between the two lines, NO blank line before a fingerprint footer. If multiple entities are stacked in one alert, separate them with one blank line between entities.
- **Fingerprints** (e.g. `[opp:abc12345]`, `[neg1:xyz98765]`) sit immediately under the second line, wrapped in backticks for inline-code styling so they're visually subtle.

**Good (single entity):**

```
🏢 <u>**Acme Corp | [Notion](https://notion.so/acme)**</u>
**Status:** Pass (DNM)
`[opp:abc12345]`
```

**Good (multiple entities — one blank line between, none within):**

```
📬 **Pipeline sweep — 2026-04-25**
🏢 <u>**Acme Corp | [Notion](https://notion.so/acme)**</u>
**Status:** Outreach → Connected ✨

🏢 <u>**Beta Co | [Notion](https://notion.so/beta)**</u>
**Status:** Qualified
```

**Bad (legacy patterns to avoid):**

- `🏢 **Acme (acme.com)**` — wrong: domain swallowed inside the bold, no underline, no live link
- `- 🏢 <u>**Acme | [Notion](url)**</u>` / `- **Status:** Pass` — wrong: bullets dropped 2026-04-27, lines are bare paragraphs now
- Blank line between the two lines — wrong: emits a visible `\n\n` spacer
- Including Description / Founders / Round details — wrong: Tom prefers compact (he clicks through to Notion for context)

---

## Summary-only alert format (no entities)

Some alerts are pure summaries with no per-entity headers — just a title line + body. For these, use a header line with emoji + bold (no underline needed since there's no entity to scan past):

```
📬 **Diligence Agent — 2026-04-25 evening sweep**
- **Pass notes drafted:** 2
- **Backchannel replies logged:** 3
- **No new feedback outreach**
```

---

## Guardrails

1. **`#claude-alerts` ONLY.** The webhook is scoped to that channel — do not attempt to redirect. Do not try the Slack MCP as a fallback (it posts as `tom`, which defeats the bot-identity split).
2. **Do NOT use iMessage, Beeper, Signal, or any other notification channel.**
3. **One message per skill invocation.** Each calling skill produces at most one alert.
4. **Failure handling:** if `send.sh` returns non-zero, append a one-line failure note to the calling skill's `audit-log/` directory (`[<ISO timestamp>] ERROR: send-alert failed — <stderr>`). Do NOT retry the POST, and do NOT fall back to another channel.
5. **Don't leak the webhook URL.** It lives in `.webhook_url` (mode 600). Never echo it, paste it into logs, or commit it. If it's compromised, rotate via https://api.slack.com/apps (the `claude` app → Incoming Webhooks → regenerate).
6. **Body-hash dedupe (1h).** `send.sh` writes a sentinel at `/tmp/send-alert-sent/<sha256(body)>` after a successful POST and silently exits 0 on any second call with the same body hash for 1 hour (sentinels GC'd at 24h). This absorbs orchestrator double-fires (e.g. a session that chains heredoc + `compose | send.sh` then re-pipes the same payload). Do NOT remove the dedupe — it protects every skill that uses send-alert against a recurring failure mode. To legitimately re-send the same body within the hour (test re-fire), `rm /tmp/send-alert-sent/<hash>` first. If duplicates still appear in Slack, check whether two distinct sessions are firing the same skill (cron + manual run, or duplicate launchd plists) — that's a different problem the dedupe also catches.

---

## How Calling Skills Should Use This

The calling skill is responsible for:

1. **Composing the message body** (header + content) per its own format, in GFM.
2. **Piping that body into `/Users/tomseo/.claude/skills/send-alert/send.sh` via stdin** (heredoc, `echo`, or file redirect — all work).
3. **Checking the exit code.** On failure, log per Guardrail 4.

The calling skill's notification section should say something like:

> Compose the alert body as GFM markdown: [skill-specific format]. Pipe it to `/Users/tomseo/.claude/skills/send-alert/send.sh` on stdin. On non-zero exit, log the failure per `send-alert/SKILL.md` guardrail 4.

This ensures that if the delivery channel, bot identity, or tool ever changes, only `send.sh` + this file need to be updated — calling skills stay the same.
