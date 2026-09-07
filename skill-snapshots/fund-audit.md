---
name: fund-audit
description: >-
  End-to-end playbook for annual fund audit cycles (Inverted Capital Fund I — auditor Frank,
  Rimerman + Co.; fund admin Vector AIS). A LIVING skill: it maps the full audit arc and carries a
  detailed playbook for each step we've actually executed; every newly handled audit-request type
  gets appended as a new step in the same pass. Currently captured: portfolio-company contact list
  for investment confirmations (designated contacts + work-email rule + legal↔CRM name map).
  Trigger when Tom forwards or references an email from the audit firm ("look at this email from
  our auditor"), or says "fund audit", "audit request", "audit confirmations", "portco contacts
  for the audit", "the auditors need [X]", or any variant asking to respond to or prepare
  materials for a fund audit. Always trigger inline — no confirmation needed before acting.
---

# Fund Audit — cycle playbook

One skill per audit *cycle type*, accreting steps as the arc unfolds. The 2026 Inverted Capital
Fund I audit is the seed cycle; future cycles (and eventually Dash, admin = Carta) reuse and
extend this file.

**🔁 Living-artifact rule (load-bearing):** whenever a new audit-request type is handled in a
session (rep letter, valuation support, PBC item, FS draft review, …), append a `## Step:` section
with the actual procedure used AND update the arc map below — in the same pass, before reporting
done. This skill is only useful if it captures the full arc.

---

## Cast & standing facts — Inverted Capital Fund I (2026 cycle)

| Role | Who |
|---|---|
| Audit firm | Frank, Rimerman + Co. LLP |
| — Senior Associate | Ryan Harvey — rharvey@frankrimerman.com (day-to-day contact) |
| — Manager | Alex Nguyen — anguyen@frankrimerman.com |
| — Partner | Steven Hong — shong@frankrimerman.com |
| Fund admin | Vector AIS — invertedcap@vectorais.com (Brentley Malan, bmalan@vectorais.com) |
| Open-items tracker | Monday.com board (auditor-managed; Tom + invertedcap@vectorais.com have access): frankrimerman.monday.com/boards/18427999231 — "Inverted Capital Fund I, LP Board" |
| Support folder | Shared Drive folder "Audit" (Vector/Align-shared, 2025–26 support) |

Division of labor: **Vector AIS owns** workbooks, support uploads, bank account numbers, and
scheduling availability on the admin side. **Tom owns** portco contacts, bank signers, intro/access
grants, and the audit inquiries call. Vector AIS is fund admin ONLY — never pull them into
diligence or intro flows.

---

## The audit arc (map)

Steps with a `## Step:` playbook below are marked ✅; others are known-but-uncaptured — append
their playbook when first executed.

1. **Kickoff & intros** — auditor emails to start the cycle; intro them to the fund admin
   (Vector for Inverted, Carta for Dash). Uncaptured.
2. **Access grants** — Drive support folder + Monday.com user list. Uncaptured (Vector-led).
3. **Q workbooks & support upload** — Vector-owned; no Tom-side procedure.
4. **Investment confirmations → portco contact list** — ✅ see Step below.
5. **Bank info** — ✅ account numbers (Vector, via Monday.com) + **bank account signers** (Tom
   answers directly — NEVER volunteer this from inference). Standing answer, given by Tom on
   Monday.com 2026-09-04: *"Bank account signers for all three entities (ManCo, GP, LP):
   Thomas Seo, Managing Member."* Future cycles: reuse unless the entity structure changed.
6. **Audit inquiries call** — scheduling via Blockit (improve@blockit.com cc'd on thread) +
   Vector availability windows. Uncaptured.
7. **Later-cycle steps** — rep letter, valuation support, financial statement draft review,
   confirmation follow-ups. Append as encountered.

---

## Step: Portfolio company contact list (confirmations)

Auditors send investment confirmations directly to each portfolio company and need **Name +
Email** per company. First executed 2026-09-04.

**1 — Read the auditor's list.** They reference companies by **legal entity name**, which often
differs from the CRM row name. Current map:

| Auditor / legal name | CRM Opp row | Notes |
|---|---|---|
| Factir, Inc. | Factir | |
| Oun Management Corporation | Oun Homes | |
| Project Obelisk, Inc. | Signal7 | legal name ≠ brand; label "(dba Signal7)" in replies |
| Quiet AI Corporation | Quiet Software | tryquiet.ai |
| Rengo AI, Inc. | Rengo | |
| Tuor, Inc. | Tuor | |

New portcos: resolve by founder-email domain match against the Opp Contact field; extend this
table when a new legal↔CRM pair is confirmed.

**2 — Pull the portfolio set.** Query the Opportunities DB
(`collection://fab5ada3-5ea1-44b0-8eb7-3f1120aadda6`):
`Fund = 'Inverted 1️⃣' AND Status IN ('Active Portfolio','Portfolio: Follow-On','Exited','Committed')`,
selecting Name, Status, Contact, 🏁 Founder(s). Follow-on rows (e.g. "Signal7 (Seed FO)")
duplicate the primary — use primary rows only.

**3 — Resolve each contact.**
- **Designated contacts** (Tom's standing picks for multi-founder companies, set 2026-09-04) —
  use these, not the full founder list:

  | Company | Contact | Work email |
  |---|---|---|
  | Factir | Monique Tuin | monique@factir.com |
  | Oun (Management Corp) | PJ Rodriguez | pj@oun.homes |
  | Project Obelisk / Signal7 | Armen Derkevorkian | armen@signal7.ai |
  | Quiet AI / Quiet Software | Nishant Karandikar | nishant@tryquiet.ai |
  | Rengo AI | Erik Ronning | erik@rengoai.com |
  | Tuor | Hardik Gupta | hardik@tuor.dev |

- **⚠️ Work emails ONLY.** The Opp row's `Contact` property carries work addresses; the People
  DB `Email` field is frequently a personal gmail — never use it for audit contacts. If a
  designated person has no work address in the Opp Contact field, resolve it (company-domain
  pattern-match against co-founders' addresses) and flag rather than substituting a personal one.

**4 — Cross-check the lists both ways.** Flag to Tom (don't silently resolve): companies on the
auditor's list with no CRM portfolio row, AND active portfolio companies missing from the
auditor's list whose investment closed inside the audit period (they may belong in the
confirmation set).

**5 — Deliver.** Two accepted channels; the auditor treats them as equivalent:
- **Monday.com item update** (Tom's revealed preference, 2026-09-04): post the bullet list as an
  Update on the matching open-request item (pulse) on the LP Board. Tom does this himself in the
  browser — Claude has no Monday access; prepare the formatted list for him. (Monday API/MCP
  tooling was considered and deliberately deferred, 2026-09-04 — don't re-pitch unless Tom asks;
  open question if revived: can his guest seat on frankrimerman.monday.com generate an API token.)
- **Email reply on the audit thread** — Gmail draft only, never send:
  - MCP `create_draft` with `replyToMessageId` = latest thread message so it threads. (The
    signature-fidelity rule in `shared-references/gmail-signature.md` prefers
    `gmail-create-draft.py`, but that script cannot reply to a thread; for this ops email,
    threading wins — pass BOTH `body` and `htmlBody` with the signature and accept the
    connector's HTML sanitization.)
  - To: auditor day-to-day contact + fund admin; Cc: auditor manager. Drop scheduling-only
    parties (Blockit) from non-scheduling replies.
  - Body: one bullet per company, `Legal name – Contact name – email`, `mailto:` anchors in the
    HTML, en dashes. Plain administrative register — no stylebook applies.
  - If Tom then delivers via Monday instead, the draft is superseded — flag it for trash
    (confirm with Tom first; drafts are his to discard).

**6 — Report to Tom** with the contact table, any flags from step 4, and any co-traveling asks
on the thread that remain open (e.g. bank signers — step 5 of the arc, Tom-only).

---

## Guardrails

- Everything sent to the auditor is a **fact from the CRM or a Tom-stated designation** — never
  inferred, never from memory alone. Unverifiable asks (signers, account details) go back to Tom.
- Drafts only; sending stays with Tom.
- Vector AIS handles admin-side items — don't duplicate their deliverables, do cc them on replies
  that touch shared workstreams.
