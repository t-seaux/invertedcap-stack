---
name: deal-share-out
description: |-
  Share a deal from Tom's OWN pipeline outbound to other venture firms as a Gmail draft (never sends). Modes: (B1) 👣 reaction on a #decision-retros card; (B2) AUTO on pass — notion-webhook fires when a non-(-1), non-FO Opp flips to (or is created as) Pass (Met) / Pass (DNM) / NR / Missed / Lost; draft lands silently; (C) Manual — an explicit ask overrides the -1/FO exclusion (that gates only B2). Recipients = Distribution List in Bcc minus the sourcing firm (firm-wide exclusion); template, registry, and stripping rules live in the skill body. Trigger phrases — all Mode C; [X] = a company or "this / this one / that" in context; firm omitted → full Distribution List: "kick [X] out to [firm]", "kick this out", "send / shoot [X] over to [firm]", "share [X] with [firm]", "share the [X] deal with [firm]", "pass [X] along to [firm]", "refer [X] to [firm]", "forward [X] to [firm]", "loop [firm] in on [X]", "put [X] in front of [firm]", "flag [X] for [firm]", "deal share [X]", "run a deal share on [X]". Pre-pass heads-up phrasings (same flow; the Shared ledger records the recipient so the later auto-pass share excludes them): "give [firm] a heads up on [X]", "early heads up to [firm] on [X]", "let [firm] know about [X] early", "float [X] to [firm]". "Primary" (any of refer/kick out/send/share to Primary) = the Primary deal-agent inbox. Distinct from log-deal-share (a share Tom RECEIVED), outreach-decliner / deal-decline (declining a received share), and the intro flows (people, not deal payloads). Gmail draft only — no Notion writes, no status changes, never sends.

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
  Status flip to `Pass (Met)` / `Pass (DNM)` / `NR / Missed` / `Lost` on a non-(-1), non-FO card,
  AND on a row *created* with one of those statuses already set (`page.created` path — how
  `add-missed-to-crm` rows are born; both added 2026-09-15). Deterministic gates in
  `notion-webhook/src/dispatch.ts` (`DEAL_SHARE_STATUSES` / `dispatchDealShare`), enqueuing
  `{skill: "deal-share-out", args: {mode: "webhook-status", page_id, status, oppName}}`. The
  draft just appears in Tom's Drafts folder — no reaction needed. See Mode B section.
- **Mode B3 — Webhook (text command).** Tom texts "kick [X] out to [firm]" (or any trigger
  phrase) to his agent iMessage line; `sms-listener` recognizes it, sends an instant ack, and
  enqueues `{skill: "deal-share-out", args: {mode: "text", company, firms, from}}` via
  claude-job-queue — the heavy flow never runs inside the warm texting loop. See Mode B section.
- **Mode C — Manual.** Tom asks in conversation ("kick [X] out to [firm]"). Steps 1–6 below are
  the canonical flow; the webhook modes reference them.

**The -1 / FO exclusion gates the B2 AUTO-trigger only** (Tom, 2026-08-20). An explicit ask —
Mode C ("kick out [the -1 opp]"), a texted B3 command, or a 👣 reaction on a -1 card — is Tom's
call: just run it, no pushback, no confirmation (precedent: the Eyal Binshtock -1 share, drafted
and sent same day).

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

**Already-shared exclusion — the `Shared` relation (fka `Support`, renamed 2026-08-21).** The
Opp's `Shared` relation is the share ledger. Writes come from three paths, all meaning SENT:
(1) the gmail-webhook `deal-share-sent` handler records every recipient of a sent
`Deal Share:` email — members via its address map, anyone else via People DB email lookup — so
a Tom-directed pre-pass share ("kick this to Fika only", "send this one to David") self-records
on send; (2) Tom tells Claude he already shared something OUTSIDE the Deal Share flow (a phone
forward, an in-person mention) → Claude adds the entry to `Shared` for him on his word; (3) Tom
hand-adds in Notion himself. Before setting Bcc, read `Shared`
and drop any member already covered — resolved FIRM-WIDE like the source exclusion (an entry of
TX Zhuo's person page excludes the Fika inbox, the Fika/Primary pseudo-pages match directly; the
`N/A` placeholder page `18200bef-f4aa-80bc-8344-fc48c7b0fdb1` means nothing-shared and is
ignored). All members excluded (source + already-shared) → nothing to send; exit silently in
webhook modes, tell Tom in Mode C.

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

