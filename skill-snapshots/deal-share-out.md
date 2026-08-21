---
name: deal-share-out
description: >-
  Share a deal from Tom's OWN pipeline outbound with other venture firms, as a Gmail draft (never
  sends). Modes: (B1) Webhook — Tom reacts 👣 :foot: to a #decision-retros card; slack-retro-webhook
  enqueues via claude-job-queue, the [opp:] fingerprint resolves the Opp, close-loop reply posts
  in-thread. (B2) Webhook — AUTO on pass: notion-webhook fires when a non-(-1), non-FO Opp's
  Status flips to Pass (Met) / Pass (DNM) / NR / Missed; the draft just lands in Drafts, silent.
  (C) Manual — Tom asks in conversation; an explicit ask overrides the -1/FO exclusion (that
  gates only the B2 auto-trigger). Recipients = the Distribution List (pilot: Primary
  deal-agent + investments@fika.vc) in Bcc, minus any member ANYONE at whose firm sourced the
  deal (firm-wide exclusion: resolve each Source(s) person's firm; a Primary-sourced deal never
  goes to the Primary agent, a Fika-sourced deal never to the Fika inbox). Given an
  Opportunity, renders Tom's canonical share
  template —
  Overview block (Company / Founder / Stage | HQ | LI / Description / Round Details verbatim /
  Investor(s) as Name (Round) — funds only, stated rounds only / Materials), attached diligence
  materials
  when the Opp has them, and the founder's Original Email blockquoted with Tom-identifying
  strings stripped — founder-authored content ONLY, a source investor's note never appears; N/A
  for anything missing, no closing or signature. Named recipients resolve through the skill's
  registry and land in Bcc (To stays empty, so multiple firms can stack on one send) — "Primary"
  (any of "refer / kick out / send / share to Primary") = deal-agent@primary-os.com. Trigger on "kick [company] out to
  [firm]", "refer [company] to [firm]", "send/share [company] to/with [firm]", "deal share
  [company] out", "pass [company] along to [firm]". Distinct from log-deal-share (logging a
  dealflow share Tom RECEIVED), outreach-decliner / deal-decline (replying "sit this one out" to a
  received share), and the intro flows (people intros, not deal payloads). Creates a Gmail draft
  only — no Notion writes, no status changes, never sends.
---

# Deal Share Out

Tom kicks a deal from his pipeline over to another firm. The output is a single Gmail **draft**
sitting in his drafts folder for review — this skill never sends, and never writes to Notion.

## Modes

- **Mode B1 — Webhook (👣 kick-out reaction).** Tom reacts 👣 `:foot:` to a card in
  `#decision-retros`; the `slack-retro-webhook` Worker enqueues
  `{skill: "deal-share-out", args: {mode: "webhook", channel_id, thread_ts, ...}}` via
  claude-job-queue. See the Mode B section at the bottom.
- **Mode B2 — Webhook (auto on pass).** The `notion-webhook` Worker fires on any Opportunities
  Status flip to `Pass (Met)` / `Pass (DNM)` / `NR / Missed` on a non-(-1), non-FO card
  (deterministic gates in `notion-webhook/src/dispatch.ts`, deployed 2026-08-20), enqueuing
  `{skill: "deal-share-out", args: {mode: "webhook-status", page_id, status, oppName}}`. The
  draft just appears in Tom's Drafts folder — no reaction needed. See Mode B section.
- **Mode C — Manual.** Tom asks in conversation ("kick [X] out to [firm]"). Steps 1–6 below are
  the canonical flow; both webhook modes reference them.

**The -1 / FO exclusion gates the B2 AUTO-trigger only** (Tom, 2026-08-20). An explicit ask —
Mode C ("kick out [the -1 opp]") or a 👣 reaction on a -1 card — is Tom's call: just run it, no
pushback, no confirmation (precedent: the Eyal Binshtock -1 share, drafted and sent same day).

**Opportunities data_source_id:** `fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`
**Agent View (for name-filter fallback):** `https://www.notion.so/5fa871c765d74251b8f96b63f248ef25?v=31400beff4aa80fdb2e0000c1b6ae673`

---

## Distribution List + Source Exclusion

Deal shares go to a standing distribution list (pilot roster, 2026-08-20 — Tom will eventually
manage this as a proper list; until then THIS TABLE is the source of truth, edit it when he
adds/removes members):

