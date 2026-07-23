---
name: talent-scan
description: Match Tom's network against open roles a portfolio (or any) company is hiring for. Takes JD links (Ashby/Greenhouse/Lever/etc.), pasted JD text, or a loose ask ("help Coverbase find AEs", "who do I know for a founding eng role at X"), builds an ideal-candidate profile per role, calibrates on any exemplars Tom names ("folks like A, B, C"), scans the cached LinkedIn network via the network-scan primitives, and returns a ranked, de-duped shortlist per role with LinkedIn URLs. Manual/conversational only. Trigger phrases: "help [company] hire for [these JDs]", "find people in my network for [role]", "who do I know who'd fit [JD link]", "scan my network for [AEs/engineers/…] for [company]", "candidates for [company]'s [role]", "talent scan [company]", "candidate scan", "hiring scan", "who do I know for [role]", or any variant where Tom drops JD links/text or names a company + role and wants network matches. Distinct from network-scan (general network queries), coinvestor-recommender (investors for a deal), and intro-agent (inbound intro triage).
---

# talent-scan — JD → Network Candidate Match

Find people in Tom's cached LinkedIn network who fit roles a company is hiring for. This is a hiring-specific wrapper over the `network_cache.py` search primitives documented in `network-scan/SKILL.md` — read that file for the full `vsearch` / `csearch` / `query` flag reference. This skill adds: JD ingestion, exemplar calibration, per-role profile construction, and by-role ranked output.

Manual/conversational only (no scheduled or webhook entry point).

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

For each role, write a one-line **ideal-candidate profile** before searching. This is what you'll vector-search on.

---

## Step 2 — Calibrate on exemplars (do this whenever Tom names names)

If Tom says "folks like **X, Y, Z**" or "someone like [name]", these are gold — they define the target archetype far more precisely than the JD. For each exemplar:

```bash
python3 ~/.claude/scripts/network_cache.py query "<exemplar name>" --limit 3
```

Read their **current employer + role** and fold the pattern into the role's ideal-candidate profile. In the Coverbase run, the three named AEs resolved to Vanta / Thoropass / Carta — i.e. compliance/GRC/regulated-SaaS sellers — which reshaped every downstream query and put the strongest matches at the top. Do NOT skip this step when exemplars are given.

Exemplars already in the cache are also themselves confirmed matches — surface them at the top of the relevant role as "already in-network" and note Tom can ping them directly or ask for referrals.

---

## Step 3 — Search the network per role

Pick the primitive by where the discriminating signal lives (same rule as `network-scan` Step 2):

- **Person-trait** (role pattern, expertise, seniority, "sells compliance SaaS", "full-stack AI product engineer") → `vsearch`. This is the default for hiring.
- **Company-trait** ("worked at Vanta/Drata/Plaid", "operators from a hypergrowth fintech") → `csearch --no-vc`.

Build the query from the ideal-candidate profile + exemplar calibration. Prefer one tight, exemplar-anchored query per role (naming archetypal companies) over a generic one — e.g. for the AE role, `"...selling compliance, GRC, security, or vendor-risk SaaS to banks/insurance/regulated enterprises, like Vanta Drata Thoropass"` beats a bare `"account executive"`.

Results are often duplicated in the cache (each profile can appear twice) and the raw JSON is large. Use this compact de-duping helper to keep output readable:

```bash
cat > /tmp/ts_vs.py << 'EOF'
import json, subprocess, sys
q = sys.argv[1]; limit = sys.argv[2] if len(sys.argv) > 2 else "40"
out = subprocess.run(["python3","/Users/tomseo/.claude/scripts/network_cache.py","vsearch",q,"--limit",limit],
                     capture_output=True, text=True)
data = json.loads(out.stdout)
seen = set()
for r in data:
    n = r.get("name","").replace("#","").strip()
    if n in seen: continue
    seen.add(n)
    parts = [p.strip() for p in r.get("context_blob","").split("\n") if p.strip()]
    hl = parts[1] if len(parts) > 1 else ""
    comps = ", ".join(c.get("name","") for c in (r.get("companies") or [])[:4])
    print(f"{r['distance']:.3f} | {n} | {hl[:95]} | {r.get('linkedin_url','')} | co:{comps}")
EOF
python3 /tmp/ts_vs.py "<ideal-candidate query for this role>" 40
```

