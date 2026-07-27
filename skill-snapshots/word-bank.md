---
name: word-bank
description: Maintain Tom's personal word bank — words he's encountered but doesn't know precisely and wants to eventually use in his writing. Two modes. (C) Manual add — when Tom says "add to word bank [word]", "add [word] to word bank", "word bank: [word]", "wordbank [word]", or hands over one or more words with intent to save them, look up a concise definition + a natural example sentence and append the entry to word_bank.md. (A) Weekly refresher — when Tom says "word bank refresher", "refresh my word bank", "what words did I add", "word bank recap", or a scheduled weekly run fires, list the words added that week and show a grounded retrofit for each (a real sentence from Tom's investment memos + LP letters, rewritten to use the word). Corpus is Tom's final investment memos + LP letters, synced from Drive (source of truth) into a local cache by refresh_corpus.py — incremental (only new/changed docs downloaded), final-only (drafts + docs edited in the last 7 days excluded), citing the exact source doc. Zero-new-words cost gate. Scheduled weekly Sunday 9 PM ET to Slack #claude-alerts. The bank is a single local Markdown file (no Notion/DB). Always trigger inline — no confirmation needed to add.
---

# Word Bank

Tom's running list of words he's met but doesn't know precisely, and wants to internalize and use in his writing.

**Bank file (source of truth):** `~/.claude/skills/word-bank/word_bank.md` — a single local Markdown file. Newest entries at the top. No Notion, no database (per design decision: simple file only).

**Design decisions (locked):**
- Storage: simple local Markdown file only.
- On add: auto-include a concise definition **and** an example sentence — never just capture the bare word.

---

## Mode C: Manual add

Trigger: "add to word bank [X]", "add [X] to word bank", "word bank: [X]", "wordbank [X]", or Tom hands over one or more words/phrases with intent to save.

### Steps

1. **Parse the input.** Extract the word(s) or short phrase(s). Tom may pass several at once (comma/line separated) — process each. If Tom included the sentence or context he saw it in, keep that verbatim for the `Seen in:` field.

2. **Build the entry** for each word:
   - **Part of speech** — `(pos)`, listing **every** part of speech the word can function as, comma-separated: `(n.)`, `(v.)`, `(adj.)`, `(adv.)`, e.g. `(n., adj.)` — not just the primary sense.
   - **Definition** — one concise sentence. Precise, plain, no dictionary padding. If the word has multiple senses, give the sense that fits Tom's context (or the most common sense, and note the others in half a clause only if genuinely useful).
   - **Example** — one natural sentence that shows the word in use. Prefer a sentence rooted in Tom's world (venture, funds, founders, writing, markets) when it reads naturally — the point is for him to be able to reuse it. Otherwise a clear general sentence. Do **not** reuse the `Seen in` sentence as the example; write a fresh one.
   - **Added** — today's date, `YYYY-MM-DD` (use the current date from context).
   - **Seen in** — only if Tom gave the source/context. Omit entirely otherwise.

3. **Dedup.** Read `word_bank.md` first. If the word is already present, don't add a duplicate — tell Tom it's already in the bank (with its date) and, if his new definition/context differs, offer to update the existing entry instead.

4. **Prepend** the new entry (or entries) directly under the `<!-- entries below -->` marker so newest sits at the top. **Exact entry shape** (3 lines; use this format verbatim for both the file and the reply):

   ```markdown
   🟩 **perspicacious (adj.)** having keen insight; sharply perceptive.
   *Her **perspicacious** read of the cap table caught the option overhang everyone else had missed.*
   _Added 2026-07-25_ · _Seen in: a review calling the author "perspicacious about power"_
   ```

   Format rules:
   - Line 1: `🟩` green-box bullet, then **`word (pos)` bolded together (bold runs through the closing paren)**, then the definition on the **same line, not bolded**.
   - Line 2: the example sentence **fully italicized**, with the target **word bold-italic** inside it (`**word**` nested in the `*…*`).
   - Line 3: `_Added YYYY-MM-DD_`, plus ` · _Seen in: …_` only if Tom gave context (drop the whole `Seen in` part otherwise).

5. **Respond with the full entry** for each word added — the exact 3-line block from Step 4 (same content written to the file), so Tom learns it on the spot. No checkmarks or status glyphs. Do not collapse to a bare one-liner; the definition + example are the point. Keep any surrounding commentary to a minimum — no lecture.

---

## Mode A: Weekly refresher (grounded retrofit)

