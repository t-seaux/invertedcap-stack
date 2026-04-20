---
name: deal-digest
description: Log a deal digest from Chris Oh (or any other source) into the Notion Notes database as a formatted research note. Trigger whenever Tom says "deal digest", "log this digest", "add digest to Notion", "save Chris Oh's notes", "log deals", "package up deals", or pastes/forwards a block of company notes with ARR, valuation, or fundraising data that looks like a periodic deal digest. Also trigger when Tom provides raw deal digest text and asks to format or structure it. Always trigger inline — no confirmation needed before acting.
---

# Deal Digest Logger

Log a deal digest into the Notion Notes database (`e8afa155-b41a-4aa2-8e9d-3d4365a11dfb`) as a well-structured research note with a benchmark summary at the top and formatted company entries below.

## Workflow

1. **Check for duplicates** — search Notion Notes DB for any existing page with a title matching "Deal Digest – [DATE]". If found, surface it to Tom and ask whether to update or create a new one. Do not create a duplicate silently.
2. **Parse the raw digest** — extract all companies, sections, and metadata from the source text (WhatsApp, pasted text, or forwarded message).
3. **Build the benchmark summary** — synthesize a structured benchmarks block at the top of the page (see Benchmark Summary section below).
4. **Format the body** — apply all formatting rules (see Formatting Rules below).
5. **Create the Notion page** — use `notion-create-pages` with the correct parent, title, category, and icon.

---

## Page Metadata

| Field | Value |
|---|---|
| **Parent DB** | Notes DB — `e8afa155-b41a-4aa2-8e9d-3d4365a11dfb` |
| **Title format** | `Deal Digest – [Month D, YYYY]` — e.g. `Deal Digest – March 25, 2026` |
| **Category** | `Research` — always set this, no exceptions |
| **Icon** | 🤝 |

---

## Benchmark Summary

Always generate a benchmark summary block at the top of the page before the main company list. It should contain the following sections, each as a **bold header** followed by a one-liner in italics on the next line, then the specific examples as inline prose on the line after that.

### Sections

**Revenue Velocity** — four sub-bullets, each with its own inline one-liner baked into the label before the examples:

- `*Scale — [one-liner on what best-in-class looks like and what multiple it commands]:*` → examples
- `*Growth / Series B+ — [one-liner]:*` → examples  
- `*Early Stage — [one-liner]:*` → examples
- `*Multi-Year Compounding — [one-liner]:*` → examples

**Megarounds**
*[one-liner on what drives premium valuations vs. compressed multiples at scale]*
[specific examples inline]

**Unit Economics**
*[one-liner anchoring what best-in-class looks like — GM%, NDR, profitability thresholds]*
[specific examples inline]

**Seed-Stage Standouts**
*[one-liner on what tier-1 seed signal looks like this cycle — round size, backer caliber, pedigree]*
[specific examples inline]

### Benchmark Construction Rules

- **Collapse ARR and revenue into one metric** — treat them as functionally equivalent throughout. Never distinguish between "ARR" and "revenue" in the benchmark summary.
- **Pair revenue with valuation wherever both are disclosed** — e.g. `Eleven Labs ($330M rev, ~3x YoY at $11B — ~33x multiple)`. This is the most useful context.
- **Revenue multiples** — compute implied rev multiples (valuation ÷ revenue) wherever both numbers are available. Surface the range across stage tiers.
- **Best-in-class framing** — each section should tell Tom what the ceiling looks like, not the median. What does the best company in this bucket look like, and what does that command at raise?
- **Seed standouts** — prioritize by round size and backer signal (Tier-1 firm + large round = standout). Pedigree (ex-OpenAI, ex-Deepmind, ex-Palantir) is also a strong signal.

---

## Formatting Rules

### All sections (Main, Seed, Other, Additional, etc.)