`vsearch` distances under ~1.05 are good; under ~1.00 is strong. For `csearch`, the JSON already carries per-person tenure tails and company momentum — filter by whether the person was there during the relevant window.

Run one search per role. For roles with the same archetype (e.g. Enterprise AE + Mid-Market AE) a single search covers both — split the shortlist by seniority/deal-size afterward rather than searching twice.

---

## ALWAYS confirm each candidate on their live LinkedIn — direct Chrome read (default for this skill)

**Volume here is low (a shortlist is ≤8, flagged adds come one at a time), so visit every candidate's LinkedIn profile directly and confirm their current role + tenure before writing their bullet.** This applies to BOTH the initial-scan shortlist and any name Tom later flags to add — never write a candidate's employer/role from the cache or an aggregator. The cache is routinely 30+ days stale, and a wrong current employer is the single error Tom always catches (this run alone: the cache had AJ at Vanta when he'd moved to Schellman, and every aggregator had Will at Thoropass when he'd left in Jun 2026).

**The confirm step for each candidate:**

1. **Read Tom's logged-in LinkedIn via Chrome — this is THE source of truth, do it by default.** Open the profile in Tom's browser and read `main.innerText` (recipe: `shared-references/chrome-read-page.md`). The live Experience section carries **end dates** no aggregator has yet (it's what surfaced Will Damron's `Account Executive, Thoropass · Dec 2024 – Jun 2026` end date). Read the **headline** too — "Having fun" / "open to work" over an ended role = between roles. Tom is almost always logged in; the per-profile Chrome read is cheap at this volume.
2. **Fallbacks only if the Chrome read is unavailable** (Chrome not open, not logged in, JS-from-Apple-Events off), in freshness order: ② Exa refresh (`enrich-url "<li_url>"` → `dump`) → ③ `WebSearch "<name> LinkedIn"` → ④ `contactout_enrich_linkedin_profile` for structured *history* only (its `Current Company` is months stale — never authoritative for "current").
3. **If none show the new employer, say so and leave a placeholder** — don't guess. (AJ's Feb 2026 company wasn't indexed on ContactOut, Exa, or public search; the honest output was `@ [new role, started Feb 2026 — prev. Vanta]`. If a profile shows a role with an end date and no successor, keep `@ <Company>` and put `left Mon YYYY` in the parens — that's how Will Damron resolved: `**Will Damron @ Thoropass** (Account Executive, left Jun 2026)`.)
4. **Write the bullet** from the fresh data (format below).
5. **Update the cache — you already paid for the lookup, so persist it:**
   ```bash
   python3 ~/.claude/scripts/network_cache.py enrich-url "<li_url>"
   ```
   This refreshes the row from Exa and upserts. If the row isn't 30 days stale yet but you KNOW it changed, force a refetch:
   ```bash
   printf '%s\n' "<li_url>" > /tmp/ts_refresh.txt && python3 ~/.claude/scripts/network_cache.py enrich-file /tmp/ts_refresh.txt --force
   ```
   Then keep search state consistent: `resolve-companies --missing-only` (re-links employers), and `embed --missing-only` (embeds only profiles brand-new to the cache). Caveat: re-embedding an *already-embedded* profile is not incrementally supported — the refreshed `raw_text` lands immediately for `query`/`dump`, but its search vector only updates on the next full `embed` pass. Acceptable for a one-off add; don't trigger a full re-embed for a single person.

---

## Step 4 — Rank, dedup, attach URLs

- Keep matches whose headline/employer genuinely fits the role. Score relevance and keep the top ~5–8 per role; a match at distance 1.06 with an on-the-nose headline beats a 1.00 with a vague one.
- The compact helper already prints `linkedin_url`. If you need a URL for a name pulled from elsewhere, use `query "<name>" --limit 1` — but note fuzzy match can return the wrong person (in the Coverbase run it mis-resolved "Raymond Kim" and "Ryan Fallon"). Verify the returned name matches before featuring, and cross-check against the calibration lookups from Step 2.
- Drop pure non-fits (e.g. a Customer Success or Product leader surfacing on an AE query) unless they're a strong referral source worth flagging as such.