Trigger: "word bank refresher", "refresh my word bank", "what words did I add [this week]", "word bank recap", or (once wired) a scheduled **weekly** run.

**Cadence:** weekly — scheduled **Sunday 9:00 PM ET** via LaunchAgent `com.tomseo.scheduled.word-bank-refresher` (wrapper at `~/.claude/scheduled-tasks/word-bank-refresher/`), delivered to Slack `#claude-alerts` via send-alert. On-demand runs (Tom asks in chat) render inline instead of posting to Slack.

The refresher isn't a plain re-list. It's a **grounded retrofit**: for each word added this week, find a real sentence from Tom's own polished writing — his **investment memos and LP letters** — and show how the word could have sharpened it (before → after, in his own voice).

### Corpus — memos + LP letters, Drive is the source of truth (kept fresh in code)

The retrofit corpus is Tom's **final** investment memos + LP letters. Drive is the source of truth: the folders `Investment Memos/` and `LP Letters/` (My Drive root). A helper keeps a local cache in sync so the corpus is never stale and unchanged docs are never re-downloaded:

`~/.claude/scheduled-tasks/word-bank-refresher/refresh_corpus.py`
→ writes `corpus_cache/<file_id>.txt` (plain text of each doc) + `corpus_cache/manifest.json` (`{file_id: {name, type, modifiedTime}}`).

The helper (SA-authenticated, no MCP) reconciles the cache against Drive every run, keyed on `modifiedTime` — so a newly written memo appears automatically, and nothing is re-fetched when nothing changed. **Final-only gates (in code):** it skips drafts (title contains draft/WIP/rough/scratch) and any doc edited within the last 7 days (`WORDBANK_STABILITY_DAYS`, still in flight — a memo Tom just started is not final), and hard-excludes templates. Degrades to the existing cache if Drive is unreachable.

### Steps

1. **Word check FIRST (cost gate).** Read `word_bank.md`, collect entries whose `Added` date is within the trailing 7 days (on-demand with no window: trailing 7 days; if empty, offer trailing 30 — ask, don't auto-widen). **If zero new words, STOP** — report "no new words this week," touch nothing else, exit. Most weeks with no adds cost nothing.

2. **Sync + load the corpus.** Only if there are new words. Ensure the corpus is fresh, then read it:
   - The scheduled wrapper's `run.sh` already runs `refresh_corpus.py` before this session. **For an on-demand (in-chat) run, run it yourself first:** `python3 ~/.claude/scheduled-tasks/word-bank-refresher/refresh_corpus.py`.
   - Read the doc texts from `corpus_cache/*.txt` and load `corpus_cache/manifest.json` to map each file to its **exact doc name** (e.g. "Signal7 - Investment Memo", "Inverted Capital I: Q1 2026 Letter"). Those names are what you cite in the retrofit.

3. **Match, don't force.** Match all of the week's words against the corpus texts. For each word, find at most one real sentence where it would have slotted in naturally, and **record which doc it came from** (manifest name). If a word has no honest fit anywhere in the corpus, **say so and carry it forward** — never fabricate. A crafted in-voice sentence is acceptable only when explicitly labeled as an illustration (not presented as something Tom wrote).

4. **Render** in this form factor:

   ```markdown
   🟩 **Word Bank — week of Jul 21–25** · 3 new

   **synoptic (adj.)** — a broad, at-a-glance view of a whole.
   ↳ You wrote (Q1 2026 LP Letter): "Here's a quick rundown of where the fund stands."
   → "Here's a **synoptic** view of where the fund stands."

   *(portmanteau — no natural slot in the memos/letters; carried forward.)*
   ```
   - One block per in-window word: bolded `word (pos)` + short definition recap, then the grounded retrofit. **Always name the exact source doc** (from the manifest) in the `↳ You wrote (<doc name>): "…"` line — never a vague "a memo" or "an LP note". → line shows the rewrite with the word bold.
   - Close with a one-line note of any words carried forward (no fit).

### Cost guardrails (do not remove)

- Zero new words → exit before syncing or reading the corpus.
- Corpus is the **local synced cache** (`corpus_cache/*.txt`); `refresh_corpus.py` only downloads new/changed docs, never re-fetches unchanged ones.
- No Gmail, no texts — the corpus is Tom's final memos + LP letters only.
- Run on **Sonnet** — retrofit matching with Tom as the review gate, no framework construction (per the model-tier framework).

---

## Notes

- The bank file can be mirrored to Drive on request — not done by default.