**UNIVERSAL FIRM-WIDE EXCLUSION — applies automatically to every member, present and future**
(Tom, 2026-08-20, final form — superseding the same day's named-identities-only experiment; he
explicitly asked for this as a cross-firm standing rule): a deal sourced by ANY individual at a
member firm never gets shared back to that firm's deal inbox. When a new deal-share agreement
adds a member to this table, the exclusion is inherited — it is NOT a per-member option and
needs no roster opt-in; the roster column just records the firm's known identifiers. Match a
`Source(s)` person to a member by ANY of: the member's pseudo-source page; the member's inbox;
the source person's Company / firm affiliation (People DB row, email signature, Tom's
description); their email domain.

| Member (aliases in Tom's prompt) | Bcc address | Entity roster (exclusion matching) |
| --- | --- | --- |
| Primary, primary-os, "the Primary deal agent" | `deal-agent@primary-os.com` | The **"Primary" People page** (`3c200bef-f4aa-8007-9fb5-d6680db3702c` — engine referrals are sourced to it per add-to-crm's aliasing rule); ANYONE at Primary Venture Partners (Jordan Fox, Emily Man, Ben Sun, …); any `@primary.vc` / `@primary-os.com` address. NB: `tseo@primary.vc` is TOM's own address — a self-forward is Direct, never a Primary source (add-to-crm rule). |
| TX, TX Zhuo, Fika | `investments@fika.vc` (TX's team inbox — was `tx@fika.vc` until 2026-08-20, per TX's text) | The **"Fika" People page** (`3c200bef-f4aa-8134-8b3b-d5ca8e46bc0c`); ANYONE at Fika Ventures (TX Zhuo's page `67ca1fdf-b255-45a9-a1f6-66061d1d574e`, …); any `@fika.vc` address |

Sourcing ATTRIBUTION is unchanged by this: an individual's personal referral still sources to
their own People page (only Jordan Fox / the deal-agent inbox alias to the Primary pseudo-page,
per add-to-crm). Firm-wide matching applies to the EXCLUSION decision only — resolve each
`Source(s)` person's firm before setting Bcc.

- **Default motion** — Mode B (👣) and a bare Mode C "kick out [X]": Bcc the FULL list (minus
  exclusions below).
- Tom naming a subset explicitly ("only to Primary") → just those members.
- Tom giving a one-off address ("also send to a@b.com") → append to that draft's Bcc.
- Tom names an unregistered firm with no address → ask; do not guess or look one up. Recurring →
  add a row (with its entity roster).

**Source exclusion — never share a deal back to its own source, matched at the ENTITY level.**
Before setting Bcc, resolve the Opp's `Source(s)` relation (and the source-email author, and
anything Tom has flagged) and drop any member whose entity roster matches a source. A deal
sourced to the Primary pseudo-page (or by Jordan Fox on legacy rows) excludes Primary's deal
agent, exactly as a deal with TX Zhuo in `Source(s)` excludes `tx@fika.vc`. E.g. NewCo (Connor
Theilmann): `Source(s) = Rachel Pavey + TX Zhuo` → `tx@fika.vc` excluded, Primary-only Bcc.

**All recipients go in Bcc — the To field stays empty** (Tom, 2026-08-20). Members shouldn't see
each other, and Tom can stack more addresses onto one draft and send a single email. When adding
to an existing attachment draft: re-create-with-full-Bcc + `deleteDraft` the stale one, since
`update_draft` drops attachments.

---

## Step 1: Resolve the Opportunity

`notion-search` for the company name scoped to the Opportunities DB. **Empty search ≠ absent** —
before concluding the Opp doesn't exist, pull the Agent View via `notion-query-database-view` and
filter locally on `Name` (semantic search misses exact names; Clara incident 2026-07-24).

If the company has multiple round cards, share from the card Tom means — default to the **newest**
(that's where current activity lives). `notion-fetch` the page; collect:

- `Name`, `Stage`, `HQ`, `Website`, `Description`, `Contact`, `Round Details` (used VERBATIM —
  see stylebook), `Diligence Materials`, `Created` (feeds the "Originally Logged" header)
- Founder full name: the Original Email's sign-off / body; else the deck's team or contact
  slide; else `🏁 Founder(s)` relation → People DB (last — DNM founders almost never have a
  People row, Tom 2026-08-20). First name only resolvable → use it; never guess a surname (an
  ambiguous LI slug like `danieliu3120` does NOT resolve one).
- Founder LinkedIn URL, in this order (deck first — same reason):
  1. **The Diligence Materials deck** — the contact/team slide usually carries the profile URL,
     and the deck is in-scope founder material, not outside research (Solderable 2026-08-20;
     DocSend captures have no text layer — Read the PDF pages visually).
  2. `contactout_email_to_linkedin` on the Contact email.
  3. The People DB row's LI field (rarely exists for shared deals).
  Still nothing → `N/A`. Never guess a slug from a name — a wrong profile is worse than N/A.
- `🕰️ Funding History` relation (list of sibling Opp cards, one per round)
- Page body → the **Original Email** section (bold label or heading; written by `add-to-crm`)
- `Source Thread ID` (fallback for Step 4 if the body has no Original Email section)

---

## Step 2: Resolve recipients

Distribution List minus source exclusions (see that section). This is the only step that can
block (unregistered firm, no address) — everything else proceeds without asking.

---

## Step 3: Resolve Investor(s)

ONE consolidated field, every name tagged `<Name> (<Round>)` — the parenthetical lets the
recipient infer current vs prior (stylebook rule "Investor(s)"). Gather from:

1. Each `🕰️ Funding History` card's `Coinvestors` (Companies DB names), round = that card's
   Stage — e.g. `Virtue (Pre-Seed)`.
2. Investors / accelerators / live-round commitments disclosed anywhere in the Opp record,
   round = STATED only (a disclosed commitment to the round being raised gets that round's
   stage; "we led their pre-seed" → `(Pre-Seed)`). Round not specified → bare fund name —
   never infer from timing (stylebook rule).

**Fact sourcing = the WHOLE Opp record** — founder materials, the source's email, AND Tom's
Gmail correspondence with the founder (`Source Thread ID` / threads with the Contact email; a
founder telling Tom he just got into YC counts — Ample 2026-08-20). The Step 4 authorship ban
gates *quoting* (identity reveal), not *facts*: a source investor writing "we led their
pre-seed" belongs here as `Focal (Pre-Seed)` (missed by the first 👣 run on Solderable
2026-08-20 when this list said "founder's own materials" only). **Funds / institutions only** —
individual angels and operator-angels (a GP's personal check, "former VPs at X") are excluded;
accelerators count. Underivable round → bare name; no "led"/role annotations; nothing from
OUTSIDE research (web, ContactOut, memory); nothing qualifying → `N/A`.

---

## Step 4: Sanitize the Original Email

1. Take the Original Email section verbatim from the Opp body. If the body has no such section,
   fall back to `Source Thread ID` → Gmail `get_thread` and use the founder's inbound message.
2. **Authorship gate — founder/company content ONLY.** Check WHO wrote it. Written by the deal's
   source (another investor's share note, a forwarder's cover) → do NOT include it, even
   sanitized (reveals the source; may pitch other deals — Ample/Justin Moore incident,
   2026-08-20). If the source email embeds a forwarded founder note, quote only the
   founder-authored part: strip the source's cover lines and the `---------- Forwarded
   message ----------` header block (From/Date/Subject/To reveal the source), keep the founder's
   own salutation-stripped body and sign-off (No Logo/Lev pattern).
3. **No founder-authored content at all** (grapevine-sourced Opp, or source-only email) → normal
   case, not an error: drop the entire *Original Email* block per the stylebook and note
   "no founder email — <grapevine deal | source email only>" in the Step 6 confirmation.
4. Apply the anonymity rules in `writing-style/deal-share-out/STYLE.md` exactly: drop the
   salutation line, redact remaining Tom-identifying strings, un-escape Notion artifacts, change
   nothing else.

---

## Step 4b: Pass Note (Pass (Met) opps only)

If the Opp's Status is `Pass (Met)`, Tom almost certainly sent a pass note — include it as its
own section under Original Email (stylebook "Pass Note rules"):

1. **Find it**: Notes DB entry titled `[Company] - Inverted follow up` with this Opp in its
   `Opportunity` relation (the pass-note-sent webhook archives every sent note there). Fallback:
   Gmail sent search `subject:"[Company] - Inverted follow up" in:sent -subject:"Re:"`.
2. **Prepare it**: verbatim minus the stylebook's listed strips — `📧 View sent email` line,
   the `Best, Tom` close onward, line-wrap artifacts (reflow), and source-identifying references
   (redact with `[…]`, same as the Original Email treatment). Header carries the sent date:
   `Pass Note (<Month DD, YYYY>)`, from the Gmail sent message / Notes archive date.
3. **Not found** (note never sent, or pre-dates the archive): omit the section and flag
   "Pass (Met) but no pass note found" in the Step 6 confirmation.

---

## Step 5: Compose and create the draft

Read `writing-style/deal-share-out/STYLE.md` and follow its subject line, scaffold, and rules
exactly.

**Dedup first** — `create_draft` is not idempotent and deleting drafts is unreliable
([[feedback_founder_outreach_draft_dedup]]): check BOTH `list_drafts` with
`query: subject:"Deal Share: <Company>"` AND sent mail
(`search_threads: in:sent subject:"Deal Share: <Company>"`). Draft exists → don't create
another (surface it in Mode C; exit silently in webhook modes). Already SENT → the share
happened; exit silently in webhook modes (re-fires after a status correction land here),
surface in Mode C so Tom can decide whether a re-share is really intended.

**Tom deleting a deal-share draft = he changed his mind — NEVER re-draft it** (Tom, 2026-08-20;
same semantics as the -1 pipeline's deleted-draft-is-a-pass rule). Concretely:
- Rebuild/reformat batches operate ONLY on drafts that currently exist (replace-in-place).
  A share that's missing from Drafts and not in Sent was deleted on purpose — do not resurrect
  it. (This exact mistake happened once same day in reverse: a SENT share got recreated by a
  reformat batch that skipped the dedup. Rebuilds run the full dedup per company, no exceptions.)
- Webhook re-fires for the same decision are absorbed by the idempotency key; a genuinely NEW
  terminal status on the same Opp is a new decision and may draft fresh.
- A later 👣 reaction or explicit Mode C ask IS fresh authorization — Tom asking again overrides
  his earlier deletion.

**Two creation paths — attachments decide which:**

**(a) No materials → MCP `create_draft`:**
- **Bcc:** the resolved registry address(es); **To: empty** (see Recipient Registry)
- **Subject:** `Deal Share: <Company>` (no stage parens — stylebook)
- **Body:** `htmlBody` per the stylebook's canonical template (direct-href anchors, blockquoted
  founder note), plus the plaintext `body` alternative. **No closing, no signature** — documented
  exception; the body ends at the founder's sign-off (or the Overview block for grapevine deals).

**(b) Materials present → the gmail-webhook draft endpoint** (extended with Drive-sourced
attachments 2026-08-20, deployed v203; end-to-end verified same day). Attachments load
server-side in Apps Script, so the ceiling is Gmail's real 25MB combined — NEVER pass file
base64 through the MCP `attachments` param (bytes transit the model's token stream; ceiling
~500KB and it burns context).

1. Ensure every material is a Drive file. Drive chips → use the file ID directly. Notion-hosted
   files → re-upload to Drive first via the drive-upload endpoint
   (`shared-references/drive-upload.md`); Notion's signed S3 URLs expire in 1 hour and must
   never appear in an email.
1b. **No captured materials but the Opp record contains a live DocSend/Papermark deck link**
   (common on source-shared deals — Solderable, Unobio): rip it via the `docsend-to-pdf` skill's
   Python recipe, upload to `Diligence/<Company>/` on Drive, chip it onto the Opp's Diligence
   Materials (`notion_files_property.py --prop "Diligence Materials"`), then attach — the
   Unobio pattern (2026-08-20, 19-page DocSend → 2.9MB PDF, full pipeline). Live DocSend URLs
   never go in the email. If the rip fails (hard email gate, data room), fall back to
   `Materials: N/A` — never block the share on it.
2. Files > 25MB combined (common for DocSend-captured decks — the Clara pair measured
   67MB / 78MB): attach what fits, render the rest as direct Drive-link anchors in the
   Materials line (confirm link-sharing first).
3. POST to the `/exec` URL in `shared-references/gmail-label.md`, secret from
   `~/.claude/secrets/gmail-label-webhook.txt` (Python `requests`, `allow_redirects=True` —
   never `curl -L`):

   ```json
   {
     "action": "createDraft", "secret": "<secret>",
     "bcc": "<registry address(es), comma-separated>",
     "subject": "Deal Share: <Company>",
     "bodyHtml": "<stylebook HTML>", "bodyText": "<plaintext alternative>",
     "attachments": [{ "driveFileId": "<id>", "filename": "<Company> Deck.pdf" }]
   }
   ```

   Response `{ok, messageId, threadId}` — the messageId is the persistent hex id.

⚠️ If iterating on a draft that has attachments, MCP `update_draft` does NOT merge them — any
body tweak must re-create via the endpoint, then delete the stale draft with the endpoint's
`deleteDraft` action (`{"action": "deleteDraft", "secret": "<same>", "messageIds": ["<hex>"]}`,
added 2026-08-20 v204 — reliable, unlike the Chrome automation script). Delete only drafts THIS
flow created; Tom's own drafts are his.

Do NOT send. Do NOT call `send-alert` — the AI-draft PostToolUse hook on `create_draft` pings
#claude-alerts on its own.

---

## Step 6: Confirm to Tom

One line:

> ✓ Deal share drafted: **<Company> (<Stage>)** → <Firm> (`<address>`) — in your
> [drafts](https://mail.google.com/mail/u/0/#drafts). <Any caveat: missing Original Email,
> fields rendered N/A, redactions made beyond the greeting.>

---

## What this skill does NOT do

- **Never sends.** Draft only; Tom reviews and sends.
- **No Notion writes.** No status change, no note logged, no relation touched. If Tom wants the
  share recorded somewhere, that's a separate ask.
- **No internal read shared.** Status/disposition, pass reasons, diligence analyses, call notes,
  and the deal's source are never included — see the stylebook's facts-only rule.
- **No materials beyond the Opp's own.** Attach what `Diligence Materials` holds (Step 5) —
  never hunt down a deck elsewhere or share internal artifacts (first-pass PDFs, memos Tom wrote).
- **No outside research.** Every fact in the email comes from the CRM or the founder's own note.

---

## Mode B — Webhook (B1: 👣 reaction · B2: auto on pass)

Invoked headlessly by the claude-job-queue processor. NEVER ask questions.

**B2 (auto on pass)** — `args: {mode: "webhook-status", page_id, status, oppName}`. Deltas:
skip the fingerprint resolution (delta 1 below) and go straight to Step 1's `notion-fetch` with
`page_id`; recipients per delta 2; always create via the endpoint (delta 2b); **no close-loop
post** — the draft appearing in Drafts IS the signal, and the Step 5 dedup exits silently on
re-fires. Guard: re-check the Opp's current Status on fetch — no longer a pass/NR status (Tom
reversed within the debounce window) → exit without drafting. On failure exit non-zero (lands in
the queue's failed state).

**B1 (👣 reaction)** — `args: {mode: "webhook", channel_id, thread_ts, reply_ts, user, text:
"👣"}` when Tom reacts 👣 `:foot:` to a card in `#decision-retros`. Still useful for re-runs and
for cards that pre-date B2. On failure post the ⚠️ close-loop reply and exit non-zero.

B1 deltas from the canonical steps (all other logic identical):

1. **Resolve the Opp from the card, not from prompt text.** Read the reacted-to message
   (`slack_read_thread` on `channel_id` + `thread_ts`; the parent is the card). Extract the
   `[opp:<short_id>]` fingerprint, then map `short_id → opp_id` via
   `/Users/tomseo/.claude/skills/decision-retro/queue.json` (match on the `short_id` field).
   No fingerprint or a `[neg1:...]` tag with no queue match → post
   `⚠️ couldn't kick out — no [opp:] fingerprint on this card` as a thread reply and exit 0.
   Then proceed from Step 1's `notion-fetch` with that page id.
2. **Recipients = the full Distribution List minus source exclusions** (see that section). Step
   2 never blocks in Mode B.
2b. **Always create via the gmail-webhook draft endpoint** (Step 5 path (b)), even with no
   attachments — the endpoint needs only Python `requests` + the local secret file, so the flow
   has zero dependency on Gmail MCP attachment in the headless `claude --print` runtime.
3. **Dedup behaves as a silent success:** if Step 5's dedup finds an existing
   `Deal Share: <Company>` draft, post the close-loop reply pointing at it instead of drafting
   (re-added reactions and job re-deliveries land here; the D1 idempotency key absorbs most).
4. **Close the loop in-thread** (replaces Step 6's chat confirmation):

   ```bash
   ~/.claude/skills/claude-alerts-listener/post_close_loop.sh \
     "<channel_id>" "<thread_ts>" \
     "👣 Deal share drafted: <Company> (<Stage>) → deal-agent@primary-os.com (bcc) — in Drafts. <caveats>"
   ```

Worker-side contract (context, don't re-derive): reaction string `foot`, channel-scoped to
`#decision-retros` (`C0B0ETQFAUT`), reactor ≠ card author, idempotency key
`deal-share-out-react-<ts>-foot`. Lives in
`~/.claude/cloudflare-workers/slack-retro-webhook/src/index.ts` (deployed 2026-08-20).