---

## Step 5 — Return the shortlist, grouped by role

Format each candidate as a single bulleted line — name links to LinkedIn, company links to the company website:

```
- [Name](linkedin_url) @ [Company](company_website) (Role, joined Mon YYYY); rationale — one-line take specific to this role, plus any qualifier (recent job change, tenure, softer-fit, seniority mismatch). Leave the rationale empty if you have no genuine take.
```

Rules for the line:
- **Name → LinkedIn, Company → website.** Role in parentheses. Then `;` then the take.
- **Bold from the name through the company** — i.e. `**[Name] @ [Company]**` is bold (in a Gmail `htmlBody`, wrap the whole `<a>Name</a> @ <a>Company</a>` span in `<strong>`, links preserved inside). The role parenthetical and rationale are NOT bold.
- **Always state tenure in the parens** using the fresh lookup's start date: `(Role, joined Mon YYYY)`. This surfaces "just moved" (unlikely to poach) vs. "years in" (likely open) at a glance.
- **If the person has LEFT their last company and no new role is indexed**, keep `**[Name] @ [Company]**` as normal (company still linked — it's their most recent) and just swap the tenure verb: `(Role, left Mon YYYY)` instead of `joined`. The `left` marks they're no longer there; the company stays because it's still the relevant context. Example: `**Will Damron @ Thoropass** (Account Executive, left Jun 2026)`. If a NEW employer is found, use the normal `joined Mon YYYY` form with the new company instead.
- **Rationale is ONE tight line** — a single clause, no run-ons. A real take on why they fit *this* role. If nothing substantive, leave it blank rather than pad. Keep the whole bullet skimmable.
- **NO speculation — state only what the data supports.** Fit (domain, what/who they sell), background, and tenure are facts and fair game. Never guess at intent or availability: no "not poachable," "likely open," "ready for a move," "referral/later," "unlikely to leave." Tenure is shown factually via `joined Mon YYYY` and the reader draws their own conclusion — don't editorialize on top of it.
- **Always fold in qualifiers** from the fresh lookup: "just started a new role in Feb, unlikely to move," "~2 yrs in, likely open," "role is community sales not a quota-carrying AE — softer fit." Qualifiers are often the most useful part.
- **Group by role under a short role-tag header** (`AE`, `SWE`, `Integration Eng`, etc.) ONLY when there is more than one role — the tag on its own line, then that role's bullets beneath it. **For a single role, omit the header entirely** — just the bullets:
  ```
  # single role — no header, just bullets:
  - **[Name] @ [Company]** (Role, joined Mon YYYY); rationale
  - ...

  # multiple roles — one header per role:
  AE
  - ...

  SWE
  - ...
  ```

**Email subject line:** `COMPANY: Role(s) Candidates`.
- One role → `Coverbase: AE Candidates`. Multiple → join the role tags with `&`: `Coverbase: AE & SWE Candidates`.
- Use the same short role tags as the body headers.

This same line format is what goes into any Gmail draft (Step / follow-up) — build the draft `htmlBody` with `<a>` tags for both links and keep the rationale inline. **Settle the final wording in chat FIRST, then create the draft ONCE.** The Gmail connector can't update or delete a draft (create-only, no modify scope), so a draft-per-edit loop just piles up stale copies — see [[gmail-connector-create-only-no-modify]].

Rules:
- ≤ 8 people per role, most-relevant first.
- Put Tom-named exemplars (if in-cache) at the top of their role, flagged "already in-network — ping directly or for referrals."
- Call out asymmetries honestly (in the Coverbase run, the network was far denser in GTM than in early-stage product engineers) and offer to widen: more specific target companies, `csearch` by competitor, or ContactOut enrichment.
- If a role returns < 3 real fits, say so and suggest broadening rather than padding.

End with the coverage line and follow-up offers:
> Cache covers {N} profiles. Want me to (a) draft intro notes to any of these, (b) log this as a talent-intro batch against [company]'s Opp, or (c) enrich a name with full tenure/contact detail before you reach out?

---

## Assembling the draft — use a multi-select pick list (Tom's preferred interface)

After presenting a verified shortlist (or a batch of newly-verified additions), **let Tom choose who goes in the email via an `AskUserQuestion` with `multiSelect: true`** — one option per candidate, a tight one-line description each (role @ company + the key qualifier). This is his preferred interface: fast, low-friction, no back-and-forth typing. Then build/update the single Gmail draft with exactly the picked candidates.

- Use it whenever there's a set of candidates to commit to the draft — both the initial shortlist and any later "add these" batch.
- One candidate per option; `AskUserQuestion` allows up to 4 options per question, so if the verified set is larger, either split into two questions or pre-trim to the top picks and say what you dropped.
- "None" / deselect-all is a valid answer — if Tom picks none, make no draft changes.
- Respect the one-draft rule: collect the full selection first, then create ONE draft (never a draft per pick).
- **Never ask "want me to send it?"** Tom's normal flow is to copy-paste the bullets into a fresh email himself — he handles sending. The clean, copy-pasteable bullets (in chat + the Gmail draft as a convenience) are the deliverable; end on the bullets, not a send prompt or a "ready to send?" question.

## Candidate outreach — after the founder opts in (the downstream step)

The full workflow is: talent-scan surfaces candidates → Tom sends the shortlist to the portfolio founder → the founder replies opting in on specific people ("reach out to X and Y", often passing on others) → **Tom then sends each opted-in candidate a warm outreach email.** When Tom shows/forwards a founder opt-in, draft those outreach emails.

- **Only the opted-in candidates.** Respect the founder's picks exactly — skip anyone they passed on (e.g. Coverbase's Clarence opted in on Raymond Kim + Will Damron, passed on AJ "given how recently he joined" — so only Raymond + Will get drafts). If Tom says one is "already drafted," skip it.
- **Fill the To field** with the candidate's email — fetch via `contactout_enrich_linkedin_profile` (`profile_only=false`) to the extent it resolves; leave blank only if no email is found.
- **One draft per candidate.** No personal signature in the body — Tom's Mail client appends his signature on paste/send.

