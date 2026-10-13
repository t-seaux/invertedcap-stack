---
name: talent-scan
description: Match Tom's network against open roles a portfolio (or any) company is hiring for. Takes JD links (Ashby/Greenhouse/Lever/etc.), pasted JD text, or a loose ask ("help Coverbase find AEs", "who do I know for a founding eng role at X"), builds an ideal-candidate profile per role, calibrates on any exemplars Tom names ("folks like A, B, C"), judges every person card in the cached LinkedIn network against each role (the network-scan judge – the same method and the same text as every network scan, on the portco-ask text lane and in a Claude session), and prints the shared scan layout headed "✅ <Co>: Talent Scan Completed". Runs in a session or from a portco-ask `hire` ask (👍 on the 🙋 card). Trigger phrases: "help [company] hire for [these JDs]", "find people in my network for [role]", "who do I know who'd fit [JD link]", "scan my network for [AEs/engineers/…] for [company]", "candidates for [company]'s [role]", "talent scan [company]", "candidate scan", "hiring scan", "who do I know for [role]", or any variant where Tom drops JD links/text or names a company + role and wants network matches. Distinct from network-scan (general network queries), coinvestor-recommender (investors for a deal), and intro-agent (inbound intro triage).
---

# talent-scan — JD → Network Candidate Match

> **Headless Gmail:** every Gmail read/write in this skill follows `shared-references/headless-gmail.md` — reads via `admin_run.py` when the Gmail MCP isn't attached (its absence ≠ Gmail down); a run that can't finish ends with a `JOB_FAILED:` line.

Find people in Tom's cached LinkedIn network who fit roles a company is hiring for. A talent scan is a network scan
(Tom 2026-10-11: "every kind of network scan regardless of what we're scanning for should be in same structure /
format" · "text and Claude sessions should refer to the same brain"): the SAME judge, recall backstop, verify step and
`scan.py render` as `network-scan/SKILL.md` Step 4 – read it. This skill adds only JD ingestion, exemplar calibration
and the role profile that becomes the judge's ask. `--kind talent` changes the headline label ("Talent Scan") and adds
the role-fit hint to the judge context (`portco_ask.KIND_HINTS`) – nothing else.

Entry points: a Claude session, or a portco-ask `hire` ask (👍 on the 🙋 card → portco-ask Mode execute runs these
steps through `portco_ask.py judge --kind talent` / `results`, which print the same text).

---

## Step 1 — Ingest the roles

Tom's input arrives in one of three forms. Resolve each to a set of `{title, key requirements, buyer/industry, deal size (sales) or tech stack (eng), seniority}` role specs.

**A. JD links (an applicant-tracking-system URL).**
- **Ashby** (`*.com/careers?ashby_jid=<uuid>` or `jobs.ashbyhq.com/<org>/...`): the page is JS-rendered — WebFetch on the careers URL returns nothing useful. Instead hit the public posting API, which returns every role with full plain-text descriptions and compensation in one call:
  ```
  https://api.ashbyhq.com/posting-api/job-board/<org>?includeCompensation=true
  ```
  `<org>` is the board slug (usually the company name lowercased, e.g. `coverbase`). WebFetch this URL and ask for each job's id, title, department, location, comp, and full description/requirements. One fetch covers all of a company's open roles — you don't need one call per `ashby_jid`; just filter to the ids Tom pasted.
- **Greenhouse**: `https://boards-api.greenhouse.io/v1/boards/<org>/jobs?content=true`.
- **Lever**: `https://api.lever.co/v0/postings/<org>?mode=json`.
- **Anything else / unknown ATS**: WebFetch the JD URL directly and extract the role spec. If cross-host redirect or empty, tell Tom and ask him to paste the JD text.

**B. Pasted JD text.** Parse directly into role specs.

**C. Loose ask** ("help [company] find AEs", "who do I know for a founding engineer at X"). No JD in hand:
- Infer the role archetype from the role name + seniority.
- If a company is named and its buyer/sector matters (it almost always does for sales roles), research what the company sells and to whom — a quick WebSearch, or the `company-scan` skill / Companies DB if it's a portfolio co. The buyer profile (regulated? fintech? dev-tool? SMB vs. enterprise?) is what makes the network match precise. Example: Coverbase sells third-party/vendor-risk management into banks + insurance → the ideal AE has sold compliance/GRC/security SaaS into regulated buyers.

For each role, write a one-line **ideal-candidate profile** before searching – it is the judge's `--ask` (and the recall
backstop's vsearch query).

---

## Step 2 — Calibrate on exemplars (do this whenever Tom names names)

If Tom says "folks like **X, Y, Z**" or "someone like [name]", these are gold — they define the target archetype far more precisely than the JD. For each exemplar:

```bash
python3 ~/.claude/scripts/network_cache.py query "<exemplar name>" --limit 3
```

Read their **current employer + role** and fold the pattern into the role's ideal-candidate profile (name the archetype
companies in the `--ask`: "…like the AEs at Vanta / Thoropass / Carta"). In the Coverbase run, the three named AEs resolved to Vanta / Thoropass / Carta — i.e. compliance/GRC/regulated-SaaS sellers — which reshaped every downstream query and put the strongest matches at the top. Do NOT skip this step when exemplars are given.

