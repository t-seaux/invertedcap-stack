---
name: linkedin-dm
description: |-
  Convert one of Tom's email drafts (intro request, forward cover note, any short note) into a LinkedIn DM, and save it as a DRAFT in the person's LinkedIn chat via Beeper – never sent. Use when there's no usable email for the recipient (People DB address is stale, nothing in Apple Contacts / past mail / ContactOut), or Tom says "DM them on LinkedIn instead", "convert this to a LinkedIn DM", "draft a LinkedIn message for [X]", "ask [X] for their email on LinkedIn". A forward becomes the cover note + the founder's blurb verbatim (never the forwarded email).
---

# linkedin-dm

Tom 2026-10-06: "a converter and adapter … if we can't find the contact email, we'd want to contemplate DM."
Mechanics are code: `li_dm.py` in this folder (docstring has every rule and exit code). This file is when + how.

## When

The recipient-address order (forward / intro-request stylebooks): People DB work address if they still work there →
Apple Contacts personal address → past correspondence (both inboxes) → ContactOut → **LinkedIn DM (this skill)**.
Never guess a work address.

## Building the DM

`python3 li_dm.py convert --html-file <the email draft's HTML>` – Tom's words stay verbatim; links become
`Text (short url)`; no signature or sign-off; opener says "out-of-the-blue DM"; the close drops "No worries if
you'd rather not…" and ends "Happy to send more info – I'm at tom@invertedcap.com in case you want to move comms to
email." With the blurb in the DM: "More context below. I'm at tom@invertedcap.com in case you want to move comms to
email.", a "—" divider, then "About <Co>" (Tom 2026-10-06).
exit 4 = self-check failed – fix, never force.

- **A forward** (Tom 2026-10-06): convert the cover note AND add the founder's blurb verbatim –
  `--blurb-html-file <the founder's blurb section, as HTML> --company <Co> --founder <First>`. It lands under
  `About <Co> / <Founder>` (first person kept – Tom 2026-10-07). Never the forwarded email itself, never the founder's person-specific pitch.
  Slice: from after the founder's blurb heading through the founder's closing paragraph before the sign-off,
  VERBATIM – when the note was written for THIS person (Charlie's Upstart note → "I think Upstart might be a great fit
  here" stays, Tom 2026-10-07). Never genericize the founder's words.
- **A written intro request**: `--with-blurb` keeps its About block.
- The short "have an intro… what's your email?" DM is retired (Tom didn't like it).

## Saving the draft (Beeper)

Proactive: any drafter that ends the address lookup with no email drafts the DM itself and alerts Tom –
`shared-references/intro-recipient-rules.md` R2–R3.


1. `python3 li_dm.py find-chat --name "<Full Name>"` – the person's EXISTING LinkedIn chat (exit 4 = none or
   ambiguous). Never start a chat – opening one can send; hand Tom the text instead (R3).
2. `python3 li_dm.py beeper-draft --chat-id <id> --text-file dm.txt` – saves into the chat's message box, unsent.
   **Adapter gate (code):** only `convert` output is accepted (exit 4 otherwise) – the email line, "out-of-the-blue DM",
   no signature are guaranteed on every DM draft.
   Uses the "Claude drafts" token (`~/.claude/secrets/beeper-draft-token.txt`, write scope – drafting needs it). Off the
   Mac set `BEEPER_API_URL=https://beeper.invertedcap.com` (Access credential added automatically). Markdown in → Beeper
   rich text (blank lines = paragraphs). Verified 2026-10-07 on a LinkedIn chat: draft in the box, nothing sent.
   An old draft in that chat is replaced (Tom 2026-10-06) – its text is kept in
   `~/.claude/data/beeper-replaced-drafts.jsonl` first; mention the replacement to Tom. exit 3 = no token / API error.
3. Tell Tom: who, which chat, the DM text. Tom sends it from LinkedIn / Beeper himself.

**All drafts at once:** Beeper has no drafts folder (its "Send later" bucket is scheduled SENDS – never use it).
`python3 li_dm.py drafts` lists every Beeper chat holding an unsent draft AND every pending InMail (Gmail drafts
tagged `InMail →`). No chat → `li_dm.py inmail … --gmail-draft --name "<Name>"` (R3).

Never use Beeper's send-message tool or endpoint – drafts only (denied in settings).

## Keeping remote access alive

Beeper is reached at `https://beeper.invertedcap.com` (Cloudflare tunnel + Access; memory
`reference_beeper_remote_tunnel`). `scripts/beeper_access_refresh.py` runs weekly (launchd
`com.invertedcap.beeper-access-refresh`): it REFRESHES the Access service token within 60 days of expiry (same
credential, new expiry – never a new key) and checks the tunnel answers; any failure → one Slack alert with the fix.

## InMail sent → Outreach (automatic)

Always pass `--person-id <People page id> --opp-id <Opp id> --opp-name "<Opp title>"` with `inmail --gmail-draft` – that
is what lets the sweep move the person. `li_dm.py inmail-sweep` rides the 5-min outlook-mail-watch tick (Tom
2026-10-07: "sit on the 5 minute watcher"): an open InMail seen SENT in Beeper (primary – InMails sync into the
LinkedIn chat) (the ONLY send signal – a deleted draft alone never counts: Tom 2026-10-07 "i might just delete without
sending"; watched 72h for a late sync, then closed silently) → ☎️ Outreach via the intro-lifecycle endpoint, the Gmail draft removed when Beeper shows it
sent (Tom's standing OK), and ONE alert in the usual `🤝 Intro Pipeline: <Opp>` shape with the InMail source + a
"Gmail draft … deleted" line. Ledger: `~/.claude/data/pending-inmails.jsonl`.

**Cold -1 notes – same flow** (Tom 2026-10-07): pass `--li <their LinkedIn URL>` instead of the intro ids. Seen sent →
candidate store `reached-out` + the -1 Opp's Status 👓 Qualified → ☎️ Outreach (only from Qualified – an Opp already
further along is left alone) + ONE `🧍 -1 Pipeline: <Name>` alert in the same stage-move grammar. Deleted and never seen
sent for 72h → the -1 pass (`Draft trashed`) + the card-thread ask for a one-line why – deleting a -1 draft has always
meant pass. The -1 Opp's Contact reads `N/A (LinkedIn DM)`.

**Their real email → Contact (automatic, Tom 2026-10-07).** The same rider (`contact_watch`, every 15 min) watches each
LinkedIn-only -1 founder (store drafted / reached-out, no email, touched in the last 120 days): an email FROM them
(display name = their full name), an address they type in their LinkedIn chat, or a calendar invite with them on it (attendee display name = their full name, or their full name in the title with one outside attendee) → the Opp Contact is replaced, the store
email backfilled, and ONE `🧍 -1 Pipeline: <Name>` alert: `✓ CRM Contact Field: N/A (LinkedIn DM) → <addr>` (+ `✓ Notion People DB Email: added` when it was empty) + Source. Two different
addresses → ⚠ alert asking which, nothing written. Tom's own addresses are ignored. If the field changes from `N/A (LinkedIn DM)` to an address some other way (by hand,
another flow), the same alert still fires once with "Source: already on the Opp" (Tom 2026-10-07: alert whenever their email changes). The
invite → Contact flow can't do this for these Opps: it matches invites by the Contact email, and `N/A` has none.
