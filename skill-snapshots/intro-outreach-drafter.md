---
name: intro-outreach-drafter
description: >
  Draft first-touch intro-request notes — the "would you be open to connecting with [X]?" ask Tom
  sends to someone in his network to gauge interest BEFORE any formal double-opt-in. Purpose-agnostic:
  the intro can be to a potential customer, investor, advisor, or strategic partner (or a hiring-related
  chat), on behalf of a company (portfolio or pipeline) OR a specific person Tom is championing. For each
  recipient: resolve/create their People-DB row, draft the note in Tom's intro-outreach voice as a Gmail
  DRAFT (never send), and — when the intro subject maps to an Opportunity — reflect it on Notion by adding
  them to that Opp's 👓 Intros (Qualified). Tom reviews and sends; the existing intro-outreach-agent then
  moves them Qualified → ☎️ Outreach on send. Manual-only (Mode C). Trigger on: "draft an intro note to
  [names] for [company/person]", "draft outreach to [X] about [Y]", "ask [X] if they'd connect with [Y]",
  "[founder] wants intros to [names]", "draft a note introducing [company] to [potential customer/investor]",
  or any variant asking to draft first-touch intro-request notes. Composes with talent-scan (candidates for
  hire intros), coinvestor-recommender (investors for a deal), network-scan, add-to-contacts (create rows).
  NOT intro-draft-agent (double-opt-in connect email, post opt-in), NOT talent-scan (candidate sourcing) —
  this is the drafting layer. Always trigger inline.
---

# Intro Outreach Drafter

Drafts the **first-touch intro-request note** Tom sends to his network to gauge interest before a
formal intro. Parallel to `feedback-outreach-drafter`: this skill DRAFTS; the existing
`intro-outreach-agent` DETECTS the send and advances the pipeline.

**Purpose-agnostic.** An intro can be to a potential **customer**, **investor**, **advisor**,
**strategic partner**, or a **hiring-related** chat — and can be on behalf of a **company**
(portfolio or pipeline) or a specific **person** Tom is championing. The note mechanics are
identical across all of these; only the ask-line framing and the one relevance line flex (see
`writing-style/intro-outreach/STYLE.md` → "Variants by intro type").

**Never sends.** Creates Gmail drafts only. Tom reviews and hits send.

## Where this sits in the intro lifecycle

```
[THIS SKILL: draft note + (if an Opp exists) log to Qualified]  →  Tom sends  →  intro-outreach-agent
   👓 Intros (Qualified)  ────────────────────────────────────────────────────►  ☎️ Intros (Outreach)
                                                                                   →  ✉️ Made / 🚫 Declined
```

Canonical lifecycle rules: `shared-references/intro-lifecycle-contract.md` (contract wins on conflict).
This skill only ever writes to **`👓 Intros (Qualified)`**. It never moves anyone to Outreach — that
happens on send and is owned by `intro-outreach-agent`. Do not duplicate that logic here.

## Composes with (don't reimplement)
- **talent-scan** — sources candidates for a hiring intro (JD → shortlist). Feed its picks here to draft.
- **coinvestor-recommender** — surfaces investors for a deal; draft the outreach here.
- **network-scan** — general "who do I know…" queries that produce recipient names.
- **add-to-contacts** — creates/enriches a recipient's People row (Step 2).
- **intro-outreach-agent** — detects the send, moves Qualified → ☎️ Outreach.
- **intro-draft-agent** — the later double-opt-in connect email (different stage).

## Notion data model
- **Opportunities** — `collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
  - `Name`, `🏁 Founder(s)` (relation → People), `Website`
  - `👓 Intros (Qualified)` — relation → People (**the only property this skill writes**)
  - `☎️ Intros (Outreach)` / `✉️ Intros (Made)` — later stages, do NOT touch
- **People** — `collection://1715ce8f-7e54-43e2-bbcd-17a5e50cb8c9` — `Name`, `Email`, `LI`, `Company`, `Role`