Exemplars already in the cache are judged like everyone else (they usually score 10); never hand-place them in the list.

---

## Step 3 — Judge every person card against the role (one scan per role)

```bash
S=~/.claude/skills/network-scan/scripts/scan.py
python3 $S judge --kind talent --ask "<ideal-candidate profile for this role, exemplar-calibrated>" \
    --context "<what the company sells, to whom, stage>" --exclude-firm "<the hiring company>" --exclude-firm-types VC
# → {"scan": "<id>", "out_file": …}   out_file.kept = every fit (score, why, section, folded …)
```

Every card is read blind (fit only; intro history is attached after scoring) – a cheap-tier recall screen, then a
precision re-score at the judge's configured tiers (`PC_SCREEN_MODEL` / `PC_PRECISION_MODEL`, unchanged; add
`--precision-model opus` only for a high-stakes search). Several roles = one judge + one render per role; roles with
the same archetype (Enterprise AE + Mid-Market AE) share one scan.

## Step 4 — Recall backstop, then verify every ≥8 keeper

1. **Recall backstop** – `vsearch` on the ideal-candidate profile (person-trait, the default for hiring) and `csearch
   --no-vc` for company-trait signals ("worked at Vanta / Drata"); `query "<name>"` for exemplars and names that surface.
   Anyone they surface that the judge did NOT keep: read their card (`person_cards.py card <url>`) and add them only if
   the card shows a real fit for the role.
