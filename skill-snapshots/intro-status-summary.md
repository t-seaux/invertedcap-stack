---
name: intro-status-summary
description: |-
  Summarize the full intro status for a company's round — intros made, intros declined (verbatim pass commentary in blockquotes), still pending — as a Gmail draft to the founder, AND mirror the same content into a `<Company>: Investor Feedback` note on the Opp. Two alias families, ONE identical operation (every run produces both draft and note): intro-status — "intro status for [company]", "summarize intros for [company]", "intro summary for [company]", "where are we on [company] intros", "track intros to/for [company]" (= run the one-time status draft NOW, not recurring monitoring), "intro update for [company]", "draft the intro status email for [company]"; investor-feedback — "summarize investor feedback on/for [company]", "update investor feedback on/for [company]", "capture investor feedback on [company]", "log the investor feedback for [company]", "what feedback have we gotten on [company]", "refresh the investor feedback note for [company]", "did we capture the feedback from the [company] intros". Only an explicit "just the note" / "no email" / "skip the email" suppresses the draft. Always ends by giving Tom the read (key points + verbatim), not a confirmation. Manual-only, always inline; never sends — draft only.

---

# Intro Status Summary

Produce a founder-facing status email covering every intro Tom has been running for one
Opportunity: who's connected, who passed (and their verbatim reasoning — useful market
feedback for the founder), and who hasn't responded yet. Output is a **Gmail draft** (never
sent) addressed to the founder.

Manual-only. One mode.

## Step 1 — Resolve the Opportunity

1. `workspace_search` the Opportunities data source (`collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`)
   for the company name; empty result → SQL `Name LIKE` fallback before concluding absence.
   Same-name rows (e.g. the two "Fair"s) disambiguate on **Fund + Status** — prefer the live
   Active Portfolio / pipeline row.