## Headless runtime — pinned tooling (webhook + text modes)

Headless runs have NO claude.ai MCP connectors — Gmail/Notion/Slack MCP tools are absent.
Never spend turns rediscovering this or hunting for binaries (the 2026-09-15 Ardent run burned
~11 min on tool discovery + a Mail.app dedup improvisation):

- **Notion reads**: `/usr/local/bin/ntn` (on PATH — never `find` for it). Page + properties:
  `ntn pages get <page-id>`. Raw API: `ntn api <path>`.
- **ALL Gmail ops** — dedup search, draft create/delete — go through the gmail-webhook `/exec`
  endpoint: URL in `~/.claude/skills/shared-references/gmail-label.md`, secret at
  `~/.claude/secrets/gmail-label-webhook.txt`, actions `searchMail` / `createDraft` /
  `deleteDraft`. POST via Python `requests` with `allow_redirects=True` — never `curl -L`.
- **Slack alert**: `~/.claude/skills/send-alert/send.sh`, unchanged.

## Performance — batch the independent reads

The canonical flow has exactly one hard dependency chain: Opp fetch → recipients/fields →
compose → create. Everything else is independent — run these in ONE parallel batch, not
sequentially (2026-09-10; sequential runs were ~2× slower for no correctness gain):

- The Step 5 dedup `searchMail` POST needs only the company name — fire it alongside the
  Step 1 Opp fetch, not after Steps 1–4.
- Drive metadata for materials-provenance vetting (multiple files → one batched turn).
- Source-person / Funding-History relation fetches (independent of each other).

The Rung-0 LI check (Step 1) is part of this: read the URL off the record you already fetched
before spending any enrichment round-trip.

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
  People row, Tom 2026-08-20). **Founder emails follow the same subset rule as LI (Tom,
  2026-09-18):** with multiple founders, render the email in parens only for those whose address
  resolved, and leave the rest as bare names — `Andrew Walters (andrew@…); Alex Lee`, never
  `Alex Lee (N/A)`. Set `email` only for resolved founders; `compose_body.py` treats empty/"N/A"
  as unresolved as a backstop. First name only resolvable → use it; never guess a surname (an
  ambiguous LI slug like `danieliu3120` does NOT resolve one).
- Founder LinkedIn URL — **the LI field should NOT render `N/A`** (Tom, 2026-08-28). Founders
  have LinkedIn profiles; an `N/A` here reads as "didn't look" and is treated as a defect, not an
  acceptable value.
  **Rung 0 — the Opp record itself (check FIRST, skip the ladder on a hit).** If a founder's
  LinkedIn profile URL is already sitting anywhere in the Opp — the page body (source texts and
  intro notes routinely carry the LI card, e.g. Ardent's `linkedin.com/in/mz21` in Original
  Text), a property, or the founder's People row linked on the card — use it and DO NOT run the
  ladder for that founder. The ladder is the single slowest stretch of this skill
  (deck page-reads + up to three ContactOut round-trips); climbing it when the URL is already on
  the record is pure latency (2026-09-10). Only when the record has no URL for a founder, climb
  the FULL ladder — in order (deck first — same reason):
  1. **The Diligence Materials deck** — the contact/team slide usually carries the profile URL,
     and the deck is in-scope founder material, not outside research (Solderable 2026-08-20;
     DocSend captures have no text layer — Read the PDF pages visually).
  2. `contactout_email_to_linkedin` on EVERY founder email you have (not just the Contact) —
     run it per founder address; a founder with an `@company` email almost always resolves.
  3. `contactout_search_people` / `contactout_enrich_person` by founder name + the deal's company
     — accept a hit ONLY when its current company/role matches the deal (that verification is
     what separates it from guessing; a name-only match is not enough).
  4. The People DB row's LI field (rarely exists for shared deals).
  Never paste an unverified slug guessed from a name — a wrong profile is worse than a hole.
  **Subset resolution (multi-founder):** when only some founders' LIs resolve, the LI field lists
  ONLY the resolved ones (join with `; `) and NEVER pads a positional `N/A` for the unresolved —
  `LI: andrew-walters-884994107`, not `LI: andrew-walters-884994107; N/A` (Tom, 2026-09-18).
  Build the `li` array from resolved founders only; `compose_body.py` also filters out any entry
  lacking a real url/slug as a backstop. But
  if the full ladder genuinely comes up empty for a founder, that is a **flag, not a silent
  `N/A`**: call it out in the Step 6 confirmation (Mode C) / the draft's Slack alert (webhook
  modes) as "couldn't resolve LI for <founder> — supply before sending", so Tom can fill it in
  rather than the draft shipping with a blank LI. If at least one founder resolved, the field still
  ships with the resolved slug(s) — only the missing founder is flagged, the field is not blanked.
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
4. Apply the anonymity rules in `~/.claude/skills/writing-style/deal-share-out/STYLE.md` exactly: drop the
   salutation line, redact remaining Tom-identifying strings, un-escape Notion artifacts, change
   nothing else.