## Inputs Tom provides
- **Recipient(s):** one or more people to reach out to (may come from talent-scan / coinvestor-recommender / network-scan).
- **The intro subject** — who/what is being introduced. One of:
  - a **company** (portfolio, pipeline, or one Tom rates) — resolve to its Opportunity if one exists;
  - a **person** (a founder raising, a candidate, someone in Tom's orbit) — no Opp needed.
- **The intro purpose** — customer / investor / advisor / partner / hire — tunes the relevance line.
- **The blurb / bio:** the "About [X]" content to paste verbatim. If it's a company already in the
  pipeline, pull the latest "Company Overview" from its Opp page body; else ask Tom for it.
- **Optional per-person context** — how Tom knows them / why relevant. If absent, use the default
  firm-relevance line (STYLE); never fabricate history.

## Workflow (Mode C — Manual)

### Step 1 — Resolve the intro subject
- **Company subject:** `notion-search` the Opportunities collection. If found, fetch the Opp; read
  `Name`, `🏁 Founder(s)` (fetch each founder for name + LinkedIn URL), `Website`, current
  `👓 Intros (Qualified)`, and the blurb source in the page body. Note whether it's a **portfolio**
  company (Status = Active Portfolio / Committed / Portfolio: Follow-On) — that gates the "one of my
  portfolio companies – I led the pre-seed" parenthetical. If no Opp exists, treat as a non-pipeline
  subject: get the founder LinkedIn + blurb from Tom, and there will be no relation to write in Step 4.
- **Person subject:** get their name, a one-line descriptor, and (if linking) their LinkedIn from Tom.
  No Opp → Step 4 skips the relation write.

### Step 2 — Resolve each recipient (dedupe → enrich-if-missing)
For each named person, in order:
1. **Dedupe** — `notion-search` the People collection with `content_search_mode: "workspace_search"`
   and the full name. Exact title match → use that row (read `Email`, `LI`).
2. **Not found → resolve identity, then create.** Check Gmail and the local LinkedIn network cache
   (`~/.claude/scripts/network_cache.db`). Common names need a disambiguator — if you can't confidently
   resolve who they are, **ask Tom for a LinkedIn URL or email** rather than guessing (never risk a wrong
   email on an outbound intro). With a LinkedIn URL, invoke **`add-to-contacts`** (Mode C subroutine) to
   enrich + create the row. That skill owns the People-DB creation gate and ContactOut caching.
3. **Email quality:** if ContactOut yields only a personal email (no work email), use it but **flag it**
   in the report so Tom can supply/confirm the right address before sending. (In practice Tom often has
   the correct address — surface the one on file and invite a correction.)

### Step 3 — Draft the note (per recipient)
Read `writing-style/intro-outreach/STYLE.md` and follow it exactly, including the correct **variant by
intro type** (portco / non-portco company / person, and the purpose-tuned relevance line). One Gmail
**draft** per recipient via `create_draft` (never send):
- Subject per STYLE: `Intro to [Subject] ([plain-English of what they do])?` (e.g. "Intro to Rengo (AI for investment firms)?").
- `htmlBody`: `<div>`-line HTML; ask line links the subject person to LinkedIn and the company to its site;
  the verbatim blurb at the bottom under an italic `About [Subject]` header, first sentence bold, company
  name linked. Plaintext `body` fallback too.
- No signature — Gmail auto-appends it below the About block.
- Use Tom's per-person context line if supplied; otherwise the default relevance line.

### Step 4 — Reflect on Notion (log to Qualified) — only when an Opp exists
If the intro subject resolved to an Opportunity, add every recipient's People page ID to that Opp's
`👓 Intros (Qualified)` relation via `notion-update-page` (`update_properties`). **Union with the existing
relation** — fetch current value first and append; never overwrite. Recipients already in `☎️ Outreach`,
`✉️ Made`, or `🚫 Declined / NR` for this Opp are past this stage → don't re-add; note as skipped.
If there is **no Opp** (person subject, or company not in the pipeline), skip the relation write and say so
in the report — the People rows still get created/deduped in Step 2.

### Step 5 — Report
- **Drafts created** — per person: recipient (company), email used, subject.
- **People rows** — created (with link) vs. matched existing.
- **Logged to Qualified** on [Opp] — or "no Opp for this subject, relation skipped."
- **Flags** — personal-email-only recipients; missing per-person context (offer to personalize); anyone skipped.
- **Handoff reminder** — "Left to send. The Qualified → ☎️ Outreach flip is **automatic** once you send:
  the `outreach-detector` / `pipeline-sent-detect` webhook fires in real time on your sent mail, with the
  daily `intro-outreach` sweep as backstop. Tom is NOT the trigger — no action needed." (Only flip a status
  in-session yourself if Tom explicitly mentions a send during a live conversation, as an immediate expedite;
  never imply he must tell you.)

## Edge cases
- **Missing subject-person LinkedIn** → link only the company; leave the name unlinked and note it.
- **No blurb/bio anywhere** → ask Tom; don't invent one.
- **Recipient already in Qualified for this Opp** → just (re)draft; don't duplicate the relation entry.
- **Multiple recipients** → one draft each, one batched Qualified update, one report.
- **Recipient not a clean identity match** → ask for LI URL/email; do not guess (hard rule).
- **Wrong/updated email after drafting** → the connector can't edit or delete a draft; create a fresh draft
  to the correct address and tell Tom which stale draft(s) to delete manually.
- **Redrafting after a bounce or a send (new address, resend, etc.)** → if Tom already SENT a version, that
  sent copy — not the base template — is the source of truth. Pull it from Sent mail (`search_threads
  in:sent` → `get_message` for the full `htmlBody`) and reproduce HIS content verbatim (his reconnect
  preamble, personalizations, PS), only swapping the recipient address and cleaning Gmail's redirect-wrapped
  links back to direct URLs. Fix only unambiguous typos, and say which you touched. Never regenerate from the
  template when an edited/sent version exists.

## Notes
- **Model / invocation:** manual, interactive (or subroutine). Gmail draft + Notion write behind a human
  send-gate → Sonnet-class if ever pinned; no scheduled `run.sh` (Mode C only, no sweep/webhook).
- **Reads** `writing-style/intro-outreach/STYLE.md` for voice/format (canonical form = Tom's 2026-07-22
  Rengo→Chad edit).
