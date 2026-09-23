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

**Shared visual identity:** `references/alert-convention.md` is the single source of
truth for how every alert *reads* — headline grammar (Title Case `Headline —
Subject`, one domain emoji), the domain-emoji + state-glyph taxonomy, and
before→after exemplars. This file (SKILL.md) owns the *transport*; alert-convention.md
owns the *shape*. Every emitter — the `claude` webhook, the `google` Apps Script
bot, and the AI-draft hook — conforms to it. Read it before composing any alert.

---

## Delivery Channel

**Slack `#claude-alerts` channel**, Inverted Capital workspace (team `T0B0B9WK1RN`), posting as the **`claude`** Slack app (distinct from `google`, which is the Apps Script bot, and `tom`, which is the Slack MCP posting as Tom's user).

- **Transport:** Slack Incoming Webhook (POST JSON to `https://hooks.slack.com/services/...`)
- **Webhook URL file:** `~/.claude/skills/send-alert/.webhook_url` (mode 600)
- **Channel:** `#claude-alerts` is fixed at webhook creation time — no `channel_id` needed in the payload.

Why a webhook instead of the Slack MCP: posts show up as the `claude` bot, visually distinguishing Claude's scheduled alerts from both Apps Script automation (`google`) and Claude's interactive MCP messages (`tom`). Also removes the MCP-discovery flakiness that required a subprocess fallback.

### Named channels (`--channel <name>`)

`send.sh --channel <name>` resolves in order:

1. **Webhook file** `.webhook_url_<name>` (same directory, mode 600) — a per-channel incoming webhook, if one was created in the `claude` app.
2. **Token route** — `channels.conf` (`name=CHANNELID` lines, same directory) + the SOPS-encrypted bot token at `~/.claude/.slack-bot-token.enc`; posts via `chat.postMessage` as the same `claude` bot. **This is the default path for new channels** — no webhook creation needed, just add a `name=CHANNELID` line and make sure the bot is a member of the channel (it can create/join channels itself: `channels:manage`, `channels:join`, `groups:write` scopes, added 2026-07-27). Works for private channels.
3. Neither resolves → warn on stderr and fall back to the default `#claude-alerts` webhook — an alert is never dropped on a missing route.

Current named channels:

- `personal-alerts` → `#personal-alerts` (private, `C0BKZ2L0BDK`) — personal-life alerts: word-bank weekly refresher, coop-finances monthly prompt + summaries.

---

## How to send

> **PRE-SEND CONVENTION CHECK — MANDATORY (Tom's standing rule, 2026-09-14).**
> Before piping ANY body to `send.sh`, verify the composed alert against
> `references/alert-convention.md` and **fix any deviation before sending** — never send
> first and correct after. Check, at minimum:
> - **Headline** — exactly one domain emoji from the closed table, *outside* the
>   `<u>**…**</u>` wrapper; Title Case `Headline: Subject` joined by a colon; date
>   suffix only on digests/sweeps; **one blank line after it** (Tom, 2026-09-22 —
>   the send boundary inserts it if you forget, but compose it that way).
> - **State** — glyphs `✓ ⚠ ✗ → ✨` inline only (never a second header emoji, never
>   the `⚠️ ✅ ❌` emoji variants); any action-required line leads, starting with `⚠`.
> - **Footer** — links on their own line *below* the rows, not jammed under the headline.
> - **Progress pings (multi-phase runs)** — follow open → hinge → close, with `→` as the
>   hinge connective and the verdict reserved for the close (see alert-convention's
>   "Progress ping sequencing").
> - **Routing keys** — preserve any documented routing-key token verbatim, and do NOT
>   put a routing token (or its reserved emoji) on an alert that shouldn't route.
>
> If the alert goes out as a **text** instead of Slack, apply the text-lane rules
> (plain render, mandatory blank line after the headline, no markup/Unicode bold,
> ~5-line budget, links as bare URL + trailing `↗`, never message-final).

**Runtime linter — `alert_lint.py` (the automated, self-healing backstop).**
`md_to_blocks.py` runs `alert_lint.py` on every Slack body at the actual send boundary,
so it also covers direct-webhook callers that never touch `send.sh` (e.g.
retro-weekly-summary posting to `#decision-retros`) — a headless `curl` POST is invisible
to Claude Code hooks, but this isn't. It lints the **headline only** (first non-blank
line) for four high-confidence, low-false-positive rules: sentence case, spaced-dash
separator, parenthetical `(YYYY-MM-DD)` date, and a missing `<u>**…**</u>` wrapper. Scope
is the **Slack lane only** (the text lane renders plain, so wrapper checks don't apply).
**Routing-key and verdict headers are protected** (`is_protected()`): the 🟢/🟡/🔴/🧾
leaders and the listener's text-matched tokens (`First Pass Diligence`, `Exit
Distribution`, `Created People DB entry`, `Writeback Review Triage`, `Three-way intro`)
are never flagged and never mutated — so auto-fix can't break reply routing.
**Headline gap (B8, 2026-09-22):** separately from lint, `md_to_blocks.py` always runs
`alert_lint.ensure_headline_gap()` — one blank line between line 1 and the body, on every
alert including protected headers (the headline line itself is untouched). Not logged,
not mode-dependent (only `ALERT_LINT=off` and threaded replies skip it). `send.sh` applies
the same gap per section when buffering into the evening digest. The `google` bot mirrors
it in gmail-webhook `slack-alerts.js` (`_ensureHeadlineGap_`).

Modes via the `ALERT_LINT` env var:
- `fix` **(default)** — deterministically repair the headline in place before sending, so
  every alert is on-convention with **zero human involvement**. Conservative (preserves
  acronyms / proper-cased subjects; protected headers pass through untouched). Violations
  are still logged even when fixed, as the healer's worklist.
- `warn` — detect + append to `lint-violations.log`, still send, no mutation.
- `block` — refuse the send (exit 3). Use where a missing alert is safer than a wrong one.
- `off` — skip. CLI: `python3 alert_lint.py [--fix] < body.md`.

**Root-cause healing — `scheduled-tasks/alert-convention-healer` (weekly Sun 6 PM ET).**
The runtime `fix` keeps *shipped* alerts correct; this agent drains the logged violations
back to their **source templates** so the fix isn't a permanent crutch. It traces each
distinct off-convention headline to the emitting template, rewrites that one headline line
to convention (same transform, placeholders preserved), and posts a quiet `🛠️ Alert Lint:
N Templates Healed` ✓ report — **informational, never an ask**. Skips rather than guesses
when a template can't be confidently located; protected headers are double-filtered. This
is the fully autonomous watch → root-cause → fix → report loop; Tom does nothing.

Pipe the GitHub-markdown body on stdin to the helper script:

```bash
cat <<'EOF' | /Users/tomseo/.claude/skills/send-alert/send.sh
📬 Agent Name: Digest · 2026-04-23

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

**Blank lines emit a `\n\n` spacer.** A blank line in the source markdown widens the visible vertical gap because `md_to_blocks.py` inserts a `rich_text_section` containing `\n\n` between blocks. Exactly one is REQUIRED after the headline (Tom, 2026-09-22 — inserted automatically if missing). Below that, keep tight single-line spacing; add further blank lines only between stacked entities.

**NO column-aligned tables — EVER** (Tom 2026-07-13). Tom reads these alerts on his phone, where Slack
wraps code blocks at ~40 chars instead of scrolling: a 5-column aligned table shatters into interleaved
fragments (row halves on alternating lines, headers orphaned). This includes space-padded tables inside
` ``` ` fences, GFM pipe tables, and anything else that only reads correctly when columns line up.
Tabular data is always **one line per row, label first, ` · ` separating the pairs**:

```
**Pre-Seed (SAFE)** — inv $850,000 · OS 6.80% · FMV $850,000 · 1.00×
**Seed (new)** — inv $728,447 · pending mark
```

Each line survives any wrap width because a wrapped continuation still reads left-to-right as the same
row. Keep units/labels on every value (`inv`, `OS`, `FMV`) — after wrapping, position no longer tells the
reader which number is which.

---

## Per-entity row convention (compact format)

**Bullet rule (Tom, 2026-09-20 — supersedes the 2026-04-27 no-bullets rule):
bullets are the NORM for list rows; a leading functional glyph IS the bullet.**
A plain-text list row (a person in a Monthly Network Refresh section, a doc in
a Materials alert) leads with `• `. A row that already leads with a functional
emoji glyph — the 🧍/🏢 entity emoji on two-line rows below, the 🟢🟡🔴 verdict
circles on `#neg1-sourcing` cards — takes NO additional bullet: the glyph
functionally looks like one, and doubling up is noise.

When an alert lists one or more entities (a company, a person, an opportunity, a deal), every entity is a flat **two-line row** with an optional fingerprint — the leading emoji serves as its bullet:

```
<emoji> <u>**<NAME> | [<LINK_LABEL>](<LINK_URL>)**</u>
**<Status-or-key-label>:** <value>
`[<fingerprint>]`  ← optional, only for entries that need a queue-dedup ID
```

Conventions:

- **No `- ` bullet prefix on emoji-led entity rows** — the leading emoji is the bullet (see the bullet rule above); adding `• ` would double it. Plain-text list rows elsewhere DO take `• `.
- **Emoji is OUTSIDE the `<u>...</u>` and `**...**` wrappers** — Tom does not want emojis underlined. Common emojis: 🏢 (company / opportunity), 🧍 (person / -1 founder), 📬 (digest / report), 🔁 (cycle / refresh), 🔔 (alert), 📈 (signal / movement).
- **Everything after the emoji is wrapped in `<u>**...**</u>`** — name + ` | ` + link. Renders bold AND underlined with the link live.
- **No Description, no Founders, no Round details, no domain link**. Tom prefers compact (entity name + Notion link is enough; he clicks through for context). Avoids extra Notion fetches in the scan logic.
- **Labels (other than Status) are bolded** (`**Status:**`, `**Last touch:**`, `**Decision:**`, etc.) — value follows after a colon and one space.
- **Single-line spacing within the body.** NO blank line between an entity's two lines, NO blank line before a fingerprint footer. If multiple entities are stacked in one alert, separate them with one blank line between entities. The one blank line that is always present is the gap after the alert's headline (line 1) — the normalizer adds it even to a single-entity alert whose first line is the entity row.
- **Fingerprints** (e.g. `[opp:abc12345]`, `[neg1:xyz98765]`) sit immediately under the second line, wrapped in backticks for inline-code styling so they're visually subtle.

**Good (single entity):**

```
🏢 <u>**Acme Corp | [Notion](https://notion.so/acme)**</u>
**Status:** Pass (DNM)
`[opp:abc12345]`
```

**Good (multiple entities — one blank line between, none within):**

```
📬 <u>**Pipeline Sweep: 2 Movers**</u> · 2026-04-25

🏢 <u>**Acme Corp | [Notion](https://notion.so/acme)**</u>
**Status:** Outreach → Connected ✨

🏢 <u>**Beta Co | [Notion](https://notion.so/beta)**</u>
**Status:** Qualified
```

**Bad (legacy patterns to avoid):**

- `🏢 **Acme (acme.com)**` — wrong: domain swallowed inside the bold, no underline, no live link
- `- 🏢 <u>**Acme | [Notion](url)**</u>` / `- **Status:** Pass` — wrong: the 🏢 emoji is already the bullet; a `- `/`• ` prefix doubles it
- Blank line between an entity's two lines — wrong: emits a visible `\n\n` spacer (the only mandatory blank line is the one after the alert headline)
- Including Description / Founders / Round details — wrong: Tom prefers compact (he clicks through to Notion for context)

---

## Summary-only alert format (no entities)

Some alerts are pure summaries with no per-entity headers — just a title line + body. For these, use a header line with emoji + bold (no underline needed since there's no entity to scan past):

```
📬 <u>**Diligence Agent: Evening Sweep**</u> · 2026-04-25
- **Pass notes drafted:** 2
- **Backchannel replies logged:** 3
- **No new feedback outreach**
```

---

## Guardrails

0. **Convention check before every send (Tom's standing rule, 2026-09-14).** Verify the
   composed body against `references/alert-convention.md` and fix any deviation BEFORE piping
   to `send.sh` — never send then correct. See the "PRE-SEND CONVENTION CHECK" gate under
   "How to send" for the checklist.
1. **Webhook channels ONLY.** Default is `#claude-alerts`; a skill may route to a named channel ONLY via `--channel <name>` with a webhook file that exists (see "Named channel webhooks"). Never attempt any other redirect, and do not try the Slack MCP as a fallback (it posts as `tom`, which defeats the bot-identity split).
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
