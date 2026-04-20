---
name: coinvestor-recommender
description: >
  Recommend qualified coinvestors for a specific deal Tom is investing in, drawing on a structured knowledge base of Tom's working relationships in the VC network. Trigger when Tom says "recommend coinvestors for [company]", "who should I bring into [company]", "who could co-invest in [deal]", "coinvestor recommendations for [round]", "who should come into the cap table", "build out the syndicate for [company]", "find coinvestors for [deal]", "who could fill out the round", "suggest investors to bring in", or any variant where Tom is looking to identify investors to co-invest alongside him in a specific deal. Also trigger when Tom describes a round structure (check size, round size, stage) and asks who in his network would fit.
---

# Coinvestor Recommender

## Purpose

Given a specific deal opportunity and round structure, recommend the most relevant coinvestors from Tom's qualified network. Recommendations are ranked and filtered by: (1) sector/thesis fit, (2) check size fit, (3) co-lead appetite if relevant, and (4) relationship depth. Always surface deduplication logic to avoid recommending multiple people from the same firm.

---

## Step 1: Load the Knowledge Base

Read the full knowledge base at:
> `/home/claude/coinvestor-recommender/references/coinvestor-kb.md`

This is the source of truth for Tom's coinvestor network. Do not make up investor data or hallucinate details not in the file.

---

## Step 2: Extract Deal Parameters

From Tom's message or the broader conversation, extract:

- **Company name** — who is the deal for
- **Round size** — total target round (e.g., $1.5m pre-seed)
- **Tom's check** — how much Tom is writing (e.g., $1m)
- **Remaining capacity** — total round minus Tom's check
- **Target co-investor check range** — what size checks are needed to fill the round (e.g., $250k–$500k)
- **Co-lead appetite** — is Tom open to a co-lead (which would bump round size) or purely looking for follower checks?
- **Stage** — pre-seed, seed, Series A, etc.
- **Sector / thesis** — what does the company do? (Pull from Notion Opportunities DB or Tom's description)
- **Geographic preference** — any preference stated

If any of these are missing, infer from context. If truly unknown, note the assumption.

---

## Step 3: Score and Filter Investors

For each investor in the knowledge base, evaluate fit across four dimensions:

### A. Sector Fit (High / Medium / Low / None)
Match the company's sector against the investor's "Sector notes" field. Fintech deals → prioritize fintech-specialist funds (Anthemis, TTV, Springbank, Vesey). AI infrastructure → prioritize AI-thesis investors. Generalists fit most deals at medium.

### B. Check Size Fit (Strong / Marginal / Excluded)
Match the needed check range against the investor's "Check size range" field. If the investor's typical range is materially below the needed check (e.g., Dorm Room Fund for a $500k slot), flag as Excluded from round composition (though still potentially valuable for other reasons).

### C. Co-lead Fit (if applicable)
If Tom is open to a co-lead, filter to investors with "Co-lead appetite: Conditional or Yes" and check sizes that could plausibly anchor a larger round.

### D. Deduplication
Apply the firm deduplication rules from the knowledge base — recommend at most one person per firm unless Tom explicitly asks for multiple contacts at the same firm.

---

## Step 4: Output Format

Present the recommendations in two tiers when relevant:

### Category 1: Friendly Follower Check
List investors best suited to fill the remaining capacity as collaborative followers. For each:
- **Name / Firm / Role / Location**
- **Email** (for easy outreach)
- **Fit rationale** — 1–2 sentences on why they're a fit for this specific deal
- **Check size expectation**
- **Any relevant notes** (existing relationship context, past deal touchpoints)

### Category 2: Co-lead / Larger Check (if applicable)
List investors who could potentially anchor a larger round alongside Tom. For each:
- Same fields as Category 1, plus an explicit note on why they could co-lead

### Category 3: Angels (if applicable)
List angels whose five-figure check is justified by outsized value-add relative to check size. Only surface when the deal would genuinely benefit from what that angel brings. For each:
- **Name / Firm or affiliation / Role**
- **Email**
- **Value-add** — specific and concrete (e.g., domain expertise, customer intros, press network, operator credibility)
- **Check size expectation** — always five figures ($10k–$99k)

### Excluded / Not Recommended (brief)
Flag any investors who are clearly misfit for this deal (wrong sector, check size too small, geography mismatch) and explain why briefly. Do not surface these as recommendations.

### First-Pass Shortlist for [Company] (or any named company)
If Tom has already named specific investors he's considering, validate each one against the scoring criteria and confirm Follower Check, Co-lead / Larger Check, Angel, or excluded — and explain why.

---

## Step 5: Draft and Save Outreach Email

When Tom asks to draft an email (e.g., to share coinvestor recommendations with a founder), always do both of the following in the same step:
1. Display the email using the `message_compose_v1` tool in the Claude UI
2. Save the email as a Gmail draft using the `gmail_create_draft` tool — do not ask for permission, just do it

Email formatting conventions:
- Bullet format: `Name, Role @ Firm — one-liner rationale. LI URL`
- Section headers: **Follower Check** and **Co-lead / Larger Check** — no subtext or qualifiers after the header
- LinkedIn URLs trail each bullet on the same line, pulled from the People DB via Notion MCP
- Do not use "Tier 1 / Tier 2" framing — use the Follower Check / Co-lead framing consistently
- Voice should be Tom's (see writing-style skill if needed) — warm but direct, no boilerplate

---

## Knowledge Base Maintenance

Tom can update the knowledge base at any time by:
1. Editing `/home/claude/coinvestor-recommender/references/coinvestor-kb.md` directly
2. Telling Claude: "Update [investor name]'s [field] to [value]" — Claude will make the edit and confirm

When Tom adds new investors from a list, table, or screenshot, Claude should parse them and append new entries to the KB following the existing schema (all fields, deduplication notes if applicable).

Fields Tom should be prompted to add for any new investor:
- Relationship tier (Working / Warm / Cold)
- Known check size range (estimate if unknown)
- Stage focus
- Sector notes / thesis
- Co-lead appetite
- Any deal-specific notes

---

## Limitations and Honest Signals

- Check size ranges are estimates unless Tom has explicitly set them. Flag when a range is inferred rather than confirmed.
- Co-lead appetite is often "Unknown" — do not invent conviction here. Surface it as an open question.
- Sector notes in the KB may be incomplete. If the company is in an unusual sector, note when no strong sector-fit candidates exist in the network.
- Never recommend an investor Tom has explicitly passed on or had a negative interaction with, even if their profile fits.