---

## Step 4b: Pass Note (Pass (Met) opps only)

If the Opp's Status is `Pass (Met)`, Tom almost certainly sent a pass note — include it as its
own section under Original Email (stylebook "Pass Note rules"):

1. **Find it**: Notes DB entry titled `[Company] - Inverted follow up` with this Opp in its
   `Opportunity` relation (the pass-note-sent webhook archives every sent note there). Fallback:
   `searchMail` with query `in:sent subject:"[Company] - Inverted follow up" -subject:"Re:"`.
   The Gmail fallback is a FULL source, not a hint — if the sent message exists, quote its body.
   The Notes archive is a convenience copy; its absence is NEVER a reason to omit the section or
   flag "archive not available" (2026-09-15 Redwagon: Tom flipped Status manually so the webhook
   skipped the archive, the run found the sent note in Gmail but still omitted it, and the share
   went out without the pass note).
2. **Prepare it**: verbatim minus the stylebook's listed strips — `📧 View sent email` line,
   the `Best, Tom` close onward, line-wrap artifacts (reflow), and source-identifying references
   (redact with `[…]`, same as the Original Email treatment). Header carries the sent date:
   `Pass Note (<Month DD, YYYY>)`, from the Gmail sent message / Notes archive date.
3. **Not found in EITHER source** (note never sent, or pre-dates the archive): omit the section
   and flag "Pass (Met) but no pass note found" in the Step 6 confirmation.

---

## Step 5: Compose and create the draft

Read `~/.claude/skills/writing-style/deal-share-out/STYLE.md` and follow its subject line,
scaffold, and rules exactly.

**Body composition is scripted — NEVER hand-write the scaffold HTML** (added 2026-08-21: the
B2 headless run hand-built `<p>`-tag HTML that rendered double-spaced in Mail, dropped the
*Originally Logged* header, and merged the Stage/HQ labels). Build `bodyHtml`/`bodyText` by
piping a fields JSON into `~/.claude/skills/deal-share-out/compose_body.py` (input schema in
its docstring; stdout is `{"bodyHtml", "bodyText"}`). You supply VALUES — dates spelled out,
Round Details verbatim un-escaped, the sanitized + metric-bolded Original Email / Pass Note
inner fragments per the stylebook — the script owns labels, line breaks, separators, and
blockquote styling. Applies to BOTH creation paths and ALL modes (the webhook runtime has
Python; this is the same pattern as the endpoint POST).

**Materials line = deck + memo ONLY** (Tom, 2026-08-28). List the founder's primary artifact
(pitch deck OR written overview — see labeling rule below) and, if one exists, the memo — nothing
else. **Label the memo just `Memo`, never `Investment Memo`** (Tom, 2026-08-28). NEVER surface a
demo/product video, data-room link, loom, or any other artifact on the Materials line, even when
it's sitting in the Opp's Diligence Materials. Attached items read `(attached)` (e.g. `Deck
(attached), Memo (attached)`).

