---
name: intro-status-summary
description: "Summarize the full intro status for a company's round — intros made, intros declined (with verbatim pass commentary in blockquotes), and still pending response — as a Gmail draft addressed to the founder. Trigger on \"intro status for [company]\", \"summarize intros for [company]\", \"intro summary for [company]\", \"where are we on [company] intros\", \"where are we on intros for [company]\", \"track intros to [company]\", \"track intros for [company]\", \"intro update for [company]\", \"draft the intro status email for [company]\", or any variant asking for a consolidated status of the intro pipeline on one Opportunity — \"track intros\" means run the one-time status draft NOW, not set up recurring monitoring. Manual-only — always trigger inline, no confirmation needed. Never sends; draft only."
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

## Edge cases

- **No declines with commentary** → keep the Passed section with NR-only entries; don't
  invent feedback.
- **Empty section** → omit the section entirely rather than writing "(0)".
- **Founder email missing** → do NOT guess; create no draft, report the gap.
- **Two same-name Opps** → Fund + Status disambiguation (see Step 1); when still ambiguous,
  ask Tom instead of picking.