2. **Verify the current role** of every keeper scored ≥8: `network_cache.py dump <url>` and read the full profile – a
   card can lag a job change. Rewrite `why` from what the profile says: what they sell / build and to whom, and tenure
   as fact ("in the seat since Feb 2026"). No speculation about availability or intent ("not poachable", "likely
   open"). By name, never he / she.
3. **Optional live LinkedIn check – only when the cache copy is older than 30 days** (`network_cache.py query
   "<name>" --limit 1` → `fetched_at`) or the profile hints at a move (an end date with no successor, "open to work").
   Read Tom's logged-in LinkedIn via Chrome (`shared-references/chrome-read-page.md`); fallbacks in freshness order:
   `network_cache.py enrich-url "<li_url>"` → `dump`, then `WebSearch "<name> LinkedIn"`. The current employer is the
   one error Tom always catches (the cache had AJ at Vanta after he'd moved to Schellman). If you refetched, the cache is
   already updated (`enrich-url` upserts); `enrich-file … --force` for a row under 30 days old you KNOW changed. No new
   employer anywhere → keep the last company and say "left <Company> Jun 2026" in the why – never guess.
4. Every judge fit ≥8 goes in `people` or `dropped: [{linkedin|name, why}]` with a concrete reason (left the field,
   at the hiring company, wrong function on a closer look). Never trim for length.

## Step 5 — Render and paste verbatim

```bash
python3 $S render --scan <id> --target "<short plural target, e.g. AEs selling GRC software to banks>" \
    --for "<hiring company>" [--founder "<founder the candidates go to>"] <<'J'
{"people": [{"name": "...", "role": "...", "company": "...", "linkedin": "https://www.linkedin.com/in/...", "why": "..."}],
 "dropped": [{"linkedin": "...", "why": "..."}]}
J
```

It prints the shared scan text – "✅ <Co>: Talent Scan Completed", numbered `N. Name (Role @ Firm). Why. link ↗`
rows, 🔁 Intro'd Before / 🆕 Never Intro'd only when someone has intro history, one row per VC firm, every ≥8 fit sent
(all of them, split across texts if long), 6–7s held, the 🔎 coverage footer and the 💬 CTA. Paste it exactly as printed – no per-role
headers, no bullets, no re-ordering; a short note AFTER it (asymmetries, e.g. "network is far denser in GTM than in
early-stage engineers", or an offer to widen with a competitor `csearch`) is fine. Exit 2 = refused (a ≥8 fit neither
sent nor dropped, a bad row) – fix the payload and rerun.

---

## Founder email (only when Tom asks)

The rendered scan is the deliverable. If Tom wants an email to the founder, he picks who goes in – the numbers off the
scan, or an `AskUserQuestion` with `multiSelect: true` (one option per candidate, ≤4 per question; "none" = no draft).
Then ONE Gmail draft with exactly the picks (collect the full selection first, never a draft per pick):
- **Subject** `COMPANY: Role(s) Candidates` (`Coverbase: AE Candidates`; several roles → `Coverbase: AE & SWE Candidates`).
- **Body** = the picked rows as the scan printed them, number dropped: `Name (Role @ Firm). Why.` with the name linked
  to LinkedIn – no re-written rationale, no tenure verbs, no speculation. Settle wording in chat FIRST (the Gmail
  connector is create-only – see [[gmail-connector-create-only-no-modify]]).
- **Never ask "want me to send it?"** Tom sends; end on the draft.

## Candidate outreach — after the founder opts in (the downstream step)

The full workflow is: talent-scan surfaces candidates → Tom sends the shortlist to the portfolio founder → the founder replies opting in on specific people ("reach out to X and Y", often passing on others) → **Tom then sends each opted-in candidate a warm outreach email.** When Tom shows/forwards a founder opt-in, draft those outreach emails.

- **Only the opted-in candidates.** Respect the founder's picks exactly — skip anyone they passed on (e.g. Coverbase's Clarence opted in on Raymond Kim + Will Damron, passed on AJ "given how recently he joined" — so only Raymond + Will get drafts). If Tom says one is "already drafted," skip it.
- **Fill the To field** with the candidate's email – lookup order `shared-references/intro-recipient-rules.md` R2 (Contacts before ContactOut; ContactOut routing: `shared-references/contactout-routing.md`). No email anywhere → the LinkedIn DM fallback (R3): draft the DM in Beeper, never send, report it.
- **One draft per candidate.** Formatting and signature per `shared-references/email-formatting.md` (the helper below appends the signature, EF5).
- **Create each draft via `~/.claude/scripts/gmail-create-draft.py --skill talent-scan`** (not the Gmail MCP) — the helper creates the draft AND writes the draft-feedback snapshot atomically, so Tom's edits feed `writing-style/talent-outreach/EDIT_PATTERNS.md` via diff mode. Write the HTML body + a plain-text snapshot body (strip tags, no signature) to scratch files, pass `--html-body-file` / `--snapshot-text-file`. Exit code 0 = success; non-zero = that candidate's draft failed (don't fall back to a snapshot-less MCP draft). If a candidate's email is unresolved, skip the draft (the helper requires `--to`) and surface the name to Tom instead.

**Canonical voice + format live in the corpus: `~/.claude/skills/writing-style/talent-outreach/STYLE.md`.**
Read `STYLE.md` + `EDIT_PATTERNS.md` + `VOICE_EXAMPLES.md` in that folder before finalizing every
candidate draft. The stylebook owns the subject-line form (`Intro to [Founder] @ [Company]?`), the
soft-ask body, the `you came up in conversation` sourcing line, role-based team phrasing (GTM vs
engineering), the JD link rule, the purple-left-border blurb block, the reconnect-preamble variant,
and the low-pressure register. Do not re-specify the template here — the stylebook is the single
source. (Still respect the founder's candidate picks and the one-draft-per-candidate / To-field rules
above.)

## Follow-ups (only if Tom asks)

- **Draft intro notes** — a short warm note to the candidate (match `writing-style/talent-outreach/STYLE.md`) or an ask-for-referral note to an exemplar (match `writing-style/neg1-cold-outreach/STYLE.md`). Never send; leave as a Gmail draft.
- **Enrich a candidate** — ContactOut via the `neg1-enricher` primitives or `contactout_enrich_linkedin_profile` for email + full tenure.
- **Log against a portfolio Opp** — if the hiring company is a portfolio Opp and Tom wants a record, note the candidates in the Opp (confirm first — writes to Notion).

Do NOT auto-run any of these; the skill's job ends at the rendered scan unless Tom picks a follow-up.