- **Each company gets its own bullet point** — never comma-separate multiple companies on a single line, even in the Seed and Other sections where the source text may be a blob.
- **Company name bolded up to the colon** — format as `- **Company Name:** commentary`. If there is no colon (e.g. Seed entries that are just a name + backer), bold just the name: `- **Name** (backer)`.
- **Do not alter the underlying text** — preserve all original wording, figures, and commentary verbatim. Formatting changes only.

### Section headers

Use `## Section Name` for each top-level section: `## Main`, `## Seed`, `## Other`, `## Additional`. Separate sections with a horizontal rule (`---`).

### Seed section

The source Seed section is typically a single comma-separated blob. Break it into individual bullets — one per name or company. Format:
- Named individuals: `- **First Last** (backer or context)`
- Named companies: `- **Company Name** (backer or context)`
- Entries with colons: `- **Company Name:** commentary`

### Other section

Similar to Seed — often a comma-separated list. Break into individual bullets. For entries with just a name + stage/backer in parens, use: `- **Name** (stage or backer)`.

---

## Source Handling

### WhatsApp via Beeper

Search Beeper for the Chris Oh chat and scan for the deal digest message using `search_messages` with keywords like `company`, `ARR`, `valuation`, or `raising`. The digest is typically a single long WhatsApp message. Note: Beeper's API truncates long messages — if the full text is not returned, ask Tom to paste it directly.

### Pasted text

If Tom pastes the digest text directly in conversation, use that as the source. No Beeper search needed.

### Forwarded email

Parse from the email body. Category and title rules still apply.

---

## Deduplication

Before creating a page, run `notion-search` for `"Deal Digest"` in the Notes DB. If a page with the same date already exists, surface the URL to Tom and ask: update the existing page, or create a new one?

---

## Example Page Structure

```
## Benchmark Summary

**Revenue Velocity**
- *Scale — Best-in-class is 2-3x+ YoY at $150M+ rev; commands 30-80x rev multiples at raise:* ...
- *Growth / Series B+ — Best-in-class is 50-500% YoY at $20-100M rev; typically prices at 40-70x:* ...
- *Early Stage — Best-in-class is 6-10x+ YoY at $5-20M rev; valuations range $100M-$1B:* ...
- *Multi-Year Compounding — Best-in-class shows 3+ consecutive years of step-change growth with a clear path to $100M+:* ...

**Megarounds**
*Largest disclosed financing events; valuations driven by market position and growth rate — best-in-class commands 50-120x rev at growth stage, compressing to 20-40x at scale.*
Anduril ($8B raise at $60B); Clickhouse ($200M rev at $15B — 75x); ...

**Unit Economics**
*Best-in-class SaaS shows 80%+ GM, 120%+ NDR, and profitability at sub-$50M rev.*
Wealth (90% GM, 120% NDR, $500M val); Pepper (135% NDR); ...

**Seed-Stage Standouts**
*Tier-1 seed signal means $10-55M raises at $100M-$1B post with top-decile backers; pedigree commands premium pre-product valuations.*
Elorian AI ($55M seed, Menlo); Simulated Labs ($25M Greylock, $150M post); ...

---

## Main

- **Eleven Labs:** Audio AI; $330M ARR (~3x YoY); profitable; raised at $11B
- **Fal:** Generative media inference; ...

---

## Seed

- **Bob McGrew** (ex-OpenAI Chief Research Officer; 8VC)
- **Reactor** (Fal for world models; LSVP + Amplify)
...

---

## Other

- **Surf** (Accel)
- **Applied Compute** (KP)
...

---

## Additional

- **Scowtt:** Sales automation; $3M ARR; Series A done
...
```


---

## Final Step: Classify the Note

After the Notion page has been created and the Opportunity relation (if any) has been set, run the `note-classifier` skill to assign the correct `Category` field.

Read the skill at `/Users/tomseo/.claude/skills/note-classifier/SKILL.md` and follow its classification logic against the note just created. Do not skip this step — every note created by this skill must have a Category set.
