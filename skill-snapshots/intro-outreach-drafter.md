---
name: intro-outreach-drafter
description: |-
  Draft first-touch intro-request notes — the "would you be open to connecting with [X]?" ask Tom sends to gauge interest BEFORE any formal double-opt-in. Purpose-agnostic: customer, investor, advisor, strategic partner, or hiring chat, on behalf of a company (portfolio or pipeline) OR a person Tom is championing. Per recipient: resolve their existing People-DB row (never auto-creates; missing → asks Tom), draft in Tom's intro-outreach voice as a Gmail DRAFT (never send), and add them to the Opp's 👓 Intros (Qualified) when the subject maps to an Opportunity; intro-outreach-agent moves them to ☎️ Outreach on send. Modes: (B) Targeted/Enqueued — gmail-webhook's handleOppHostIntroOptIn fires this when an Opp's own contact replies YES to an intro Tom offered them to someone in his network ("would love to intro you to Liam, up for it?" → "yes please"); drafts the ask to the OTHER person (the target) so their opt-in can be gathered too. Distinct trigger from intro-note-processor (which finds intro offers by scanning call TRANSCRIPTS, not Gmail replies) but identical output — kept in one skill, not duplicated. (C) Manual. Trigger on: "draft an intro note to [names] for [company/person]", "draft outreach to [X] about [Y]", "ask [X] if they'd connect with [Y]", "[founder] wants intros to [names]", "draft a note introducing [company] to [potential customer/investor]", "[person] said yes to the [target] intro, draft the note", or any variant asking for first-touch intro-request notes. Composes with talent-scan, coinvestor-recommender, network-scan, add-to-contacts. NOT intro-draft-agent (double-opt-in connect email, post BOTH opt-ins), NOT talent-scan (candidate sourcing) — this is the drafting layer. Always trigger inline.

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

**Never sends.** Creates Gmail drafts only (EF7). All email formatting follows `shared-references/email-formatting.md`.

## People DB Guardrails (MANDATORY)

Canonical rules and incident: `shared-references/people-db-guardrails.md` – read it before any People DB lookup or write. It overrides anything else in this skill. The People DB syncs both ways with Tom's iPhone Contacts, so a wrong write here lands on his phone.

