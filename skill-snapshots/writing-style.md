---
name: writing-style
description: >-
  Central router for Tom's writing-style corpus — the canonical voice + structural rules live in
  writing-style/<type>/STYLE.md; this entry point picks the right sub-stylebook. Two families.
  SHORT-FORM EMAIL (one stylebook per use-case, each read by its drafter): neg1-cold-outreach
  (cold founder -1 outreach; founder-outreach); newco-cold-outreach (cold outreach to a founder who
  ALREADY has a company; ad-hoc); intro-outreach (first-touch "open to connecting?" note;
  intro-outreach-drafter); intro-offer (casual "would love to intro" from-a-call note;
  intro-note-processor); intro-connect (double-opt-in connect email; intro-draft-agent);
  feedback-outreach (diligence backchannel request; feedback-outreach-drafter); talent-outreach
  (candidate/hire outreach; talent-scan); pass-note (founder pass notes; pass-note-drafter);
  deal-decline ("sit this one out" deal-share declines; ad-hoc); lp-raise-outreach (prospective-LP
  raise notes + forwardable note; ad-hoc); reference-request (cold founder reference-check asks;
  ad-hoc); portco-ask-forward (Fwd: of a portco's ask + casual cover note; ad-hoc); deal-share-out
  (outbound deal share to another firm's deal inbox; deal-share-out). LONG-FORM: letters-and-memos (LP letters, investment memos, pre-mortems, first-pass-diligence prose,
  investor-update prose). Trigger whenever Tom asks to "draft", "write", "clean up", "edit", "polish",
  or "refine" any prose or email — infer the type from context and route to the matching stylebook.
  Also on "log to writing style" / "log this letter/memo/checkpoint" (log a draft into the right
  VOICE_EXAMPLES.md), and on "log a new email form" / "add this to the email corpus" / "capture this
  as a new email type" (scaffold a brand-new email stylebook from a sample and register it here).
---

# Writing Style — Central Router

## What this skill is

The single front door to Tom's writing-style corpus. Every request to draft or edit prose or an email
routes THROUGH here to the right sub-stylebook. The real voice rules live in the sub-stylebooks; this
skill is a thin router plus two maintenance triggers ("log to writing style", "log a new email form").

```
writing-style/
  # ── SHORT-FORM EMAIL (one stylebook per use-case) ──
  neg1-cold-outreach/ STYLE + EDIT_PATTERNS + VOICE_EXAMPLES ← cold founder (-1) outreach
  newco-cold-outreach/ STYLE + EDIT_PATTERNS + VOICE_EXAMPLES ← cold founder WITH a company
  intro-outreach/    STYLE (+ EDIT_PATTERNS/VOICE as they accrue) ← first-touch "open to connecting?"
  intro-offer/       STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← casual "would love to intro" (from a call)
  intro-connect/     STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← double-opt-in connect email
  feedback-outreach/ STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← diligence backchannel request
  talent-outreach/   STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← candidate / hire outreach
  pass-note/         STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← founder pass notes
  deal-decline/      STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← "sit this one out" deal-share declines
  lp-raise-outreach/ STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← prospective-LP raise notes + forwardable note
  reference-request/ STYLE + EDIT_PATTERNS + VOICE_EXAMPLES   ← cold founder reference-check asks
  portco-ask-forward/ STYLE + EDIT_PATTERNS + VOICE_EXAMPLES  ← Fwd: of a portco's ask + casual cover note
  deal-share-out/    STYLE                                    ← outbound deal share to another firm's inbox
  # ── LONG-FORM ──
  letters-and-memos/ STYLE + VOICE_EXAMPLES                   ← LP letters, memos, pre-mortems, etc.
```

Each stylebook is a folder of:
- **`STYLE.md`** — canonical voice + scaffold + formatting rules + worked examples + anti-patterns.
- **`EDIT_PATTERNS.md`** — corrective signal: what Tom cuts/reorders/rewords, auto-fed by the
  `draft-feedback` pipeline. (Not present for letters-and-memos — that workflow is iterative
  co-writing, not single-shot drafting.)
- **`VOICE_EXAMPLES.md`** — positive ground truth: finished sends + voice analyses.

Drafters read directly from their corresponding sub-stylebook (see the routing table). This umbrella
skill exists for ad-hoc writing requests and the two maintenance triggers below.

## Email routing table — the core dispatch

**When Tom (or a skill) is drafting an email, match the use-case to its stylebook.** This IS the
"route every email to the right writing style" mechanism. The drafter named in the last column owns
the end-to-end flow; the stylebook is its voice source.

| Email use-case | When it applies | Stylebook | Drafter |
|---|---|---|---|
| **Cold founder (-1) outreach** | Introducing Inverted Capital to a pre-founder, anchored on a spike signal. **Trigger: "draft -1 note [for X]"** | `neg1-cold-outreach/` | `founder-outreach` |
| **Cold founder (NewCo) outreach** | Cold note to a founder who ALREADY has a company and is on other people's radar (investors included). Same subject as the `-1` template; hook is "Have heard great things…" or "Came across {Company} via…". Never mentions the round; ships with no personalization slot. **Trigger: "draft newco note [for X]"** | `newco-cold-outreach/` | — (ad-hoc) |
| **First-touch intro-interest** | "Would you be open to connecting with [X]?" — formal note with an About-blurb appendix, gauging interest before a formal intro (customer / investor / advisor / hire variants) | `intro-outreach/` | `intro-outreach-drafter` |
| **Warm "would love to intro" offer** | Casual, from-a-call note offering to intro someone to a company's founders — canonical subject `[Company] – would love to intro` | `intro-offer/` | `intro-note-processor` |
| **Double-opt-in connect** | Both sides said yes — the actual intro that wires them together | `intro-connect/` | `intro-draft-agent` |
| **Backchannel / feedback request** | Asking an expert in Tom's network for a diligence gut-take on a company | `feedback-outreach/` | `feedback-outreach-drafter` |
| **Candidate / hire outreach** | Reaching a potential hire for a portfolio company ("interest in connecting with [Founder]?") | `talent-outreach/` | `talent-scan` |
| **Founder pass note** | Declining a founder warmly | `pass-note/` | `pass-note-drafter` |
| **Deal-share decline** | "Sit this one out" reply to an investor who forwarded a deal/co-invest | `deal-decline/` | — (ad-hoc) |
| **LP raise outreach** | Prospective-LP note — final close, remaining allocation, soft ask; incl. the forwardable-note sub-form | `lp-raise-outreach/` | — (ad-hoc) |
| **Reference request** | Cold ask to someone in a founder's orbit for a diligence reference call. Trigger phrases: "draft reference [request/outreach] notes for [Founder]", "reach out to folks for [Founder]'s references", "founder reference [check/call] emails", "doing references on [Founder]" | `reference-request/` | — (ad-hoc) |
| **Portco ask forward** | `Fwd:` of a portco's request (vendor search, customer lead, partnership) to a contact who might be the fit or can route it onward — short casual cover note, forward carries the substance | `portco-ask-forward/` | — (ad-hoc) |
| **Outbound deal share** | Kicking a deal from Tom's own pipeline to another firm's deal inbox (e.g. Primary's deal-agent) — structured payload + sanitized founder email, draft only | `deal-share-out/` | `deal-share-out` |

If a request is a NEW email shape not in this table, see **"Log a new email form"** below — don't
force-fit it into the nearest stylebook.

## Signature — global rule for EVERY short-form email

**Always append Tom's signature** to any Gmail draft created via the API (`create_draft` /
`update_draft`), for ALL stylebooks above, using the exact canonical block at
`shared-references/gmail-signature.md` (plaintext + HTML). Gmail's native signature only
auto-appends inside the client — API-created drafts land signature-less otherwise. Confirmed
2026-08-03 on the reference-request batch and now the default everywhere. This overrides any
"no signature — Gmail auto-appends" language still lingering in an individual stylebook.

## Links — global rule for EVERY email

**Any link baked into any email draft must point DIRECTLY at its destination** (Tom, 2026-08-20,
caught on the first deal-share-out draft). A bare URL in a plaintext-only body gets auto-linkified
by Gmail through its `google.com/url?q=…&ust=…` redirect wrapper — the recipient sees an ugly
tracking URL instead of the site. The fix, for every stylebook and every ad-hoc email:

- Author an `htmlBody` with explicit anchors — display text is the bare domain or a natural label,
  href is the direct URL: `<a href="https://sagecare.ai">sagecare.ai</a>`. Contact emails get
  `mailto:` anchors.
- Keep the HTML minimal (no font styling) so Gmail applies its defaults; pass the plaintext `body`
  as the alternative with bare-domain link text.
- Links inside quoted/forwarded founder content follow the same rule — re-anchor them, don't leave
  bare URLs.

**Three documented exceptions — do not add a signature here:**
- `intro-connect` — the bare hand-off shape (`You both have context so [X] – will let you take it
  from here!`) has no `Best, Tom` closer at all; a signature would look bolted on.
- `deal-share-out` — Tom's hand-built template (2026-08-20, Sage Care) ends at the quoted
  founder's sign-off with no closing or signature; the recipient is a machine-read deal inbox and
  the From line carries his identity.
- `pass-note` — explicitly and repeatedly rules out the signature block (STYLE.md fixture #14),
  which reads as a deliberate calibrated voice choice (warm personal decline), not the auto-append
  misconception the other stylebooks were written under. Flagged to Tom 2026-08-03, not yet
  overridden — ask before changing.

## Routing logic (all writing, not just email)

When invoked for an ad-hoc draft/edit, infer the destination:

| If Tom is writing… | Read |
|---|---|
| Any email in the table above | that row's stylebook (`STYLE` + `EDIT_PATTERNS` + `VOICE_EXAMPLES`) |
| LP letter, investment memo, pre-mortem, first-pass diligence, investor update, or any long-form analytical prose | `letters-and-memos/{STYLE,VOICE_EXAMPLES}.md` |

If genuinely ambiguous, ask Tom which type. **Never guess between long-form and short-form** — they
have very different conventions. **The two cold-founder stylebooks disambiguate on one fact: does the
recipient already have a company?** No → `neg1-cold-outreach` (the `-1` template: engine-surfaced,
empty personalization slot). Yes → `newco-cold-outreach` (human/public sourcing, no mention of the
round, ships finished with no slot). They share the subject `Introducing Inverted Capital`, the
opener sentence, and the close verbatim, so **the hook sentence is the only reliable discriminator** —
"You were surfaced by an admittedly janky, homegrown tool" vs. "Have heard great things about
{Company}…" / "Came across {Company} via {source}" (all three newco radar variants count). Never classify these two on subject, or on the language their MO paragraphs still share
("pre-idea, pre-team", "non-obvious").

Within short-form email, if two other use-cases are close (e.g. intro-interest
vs candidate outreach), disambiguate on *who the recipient is and what's being offered*: a connection
to a founder as a customer/investor → `intro-outreach`; as a hire for a portco → `talent-outreach`.
`intro-outreach` and `portco-ask-forward` share the same spirit — connecting someone in Tom's
network to a portfolio company — and differ only in vehicle: a portco-originated ask email Tom can
forward (the forward carries the substance) → `portco-ask-forward`; Tom initiating with no
forwardable artifact (formal note + About-blurb) → `intro-outreach`.

## "Log to writing style" trigger

When Tom says "log to writing style" or any variant during or after a co-writing session:

1. Identify the writing type from the current session context (typically `letters-and-memos`, since
   that's the iterative-workflow stylebook; the email stylebooks are usually auto-fed by
   `draft-feedback` instead).
2. Run the voice-analysis prompt from `~/.claude/skills/draft-feedback/processor.py`
   (`analyze_voice_via_claude`, lines 274-313) on the current draft state.
3. Append a dated entry to the corresponding `VOICE_EXAMPLES.md`, using the same format as
   `append_to_voice_examples` (processor.py:251-271).
4. Confirm to Tom: which file, what got appended.

Mid-session checkpoints are supported — multiple log calls = multiple timestamped entries.

## "Log a new email form" trigger

When Tom says "log a new email form", "add this to the email corpus", "capture this as a new email
type", or hands over a sample email that doesn't fit any row in the routing table:

1. **Confirm it's genuinely new.** Compare against the routing table. If it's a variant of an existing
   use-case, log it into that stylebook's `VOICE_EXAMPLES.md` instead (don't spawn a near-duplicate
   stylebook). Only create a new stylebook when the register/scaffold is meaningfully distinct.
2. **Name the use-case.** Pick a short kebab-case slug (e.g. `co-invest-pitch`, `lp-update-nudge`) —
   confirm the name with Tom.
3. **Gather examples.** Use the sample Tom provided; optionally pull 2–4 more real sends of the same
   shape from Gmail (`search_threads` by subject pattern or a signature phrase) to triangulate the
   voice. Read the actual sent bodies, not just snippets.
4. **Scaffold the stylebook** at `writing-style/<slug>/`:
   - `STYLE.md` — follow the house shape of the existing email stylebooks: Purpose · Fixed elements ·
     Template/Scaffold · Subject line · any Variants · Formatting Rules · Voice · Worked example(s) ·
     Anti-patterns. Derive the rules from the real sends, not from generic email advice.
   - `EDIT_PATTERNS.md` — the standard header + empty Style Canon / Recent Edits sections (copy the
     shape from an existing one, e.g. `intro-connect/EDIT_PATTERNS.md`).
   - `VOICE_EXAMPLES.md` — seed with the real sends gathered in step 3.
5. **Register it** by adding a row to the Email routing table above (and the file tree), naming the
   drafter that will use it (or `— (ad-hoc)` if none yet).
6. **Optionally wire the learning loop.** If a dedicated drafter exists, add the skill→type mapping to
   `draft-feedback/processor.py` `PATTERN_FILES` and a subject/heuristic rule to its classifier so
   future edits auto-feed the new stylebook. If there's no drafter, skip — the stylebook is still
   usable by this umbrella router and by ad-hoc drafts.
7. **Confirm to Tom:** the slug, the files created, the routing row, and whether the learning loop was
   wired.

This is the "seamlessly add a new email form" path — a new use-case becomes first-class in the corpus
in one pass.

## Canonical principles

All stylebooks share a few principles worth naming once:

- **Specific quoted exemplars beat generic descriptors.** "Warm" / "thoughtful" / "candid" are useless
  without a quoted phrase that demonstrates them.
- **Patterns are observations, not commands.** Drafters apply them as priors, not rigid rules. Tom's
  voice is contextual — what's right for an LP letter isn't right for a pass note.
- **`EDIT_PATTERNS` is corrective signal; `VOICE_EXAMPLES` is positive ground truth.** Different
  cognitive cues — don't merge them.
- **Sibling stylebooks share tone, not format.** (Tom, 2026-08-03, on intro-outreach vs
  portco-ask-forward.) When two forms serve the same spirit through different vehicles, the voice
  hallmarks (hedged fit, candid mechanism-naming, apologetic openers, easy outs) are valid priors
  across both — but the scaffold/format rules never transfer. A thin corpus can borrow tone from
  its sibling; it can't borrow structure.
- **Em dashes (`—`) only in signatures and as rhetorical pauses in long-form prose. En dashes (`–`)
  elsewhere.** A Tom invariant across every stylebook.
- **Never send — Gmail draft only.** Every email stylebook produces a draft; sending stays with Tom.

## What NOT to do

- Don't hand-edit `EDIT_PATTERNS.md` Recent Edits sections — auto-fed by the `draft-feedback`
  processor. The Style Canon section above the divider IS hand-curated; promote patterns up from
  Recent Edits when they're durable and evidenced in ≥2 sends.
- Don't write `VOICE_EXAMPLES.md` entries by hand for a stylebook with a live drafter unless seeding a
  brand-new one — those are auto-fed. For `letters-and-memos`, manual logging is the only path.
- Don't merge stylebooks. Each writing type's voice is calibrated to its register and audience.
- Don't force a genuinely new email shape into the nearest stylebook — use "Log a new email form".