**Label materials by what the file ACTUALLY IS — never assume "Deck"** (Tom, 2026-09-10; caught
on the Root → Fika share, which attached Root's 4-page written overview but labeled it `Deck
(attached)` and named the file `Root Deck.pdf` — Root never had a deck). The Diligence Materials
slot holds whatever the founder sent, and the Drive filename is not authoritative (Root's file
was literally titled `Root - One-Pager.pdf` yet ran 4 pages). BEFORE writing the Materials label
and the attachment `filename`, OPEN the file and read it: a slide deck → `Deck`; a written prose
overview / narrative memo (regardless of the founder calling it a "1-pager", and regardless of
page count) → `Overview`; an investment memo → `Memo`. The body label and the attachment filename
MUST match the real artifact (`Materials: Overview (attached)` ↔ `Root - Overview.pdf`). When the
type is genuinely ambiguous, prefer the founder's own naming over guessing "Deck".

**Attach ONLY company-provided materials — never another firm's diligence material, even from
inside the Opp's Diligence Materials field** (Tom, 2026-09-01; caught on the MaxHeap → Primary
share, which attached `MaxHeap Fika Version.pdf` — a copy that had passed through Fika's own
diligence, not the founder's original deck). The Diligence Materials property is NOT a trusted
allowlist by itself — it can hold files sourced from another investor (forwarded by a syndicate
partner, pulled from another firm's data room, a version another firm annotated or re-saved).
Before attaching, confirm provenance traces back to the company/founder (sent by the founder,
pulled from the founder's own DocSend/Papermark link, or explicitly logged as founder-provided).
Red flags in the filename or file history — another fund's name, "shared by [other firm]",
a version note that isn't the founder's — mean skip it and fall back toward `Materials: N/A`
rather than guess. This is the same principle as the Original-Email rule two paragraphs up
(founder-authored content only, a source investor's note never appears) applied to attachments.

**Materials lists ONLY files actually ATTACHED to the email — never a third-party viewer link**
(Tom, 2026-08-28). A deck or memo that lives behind a DocSend / Papermark / data-room / any
external-viewer link does NOT go in the email as a link. If it can be ripped to a PDF and
attached (Step 1b's DocSend/Papermark rip → Drive → attach), attach it and list it `(attached)`.
If it can't be attached (hard email gate, un-rippable), OMIT it — do not paste the link. The
Materials line only ever names things the recipient can open from the email itself. If neither a
deck nor a memo can be attached → `Materials: N/A`.

**Dedup first** — `create_draft` is not idempotent and deleting drafts is unreliable
([[feedback_founder_outreach_draft_dedup]]): ONE `searchMail` POST to the gmail-webhook
`/exec` endpoint (same URL + secret as `createDraft`; works in EVERY mode including headless
— added v246 after the Ardent run spent ~8 min improvising dedup via Mail.app):

```json
{ "action": "searchMail", "secret": "<secret>",
  "query": "subject:\"Deal Share: <Company>\" (in:draft OR in:sent)" }
```

Any result with `"draft": true` → a draft exists → don't create another (surface it in
Mode C; exit silently in webhook modes). Any non-draft result → already SENT → the share
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
- **Body:** `htmlBody`/`body` from `compose_body.py` (see the scripted-composition rule above). **No closing, no signature** — the stylebook's
  declared EF5 exception (`shared-references/email-formatting.md`); the body ends at the founder's sign-off (or the Overview block for grapevine deals).

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
3. POST to the `/exec` URL in `~/.claude/skills/shared-references/gmail-label.md`, secret from
   `~/.claude/secrets/gmail-label-webhook.txt` (Python `requests`, `allow_redirects=True` —
   never `curl -L`):

   ```json
   {
     "action": "createDraft", "secret": "<secret>",
     "bcc": "<registry address(es), comma-separated>",
     "subject": "Deal Share: <Company>",
     "bodyHtml": "<compose_body.py bodyHtml>", "bodyText": "<compose_body.py bodyText>",
     "attachments": [{ "driveFileId": "<id>", "filename": "<Company> <Type>.pdf" }]
     // <Type> = the artifact's REAL type after opening it — Deck / Overview / Memo
     // (see "Label materials by what the file ACTUALLY IS"). Never default to "Deck".
   }
   ```

   Response `{ok, messageId, threadId}` — the messageId is the persistent hex id.

⚠️ If iterating on a draft that has attachments, MCP `update_draft` does NOT merge them — any
body tweak must re-create via the endpoint, then delete the stale draft with the endpoint's
`deleteDraft` action (`{"action": "deleteDraft", "secret": "<same>", "messageIds": ["<hex>"]}`,
added 2026-08-20 v204 — reliable, unlike the Chrome automation script). Delete only drafts THIS
flow created; Tom's own drafts are his.

Do NOT send — draft only.

**Fire a Slack alert on EVERY draft creation** (Tom, 2026-08-28). The two creation paths alert
differently on their own: the MCP `create_draft` PostToolUse hook pings #claude-alerts only on
path (a), while path (b) / the endpoint — which ALL webhook-mode drafts and every
materials-present draft use — bypasses that hook and would otherwise land SILENTLY. So this skill
sends ONE consistent alert itself, in all modes:

1. **Before** creating the draft, mute the generic hook so path (a) doesn't double-fire:
   `~/.claude/scripts/draft_alert_mute.sh on --label deal-share-out`
2. **After** the draft lands, pipe a summary to `send-alert` (`send-alert/send.sh`). Follow the
   house grammar (`send-alert/references/alert-convention.md`) exactly — `✍️` domain emoji, Title-Case
   `Headline: Subject` headline (Subject = the company alone, NO stage in the headline, NO date
   suffix — single event), a `**Key:** value · …` meta line (never a `→ … — in Drafts` prose
   line), a `✓`/`⚠` state line, and any action-required caveat led by `⚠` (never a prose blob):

   ```bash
   cat <<'EOF' | ~/.claude/skills/send-alert/send.sh
   ✍️ <u>**Deal Share: <Company>**</u>
   **To:** <Firm(s)> · **Stage:** <Stage> · **Materials:** <what's attached — e.g. "Memo (attached)", "Deck, Memo (attached)" | "N/A">
   ✓ Drafted<qualifier: ", on pass" | " — pre-pass share, <Company> still Active" | " — 👣 re-share">
   <⚠ ONE line per genuinely action-required caveat, ONLY when present — "⚠ Couldn't resolve LI for <founder> — supply before sending" · "⚠ Pass (Met) but no pass note found" · "⚠ Attachment omitted — another firm's material">
   EOF
   ```

   - **`To:` names the firm(s) only** — recipients are ALWAYS Bcc, so never write "(bcc)"; it's implied.
   - **`Materials:` states only what IS attached.** Never flag what's missing — "no deck on file",
     a dead Notion link, an absent memo are NOT caveats. Nothing attached → `Materials: N/A`, full stop.
   - **No footer draft link** — Tom reviews in Mail; the alert never links to Drafts.
   - Neutral context (intro-sourced, no Original Email block, source note withheld) is EXPECTED
     behavior, not a caveat — keep it off the alert. `Headline` stays `Deal Share:` (no
     claude-alerts-listener branch keys off it, so it's safe to keep terse).

This fires in Mode C, B1, and B2. Mode B1 STILL posts its `#decision-retros` close-loop reply in
addition (that answers the 👣 reaction; the #claude-alerts ping is the standard draft notice).

**Ad-hoc deal-share drafts follow the same convention.** Any draft created outside the main flow
but belonging to a deal share — a follow-up in the share thread (e.g. a pass-note addendum), a
correction, a re-send — gets the SAME treatment: mute the generic hook first
(`draft_alert_mute.sh on --label deal-share-out`), then send the ✍️ `Deal Share: <Company>` alert
above with the state line qualified (e.g. `✓ Drafted — follow-up: pass note addendum, in share
thread`). Never let such a draft fall through to the generic `Email Draft:` hook ping (2026-09-15
Redwagon: the pass-note follow-up surfaced as "Email Draft … To: (recipient unknown)" because the
recipients ride Bcc and the ad-hoc path had no convention).

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
- **No materials beyond the Opp's own, and only if company-provided.** Attach what
  `Diligence Materials` holds (Step 5) IF it traces back to the founder/company — never hunt down
  a deck elsewhere, never share internal artifacts (first-pass PDFs, memos Tom wrote), and never
  attach a file that originated from another firm's own diligence (see the provenance rule above).
- **No outside research.** Every fact in the email comes from the CRM or the founder's own note.

---

## Mode B — Webhook (B1: 👣 reaction · B2: auto on pass)

Invoked headlessly by the claude-job-queue processor. NEVER ask questions.

**B2 (auto on pass)** — `args: {mode: "webhook-status", page_id, status, oppName}`. Deltas:
skip the fingerprint resolution (delta 1 below) and go straight to Step 1's `notion-fetch` with
`page_id`; recipients per delta 2; always create via the endpoint (delta 2b); **no close-loop
post** — but it STILL fires the Step 5 #claude-alerts draft alert (that's the notice Tom gets;
"no close-loop" means no `#decision-retros` reply, since there's no reaction thread to answer),
and the Step 5 dedup exits silently on re-fires (a re-fire that finds the existing draft does NOT
re-alert). Guard: re-check the Opp's current Status on fetch — no longer a pass/NR status (Tom
reversed within the debounce window) → exit without drafting. **Read the `Shared` relation
FRESH from that same fetch and honor every entry regardless of who wrote it** — Tom hand-adds
entries in Notion (often right before flipping the status), so the already-shared exclusion must
run off the live relation, never off assumptions about which shares went through the email flow.
On failure exit non-zero (lands in the queue's failed state).

**B3 (text command)** — `args: {mode: "text", company, firms, from}`, enqueued by
`sms-listener` when Tom texts a trigger phrase (see the description's trigger list). `company`
= the name as texted; `firms` = member names as texted (may be `[]` → full Distribution List);
`from` = Tom's E.164 for the reply. Deltas from the canonical steps:

1. **Headless — NEVER ask questions.** Resolve the Opp from `company` via Step 1's search +
   Agent View name-filter fallback. Not found → text Tom (see delta 4's send recipe)
   `✗ Deal share: no Opp found for "<company>"` and exit 0.
2. **Recipients**: each `firms` entry resolves through the Distribution List registry;
   an unregistered firm → text `✗ <firm> isn't on the deal-share list` and exit 0 — never
   guess an address. Empty `firms` → full list minus exclusions (source + already-`Shared`).
3. **Always create via the gmail-webhook draft endpoint** (Step 5 path (b)), even with no
   attachments — zero Gmail-MCP dependency in the headless runtime. The endpoint path never
   trips the generic draft hook, so SKIP the Step 5 mute/unmute steps entirely.
4. **Alerts follow the surface (Tom's standing rule): a text-triggered share confirms by TEXT,
   not Slack.** Skip the Step 5 #claude-alerts alert. When the draft lands, text Tom the
   master-grammar text rendering (`send-alert/references/alert-convention.md` → "Text lane";
   `SENDBLUE_API_SECRET` is injected by the processor; body on stdin, NEVER a double-quoted
   argv — zsh eats `$<digits>`):

   ```bash
   ~/.claude/skills/sms-listener/send_imessage.sh "<args.from>" --stdin <<'MSG'
   ✍️ Deal Share: <Company>

   To: <Firm(s)> · Stage: <Stage> · Materials: <what's attached | N/A>
   ✓ Drafted<same qualifier rules as the Slack alert>
   <⚠ lines, same rules as the Slack alert: action-required only>
   MSG
   ```

5. **Dedup exits also text** (never silent — Tom asked for this share seconds ago): existing
   draft → `Deal Share: <Company> — already in Drafts`; already sent → `already sent <date>,
   Shared has <firm>` so Tom knows the dedup was the reason.
6. On any failure text `✗ Deal share <company> failed: <one-line reason>` and exit non-zero
   (lands in the queue's failed state).

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