1. **Never create a People entry — text Tom and wait for his 👍.** If a person isn't found after BOTH the scoped People DB search and the workspace search, run `python3 ~/.claude/skills/shared-references/people_db_ask.py --name "<Name>" --source-skill <this skill> [--email] [--li] [--company] [--opp-id --opp-name --relation] --context "<why>"` — it texts Tom "🧍 People DB: <Name> … ⚠ Not in the People DB yet … 👍 to add to People DB" and stages the payload (idempotent: re-runs never double-text). Tom's 👍 makes sms-listener §4b create the row via add-to-contacts and finish the skipped relation write. Until then, skip every Notion write for that person, in every mode (manual, scheduled, webhook). In reports, list them as "🧍 texted for 👍: <Name>".
2. **Never modify contact fields on an existing People page** (Email, Name, Company, Role, LI, phone) unless Tom explicitly asks. This skill writes only Opportunity-side relation fields. If a recipient's email doesn't match the People page they resolved to, flag the mismatch – never copy the email over.
3. **Match on identity, not proximity.** Resolve a person by exact email, exact name + company, or exact LinkedIn URL. Never infer a person from a shared Opportunity relation (e.g. the Opp's Qualified roster), first name alone, or the closest fuzzy search hit.
4. **Ambiguous → flag, don't guess.** Multiple candidates or conflicting keys → flag with the candidates and skip all writes for that person.


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
- **add-to-contacts** — creates a recipient's People row, ONLY after Tom approves that person (Step 2).
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
- **The blurb / bio:** the "About [X]" content, sourced per `shared-references/email-formatting.md`
  EF6 (📚 callout → founder-supplied → compose from the Opp body + call notes). When composing, find
  the call notes by title-matching `<Name>` in the Notes DB if the `✍️ Notes` relation is empty. Only
  ask Tom when there's no Opp and no notes at all.
- **Optional per-person context** — how Tom knows them / why relevant. If absent, use the default
  firm-relevance line (STYLE); never fabricate history.

## Modes

- **Mode B — Targeted (Enqueued).** Fired by `gmail-webhook`'s `handleOppHostIntroOptIn` when an
  Opp's own contact replies "yes" to an intro Tom offered them to someone in his network. See "Mode B
  — Opp-host opt-in" below; it resolves the subject (the Opp) and the recipient (the target) itself,
  then falls through to Steps 1-5 below unchanged.
- **Mode C — Manual.** Tom names the recipient(s) and subject directly. Standard entry point, Steps
  1-5 below.

## Mode B — Opp-host opt-in (Targeted, Enqueued)

Tom, 2026-09-21: "when I ask if someone wants to chat with someone and they say yes, you draft the
note I need to send to the other person to get the double opt-in." **Not the same trigger as**
`intro-note-processor` (which finds intro offers by scanning Notion AI **call transcripts**) — this
fires off a **Gmail reply**. Different signal, identical output (a Step 3 draft + Step 4 Qualified
entry), so it lives here rather than as a separate skill — keep the drafting logic in ONE place.

**Args** (from `gmail-webhook`):
```json
{
  "messageId": "<Gmail message id of the Opp-host's reply>",
  "threadId": "<Gmail thread id>",
  "senderEmail": "<Opp-host's email — matched an Opp's Contact property>",
  "oppId": "<Notion Opportunity page id>",
  "oppName": "<Opportunity title, for logging/alerts>",
  "qualifiedPersonIds": ["<People DB page id>", "..."]
}
```
`qualifiedPersonIds` is the Opp's FULL `👓 Intros (Qualified)` roster at enqueue time, not necessarily
just the person this reply names — resolved below.

**B1 — Confirm it's a real opt-in AND extract the named target, deterministically first.** Fetch
`messageId` (plain text) and Tom's prior message in the same thread (`SENT` label). Tom, 2026-09-21:
"I'll be explicit about intro'ing someone, so in the outreach note I'll almost always include a name
and where that person works" — his offer email is reliably explicit, so parse it as data before
reaching for any semantic judgment:
1. **Deterministic extract — links first, they're the strongest signal.** Tom, 2026-09-21: "I'll
   also often link the person's LI and company website" — pull the plain-text message's `href`s (or
   fetch `htmlBody` if the plain-text extraction dropped them) and check any anchored on the target's
   name / near the intro-offer language for a `linkedin.com/in/...` URL, and any anchored on the
   company name for a company-site URL. A `linkedin.com/in/<slug>` hit is near-certain identity on
   its own — no roster match needed to trust it.
2. **No usable link → fall back to the `<Name> @ <Company>` text shape.** Regex/pattern the SENT
   message for `<Name> @ <Company>` / `<Name> at <Company>` / `<Name> (<Company>)` near intro-offer
   language ("connecting with", "intro you to", "speaking to", "chatting with") — the same shape his
   real sends use verbatim ("Liam @ Level Ventures", "TJ Agnihotri @ FourBridge Partners").
3. **Check the extracted identity (LI URL, or `<Name>`/`<Company>`) against `qualifiedPersonIds`'s
   People-DB rows first** (fetch `Name`/`LI`/`Company` for each) — a match there is the fastest path,
   not the only valid outcome (see B2: no roster match is normal, not a failure). The roster is a
   list of CANDIDATES, not evidence (People DB Guardrails rule 3 — the 2026-09-22 Katie Fifer / Eric
   Grant incident was exactly a roster-proximity match). A candidate resolves only on identity: exact
   LI URL; or first name + a Company field that matches the stated company (Tom, 2026-09-21: "Liam at
   Level Ventures — should make it pretty darn clear that this is Liam Shalon") — the company match is
   REQUIRED, "no other roster candidate shares that first name" is NOT sufficient on its own. Company
   mismatch, a blank Company, or two same-first-name candidates both matching is ambiguous → B2's
   `target-ambiguous` exit.
4. **Only fall back to LLM judgment** (reading B1's opt-in reply for affirmative language, and
   loosely matching offer phrasing to any candidate name) when NEITHER the link check (1) nor the
   `Name @ Company` text shape (2) extracts anything at all — a genuinely atypical, non-explicit
   offer. A link or `Name @ Company` shape that simply isn't on the roster is NOT this case — that's
   B2's off-roster path, still deterministic.
Confirm the REPLY (`messageId`) is a clear affirmative either way — "yes", "sounds great", "I'd love
that", "happy to chat" — not a deferral ("maybe later") or decline. Not clear → log
`not-a-clear-optin`, exit 0, no draft. No offer (explicit or otherwise) findable in Tom's message →
log `no-offer-found-in-thread`, exit 0. If Tom's message is instead a reference-check ask (asking the
contact FOR a reference, not offering one) — not an intro offer at all; log `reference-ask-not-intro`,
exit 0.

**B2 — Resolve the target person, whether or not they're already on `qualifiedPersonIds`.**
`qualifiedPersonIds` is a head start, not a precondition — the JS-layer gate no longer requires it to
be non-empty (a separate scan may not have staged this person yet; don't depend on one having run).
- **B1 matched a roster candidate** → that's the recipient. Fetch `Email`/`LI`, skip to Step 2's dedupe.
- **B1 extracted a name/LI-URL/company but it's NOT on the roster (new target, nothing staged yet)**
  → this is normal, not an error. Run Step 2's full dedupe → enrich-if-missing exactly as Mode C
  does: scoped + `workspace_search` of the People collection for the extracted name / LI URL. **No
  existing row → do NOT create one** (this path is unattended): log `target-not-in-people-db`, list
  the person via `people_db_ask.py` (texts Tom a 👍 card with the extracted LI URL / company,
  `--relation "👓 Intros (Qualified)"`, the Opp, and `--then "draft the intro-outreach note (Step 3)"` if the draft was skipped), note "🧍 texted for 👍" in the ✍️ alert, and skip the Step 4 Qualified write for them. The draft itself may still be created
  if the target's email is known from the thread — it needs no People page — but say so in the alert.
- **B1 found nothing at all, or two genuinely ambiguous candidates** → log `target-ambiguous`, exit
  0. Do not guess; a wrong target drafted to a stranger is the exact failure this gate exists to
  prevent.
- **Run Step 2 point 4's self-relation guard before finalizing.** Highest-risk path for it: Mode B
  resolves the Opp FROM the sender's contact email, so a loose B1 extraction that lands back on the
  Opp-host's own name (rather than the target they actually named) would otherwise self-loop — draft
  an "intro" to the person the Opp already IS. Guard, don't skip.

**B3 — Intro subject = the Opp** (`oppId`/`oppName`) — run Step 1's company-subject path using this
Opp directly (no `notion-search` needed, you already have the ID). **Relevance line:** reuse whatever
hook Tom's own offer email (B1) already gave — it usually states it ("figured you two would have a
lot to compare notes on given X").

**B4 — Then run Steps 2 (recipient already resolved per B2, just finish dedupe/enrich if that's
still pending) through 5 unmodified.** Same About-block rule, same `gmail-create-draft.py` call, same
mandatory ✍️ alert (never pass `--no-alert`), same post-create `list_drafts`-confirms-exactly-one-draft
check. **Step 4's Qualified write is a REAL write here, not a no-op** — this is the mechanism that
stages the target when nothing else has yet (union with existing relation, per Step 4's rule); confirm
it landed (readback) before Step 5 reports done.

**B5 — Log and exit.** Run-log entry: `oppId`, `oppName`, resolved target name + id, draft URL. No
separate Slack alert beyond Step 3's ✍️ draft-created ping.

**Manual equivalent (Mode C phrasing that means the same thing):** "TJ said yes to the Liam intro,
draft the note" / "[Opp] is up for chatting with [target], draft the ask" — skip B1/B2's email
detection, resolve Opp + target by name instead, run B3-B5.

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
   A hit counts only on identity (exact email, exact full name + matching Company, or exact LI URL) —
   never the closest fuzzy hit, never "they're already on this Opp's roster". Two candidates →
   ask Tom.
2. **Not found (after scoped + workspace search) → text Tom and stop for that person.** Check Gmail
   and the local LinkedIn network cache (`~/.claude/scripts/network_cache.db`) for an email / LI URL so
   the card shows who you think it is, then run `shared-references/people_db_ask.py` (with the Opp +
   `--relation "👓 Intros (Qualified)"` when there is one, plus `--then "draft the intro-outreach note (intro-outreach-drafter Step 3) unless one already exists"`). Create nothing; Tom's 👍 on the text makes
   sms-listener §4b run `add-to-contacts` and the Qualified append.
3. **Email used for the draft vs. the People page.** Draft to the `Email` on the resolved People page.
   If Tom (or the thread, or ContactOut) gives a different address, draft to the one Tom confirms but
   **never write it onto the People page** — report "⚠️ Email mismatch – [Name]'s People page has [X],
   draft uses [Y]; not updating the page." A blank page Email is reported the same way, not filled.
   Personal-email-only from ContactOut → flag it in the report so Tom can confirm.
4. **Self-relation guard (every recipient, every mode) — the Opp's own Contact/Founder is never a
   valid recipient for ITS OWN Opp.** Tom, 2026-09-21: "you can't add TJ's People DB entry to his -1
   TJ opportunity, that doesn't make logical sense" — you don't introduce someone to themselves. Check
   the resolved recipient's People-DB id/email against the Opp's `Contact` property and `🏁 Founder(s)`
   relation; a match on either means this recipient IS the Opp, not a target of it — drop them
   silently from this Opp's batch (log `self-relation-skip`) and do NOT draft or write Qualified for
   them on this Opp. Same person is a perfectly valid recipient on a DIFFERENT Opp (e.g. TJ Agnihotri
   is a legitimate target on Rengo's or Caplight's Opp — he's their contact, not the Opp's own). This
   generalizes the founder-self-relation rule already enforced elsewhere in the intro pipeline
   (`feedback_intro_pipeline` memory, rule #4) to this skill's every entry point, Mode B's
   contact-email-derived resolution most of all — that path resolves an Opp FROM a contact email, so
   a parsing slip that lands back on the same person is the exact failure this guard exists to catch.

### Step 3 — Draft the note (per recipient)
Read `writing-style/intro-outreach/STYLE.md` (+ `EDIT_PATTERNS.md` + `VOICE_EXAMPLES.md`) and follow it
exactly, including the correct **variant by intro type** (portco / non-portco company / person, and the
purpose-tuned relevance line). One Gmail **draft** per recipient via
`~/.claude/scripts/gmail-create-draft.py --skill intro-outreach-drafter` (never send) — the helper
creates the draft AND writes the draft-feedback snapshot atomically, so Tom's edits feed
`writing-style/intro-outreach/EDIT_PATTERNS.md` via diff mode. Write the HTML body + a plain-text
snapshot body (strip tags; no signature) to scratch files, pass `--html-body-file` /
`--snapshot-text-file`; exit code 0 = success, non-zero = that recipient's draft failed (don't fall
back to a snapshot-less MCP draft). Draft rules:
- Subject per STYLE: `Intro to [Subject] ([plain-English of what they do])?` (e.g. "Intro to Rengo (AI for investment firms)?").
- Body formatting (HTML, links, About block, signature) per `shared-references/email-formatting.md`.
  The plain-text snapshot excludes the signature (that keeps the edit-diff clean) — the draft itself
  carries it.
- Use Tom's per-person context line if supplied; otherwise the default relevance line.

### Step 4 — Reflect on Notion (log to Qualified) — only when an Opp exists
If the intro subject resolved to an Opportunity, add every recipient's People page ID to that Opp's
`👓 Intros (Qualified)` relation via `notion-update-page` (`update_properties`). **Union with the existing
relation** — fetch current value first and append; never overwrite. Recipients already in `☎️ Outreach`,
`✉️ Made`, or `🚫 Declined / NR` for this Opp are past this stage → don't re-add; note as skipped.
If there is **no Opp** (person subject, or company not in the pipeline), skip the relation write and say so
in the report. No People page is ever written by this skill (no creates without Tom's approval, no
field edits ever).

### Step 5 — Report
- **Drafts created** — per person: recipient (company), email used, subject.
- **People rows** — matched existing (with link); "🧍 texted for 👍" list; any ⚠️ email mismatches.
- **Logged to Qualified** on [Opp] — or "no Opp for this subject, relation skipped."
- **Flags** — personal-email-only recipients; missing per-person context (offer to personalize); anyone skipped.
- **Handoff reminder** — "Left to send. The Qualified → ☎️ Outreach flip is **automatic** once you send:
  the `outreach-detector` / `pipeline-sent-detect` webhook fires in real time on your sent mail, with the
  daily `intro-outreach` sweep as backstop. Tom is NOT the trigger — no action needed." (Only flip a status
  in-session yourself if Tom explicitly mentions a send during a live conversation, as an immediate expedite;
  never imply he must tell you.)

## Step 6 — Batch status: re-read state before EVERY report (HARD GATE)

Once a batch is in flight, Tom will keep working the drafts in Gmail while the session continues.
**Any later statement about that batch — "sent", "still open", "waiting to hear" — must be re-derived
from live state, never from what happened earlier in the conversation.** Before reporting:

1. **Query the Opp's four intro relations** (`👓 Qualified` / `☎️ Outreach` / `✉️ Made` /
   `🚫 Declined / NR`). The reply webhooks move people within minutes; the relation is fresher than
   anything in context.
2. **Search each recipient's thread and read the REPLIES**, not just the send.
3. **Lead the report with terminal outcomes** — declines and opt-ins — quoting the person's own words.
   A decline changes who is left and what coverage remains; it is the most valuable line in the report.

⛔ **A draft disappearing from the drafts list is NOT evidence it was sent.** It means sent, OR
hand-edited into a new draft id (Gmail remints as `s:…` on UI edit), OR discarded. Never report a send
from an absence — confirm in sent mail.

⚠️ Failure this gate exists to prevent (Fair, 2026-08-26): four drafts left the list, the drafter
reported them "sent," and never opened a thread. Helen Min and Rapha Danilo had both already passed
and were already filed under 🚫 Declined by the webhook. Tom had to point it out — his instruction was
that he should never have to. **Catching a decline is the drafter's job, not Tom's.**

## Edge cases
- **Missing subject-person LinkedIn** → link only the company; leave the name unlinked and note it.
- **No blurb/bio anywhere** → compose per EF6; ask Tom only when there's no Opp and no notes.
- **Recipient already in Qualified for this Opp** → just (re)draft; don't duplicate the relation entry.
- **Multiple recipients** → one draft each, one batched Qualified update, one report.
- **Recipient not a clean identity match** → ask for LI URL/email; do not guess (hard rule).
- **Wrong/updated email after drafting** → the connector can't EDIT a draft (update_draft flattens the
  signature), so create a fresh draft via `gmail-create-draft.py` — then **trash the superseded draft
  yourself** via the Gmail connector's `trash_message` (pass the old draft's messageId), per Tom's
  superseded-draft auto-delete rule (revised draft of the SAME email → trash the old one). Never leave
  stale copies in Drafts or tell Tom to delete them manually. (Corrected 2026-09-11: this line previously
  claimed the connector can't delete; `trash_message` works fine on a draft's message.)
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