**Template** (match voice/tone exactly — warm, brief, genuinely low-pressure):

- **Subject:** `Intro to [Founder] @ [Company]?`
- **Body:**
  ```
  Hey [First name],

  Hope you're well! Apologies for the out-of-the-blue note, but would you have any
  interest in connecting with [Founder](founder LinkedIn) @ [Company](company site)
  (a portfolio company).

  The company's building out their [GTM / engineering] team and you came up in
  conversation. More on the company below and JD [here](role JD link) in case of
  interest. Though zero pressure... no worries if not interesting / not the right time.

  Best,
  Tom

  --

  [company blurb block — left purple border, border-left:3px solid #8B5CF6]
  What is [Company]?
  [1-2 sentence what-it-does] :
  1. [Value prop 1 — bold lead-in]: …
  2. [Value prop 2 — bold lead-in]: …
  3. [Value prop 3 — bold lead-in]: …
  ```
- **Links:** Founder name → founder LinkedIn; Company → company website; "here" → the JD for *that candidate's* role (Enterprise AE JD for senior AEs, the eng JD for engineers, etc.).
- **Team phrasing:** "GTM team" for AE/sales roles, "engineering team" for eng roles.
- **Company blurb:** reuse the company's canonical one-pager blurb verbatim (Tom maintains it). Coverbase's, for reference: "Third-party risk management (TPRM) involves more admin than risk mitigation. Coverbase is a TPRM copilot that automates 90% of third-party risk assessments using AI:" + (1) Automate data collection (2) AI gap analysis (3) Conduct continuous diligence.

## Follow-ups (only if Tom asks)

- **Draft intro notes** — a short warm note to the candidate or an ask-for-referral note to an exemplar. Match `writing-style/outreach/STYLE.md`. Never send; leave as a Gmail draft.
- **Enrich a candidate** — ContactOut via the `neg1-enricher` primitives or `contactout_enrich_linkedin_profile` for email + full tenure.
- **Log against a portfolio Opp** — if the hiring company is a portfolio Opp and Tom wants a record, note the candidates in the Opp (confirm first — writes to Notion).

Do NOT auto-run any of these; the skill's job ends at the ranked shortlist unless Tom picks a follow-up.