2. `notion-fetch` the Opp. Collect: `Name`, `Contact` (founder email — the draft's To),
   `🏁 Founder(s)` (fetch for the founder's first name), and all four intro relations:
   `👓 Intros (Qualified)`, `☎️ Intros (Outreach)`, `✉️ Intros (Made)`, `🚫 Intros (Declined / NR)`.
3. Fetch every related People page → Name, Company, Email. (Never include a Tom self-row —
   if one appears, apply the lifecycle contract's cleanup rule and report it.)

## Step 1.5 — Time-bound to the CURRENT campaign

The intro relations accumulate across rounds — a person intro'd for last year's raise must
NOT resurface in this round's status email. Relations carry no per-entry timestamps, so the
campaign window comes from Gmail:

1. From the `in:sent subject:"<outreach subject>"` sweep (Step 2), take every outreach send
   date. **Campaign start** = walk backwards from the most recent send while gaps between
   consecutive sends are ≤ 14 days; the first send of that contiguous burst starts the
   window. (A prior round's sends sit across a months-long gap and never chain in; different
   rounds usually also use different outreach subjects, which separates them further.)
2. **Include a person only if their activity falls inside the window:** outreach sent,
   connect sent, or decline reply received on/after campaign start.
3. **Prior-campaign holdovers are excluded silently** from all three email sections. Count
   them in the run report (`excluded N prior-campaign entries`) — never in the email.
4. **Explicit override wins:** if Tom names a window ("since June", "past month", "for the
   seed round"), use it verbatim. If no clean burst exists (dribbled one-off sends), fall
   back to a 30-day lookback.
5. NR ("no response after k weeks") is measured from that person's outreach send inside the
   current campaign.

## Step 2 — Gather the evidence from Gmail

For each person, the classification comes from the relations; Gmail supplies the color.
Start with ONE sweep of `in:sent subject:"<outreach subject>"` (e.g. `"Fair Pre-Seed"`) —
it surfaces every outreach send in a single call and anchors the per-person searches.
A Gmail MCP "service unavailable" error is transient — retry once before treating any
search as empty.

- **Made**: verify the connect actually went OUT (dual-recipient sent email) — resolution
  flows can move an opted-in person to Made minutes before the connect is sent, so the
  relation alone can briefly lead reality. Optionally note who has already replied.
- **Declined / NR**: search `from:<email> newer_than:120d` (add the outreach subject as a
  secondary term for high-touch contacts — bare from: is noisy), open the reply thread with
  `get_thread` (NEVER judge from `search_threads` snippets — they truncate). Extract the
  **verbatim decline sentences** — the exact words carrying their reasoning, trimmed of
  greetings/sign-offs/quoted history. No paraphrase, no cleanup of their grammar. If the
  person simply never replied (NR), there is no quote — label them `NR` instead.
- **Outreach (pending)**: check the outreach thread for a reply. No reply → plain pending.
  Replied opting in but the connect hasn't gone out yet → annotate `(opted in – connecting now)`.

## Step 3 — Compose the email

**To:** the founder (Opp `Contact`). **Subject:** `[Company] – status on intros FYI`.

Structure (HTML body, Gmail-native `<div>` lines; en dashes in prose, never em dashes;
any links as direct-href anchors). CANONICAL FORMAT — Tom's own edit of the first run
(2026-08-27 Fair draft), reproduce it exactly:

```
**Pending (N):** [Name] ([Firm]), [Name] ([Firm]) (opted in – connecting now), …

**Connected (N):** [Name] ([Firm]), [Name] ([Firm]), …

**Passed (N) – with verbatim feedback:**

[Name] ([Firm]):
▍exact decline sentences from their reply

[Name] ([Firm]) – NR, no response after [k] weeks

[Name] ([Firm]):
▍…

[signature]
```

- **NO quotation marks around the feedback** (Tom's final call, 2026-08-27) — the
  blockquote's vertical line already marks it as a quote; the text inside is bare. Same in
  the plaintext snapshot: `> ` prefix, no `"…"` wrapping.

- **Section order is Pending → Connected → Passed**, exactly as shown.
- **Pending and Connected are single inline paragraphs**: the bolded label
  (`<b>Pending (4):</b>`) followed by a comma-separated run of `Name (Firm)` — NO bullets,
  no one-per-line. The opted-in annotation rides inline.
- **Passed** gets the fully-bolded header `**Passed (N) – with verbatim feedback:**`, then one
  block per decliner: a plain `Name (Firm):` line with the blockquote directly beneath,
  blank line between decliners. NR entries are a single plain line, no blockquote.
- **No opening line and no `Tom` closer** — the body starts at `Pending` and ends after
  the last quote, followed immediately by the signature block.

### Line breaks (EXACT — from Tom's edited draft; blank line = `<div><br></div>` in HTML)

1. Each of Pending / Connected is ONE `<div>` paragraph — never hard-wrap the name run;
   let the client wrap it.
2. Exactly ONE blank line between sections (Pending ⏎⏎ Connected ⏎⏎ Passed header).
3. ONE blank line between the `Passed (N) – with verbatim feedback:` header and the first
   decliner block.
4. NO blank line between a decliner's `Name (Firm):` line and their blockquote — the
   quote sits directly attached beneath the name.
5. Exactly ONE blank line between decliner blocks (i.e. between one blockquote's end and
   the next name line; same for an NR line).
6. NOTHING after the last blockquote — no trailing blank `<div>`, no `Tom` closer. Append
   the RAW verbatim signature fragment (`<div><meta charset="UTF-8">…</div>`) directly
   after the final decliner block — unwrapped (this email type has no `Tom<br>` wrapper,
   unlike intro-connect). The fragment's own leading
   `<br class="Apple-interchange-newline">—` provides the separator spacing.
7. Plaintext snapshot mirrors the same breaks: one blank line between sections and between
   decliner blocks, `> ` quote lines directly under their name line, ends at the last quote.

- The **verbatim quote** renders as a Gmail blockquote (Tom's "vertical line" format):
  `<blockquote style="margin:0 0 0 .8ex;border-left:2px solid #ccc;padding-left:1ex">…</blockquote>`
  directly under the decliner's name line. One blockquote per decliner; multi-sentence
  quotes stay in one block. In the plaintext snapshot, prefix quote lines with `> `.
- Quotes are **verbatim** — their words, their phrasing. Trim to the reasoning only.
- Keep the register terse and factual (this is a status note, not prose). No editorializing
  on the passes — the quotes speak for themselves.
- NR entries get a plain `– NR` note, never a fabricated quote.
- Order within each section: most recent activity first.

## Step 3.5 — Which artifacts does this run produce?

**Every run produces BOTH** — the Gmail draft to the founder (Step 4) and the Notion note
(Step 5). **Phrasing does not change the work** (Tom, 2026-08-31): the intro-status aliases
and the investor-feedback aliases are two doors into one identical operation. Do not infer
a note-only run from feedback wording — if Tom asks to "summarize investor feedback on X",
he still gets the draft.

- **Note-only** fires ONLY on an explicit opt-out: "just the note", "no email", "don't draft
  anything", "skip the email". Nothing else suppresses Step 4.
- **Email-only** is not a mode at all. The note is cheap, it's the whole reason the feedback
  survives the inbox, and skipping it is how the Fair sweep ended up living in one sent
  email.

The draft is never sent, so an unwanted draft costs one deletion while a missing note costs
the record. Default toward producing both.

## Step 4 — Create the draft

Create via `~/.claude/scripts/gmail-create-draft.py` (NEVER the MCP connector for this —
signature fidelity):

```
~/.claude/scripts/gmail-create-draft.py \
  --to "<founder_email>" \
  --subject "[Company] – intro status" \
  --html-body-file /tmp/<scratch>.html \
  --snapshot-text-file /tmp/<scratch>.txt \
  --skill intro-status-summary
```

- The HTML body ends with the RAW **exact** verbatim signature fragment from
  `shared-references/gmail-signature.md`, appended directly after the last decliner block
  with NO `Tom` closer and NO wrapper div (see Line breaks rule 6) — the script's
  signature-fidelity gate will BLOCK any hand-rolled simplification; do not fight it, paste
  the fragment.
- The snapshot text ends at the last quote line (no closer, no signature), with `> ` quote
  prefixes.
- Never send. Report the draft URL plus a one-line count summary
  (`[Company]: N connected · N passed (k with feedback) · N pending`).

## Step 5 — Mirror the email into Notion

Gmail is not a record. Every run ALSO writes the same content to a Notes DB row on the Opp,
so the feedback stays retrievable after the thread scrolls out of the inbox. Added
2026-08-31 after a full Fair sweep — 5 verbatim passes plus 3 substantive connected
reactions — lived only in one sent email.

1. **Find or create** the note. Title: `<Company>: Investor Feedback` (no date — it's an
   append target across the campaign). Dedup by reading the Opp's `✍️ Notes` relation and
   matching that title — NOT Notion search, whose index lags same-run writes.
2. Row properties: `Category` = `Diligence`, `Opportunity` → the Opp. Notes DB data source
   is `collection://e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`. **No page icon** — the page is
   mostly third-party verbatim text (see `shared-references/claude-note-icon.md`).
3. First thing on the page is an italic stamp — `*Last Updated: <Month DD, YYYY>*`, full
   month name, plain body text NOT a heading block (Tom, 2026-08-31). Then the email's three
   sections reproduced exactly — `Pending` → `Connected` → `Passed`, same counts, same
   `Name (Firm)` runs, same verbatim quotes, still no quotation marks. No preamble, no
   "mirrors the email sent to…" scene-setting, no closing synthesis / "Read" / takeaways
   section. The quotes speak for themselves here exactly as they do in the email.
3b. **No spacer blocks.** Do NOT emit `<empty-block/>` between sections — once each person
   is a bullet with an indented quote (see below), the nesting carries the visual
   separation and the spacers just add dead rows. Tom added them, then stripped them
   himself once the bullets landed (2026-08-31); don't reintroduce them on a re-run.
   (Retained for the general case: a bare blank line is a no-op in Notion — empty lines are
   stripped unless written as `<empty-block/>`. That's the tool if a future layout genuinely
   needs air. This one doesn't.)
4. **Re-runs refresh the page in place** — rewrite the three sections to current campaign
   state and bump the stamp. The note is a living current-state mirror, not an append log;
   that's what the `Last updated` stamp commits to, and it's why there are no dated section
   headings. Prior states stay recoverable via Notion version history, and the sent emails
   are the dated record. **Before replacing, read the existing body** and carry forward any
   section or quote Tom added by hand that regeneration wouldn't reproduce.
5. Report the note URL alongside the draft URL.

### Notion quote formatting — Notion ≠ Gmail

A blank `>` line inside a blockquote renders as a visible **empty quote block** in Notion
(caught 2026-08-31 on the Fair note). When one person's verbatim spans multiple paragraphs,
emit each paragraph as its OWN quote block separated by a real blank line carrying no `>`:

```
> first paragraph

> second paragraph
```

NOT `> first` / `>` / `> second`. This applies anywhere a multi-paragraph quote lands in
Notion, not just here.

Two more Notion-side rules (Tom, 2026-08-31):

- **No per-entry `[View thread]` / Gmail deep links.** They clutter the note. The Opp and
  the email thread are one search away.
- **Multi-day feedback from one person gets a date RANGE in the heading** — `Aug 27–31`,
  en dash, never `Aug 27 → Aug 31` and never just the latest date.

### Quotes attach INSIDE the section the person belongs to

The email only quotes decliners, so Pending and Connected are bare name runs there. The
Notion note carries more: anyone in ANY section who gave a substantive read gets their
verbatim, and it sits **under that person's own section** — never in a separate catch-all
bucket at the bottom (Tom, 2026-08-31; the first pass filed Aadik Shekar under a
`Not in the status email` heading when he belongs under Connected).

Per section: the bolded header and its inline `Name (Firm)` run stay exactly as in the
email. Beneath it, **each person who said something is a BULLET, with their quote(s) as
INDENTED CHILDREN of that bullet** (Tom, 2026-08-31 — indent with TABS per the NFM spec):

```
**Connected (10):** Eric Stern (Tiger Global), Samit Kalra (1984 Ventures), …

- Aadik Shekar (POV Ventures) — Aug 27–31:
→   > first message

→   > second message
- Anthony Danon (Rerail) — Aug 27:
→   > …
```

(`→` above = a literal tab. Spaces will NOT nest the quote under the bullet.)

- The bullet line carries a date (or range), unlike the email's bare `Name (Firm):`.
- People with nothing substantive appear only in the roster run — never a bullet with no
  quote under it.
- Multiple messages from one person = multiple child quote blocks under the ONE bullet,
  oldest first. Consecutive `>` lines already render as separate quote blocks, which is
  what's wanted for distinct dated messages — do NOT join them with `<br>`.
- A decliner whose reply had substance beyond the decline itself (Keith Bender's "why this
  wedge, I've tracked Ownwell") gets BOTH quotes under their one bullet. The email may
  carry only the decline sentence; the note keeps everything.
- Escape stray NFM control characters inside verbatim text — a bare `[` in someone's typo
  (`he[s good friends`) must be written `he\[s` or the rest of the quote is swallowed.

## Step 6 — Report the latest back to Tom

Whatever the mode, the chat reply ends with **the read, not a receipt**. Tom's ask is
usually "summarize / update investor feedback on X" — he wants to know what came in, not
that a page was written.

Follow the synthesis shape (see the notes/feedback memory): **a few key points, each
carrying the verbatim quote that earns it**, with sub-points where they help. Never re-list
the full roster — that's what the note is for.

- Lead with what's NEW since the last run (or since the last status email, if there is one)
  — that's the part Tom hasn't seen.
- Surface **convergence explicitly**: when two or more investors land on the same objection
  from different angles, say so and name them. That's the signal worth acting on.
- Separate feedback that's **about the business** from passes that are about fit, timing,
  conflict, or fund mechanics. The latter carry no signal on the company and should be
  labeled as such so they aren't read as market feedback.
- Then the artifact URLs (note, and draft if one was created), one line, at the end.

## Edge cases

- **No declines with commentary** → keep the Passed section with NR-only entries; don't
  invent feedback.
- **Empty section** → omit the section entirely rather than writing "(0)".
- **Founder email missing** → do NOT guess; create no draft, report the gap.
- **Two same-name Opps** → Fund + Status disambiguation (see Step 1); when still ambiguous,
  ask Tom instead of picking.
