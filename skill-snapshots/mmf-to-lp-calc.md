---
name: mmf-to-lp-calc
description: Calculate how much to transfer from money market fund (MMF) accounts to LP checking accounts at the start of each quarter for Dash Fund entities. Trigger when Tom says "MMF to LP", "quarterly transfer", "how much do I need to move from MMF", "run the MMF calc", "MMF transfer calc", or any variant asking about quarterly fund transfers from money market to LP checking accounts. Also trigger if Tom mentions moving money for Dash Fund II, Dash Fund II-A, or Fund I at the start of a quarter. If a Citi checking account snapshot is not provided, explicitly request it before proceeding — never run the calc without current balance data.
---

# MMF to LP Calculation Skill

## Purpose
At the start of each quarter, Tom moves funds from money market fund (MMF) accounts into the LP checking accounts for three Dash Fund entities. This skill computes the exact transfer amount needed for each entity based on fixed quarterly targets and current checking account balances.

## Fixed Quarterly Targets
These amounts are fixed each quarter and should be hardcoded as the targets:

| Entity | Target Amount |
|---|---|
| Fund I, a Series of Dash Venture Fund (Fund I LP) | $20,584.38 |
| Dash Fund II, LP (Fund II LP) | $109,639.28 |
| Dash Fund II-A, LP (Fund II-A LP) | $19,688.85 |

## Required Input: Citi Checking Snapshot

**If the user has not provided a screenshot or data export of current Citi checking balances, do not proceed. Instead, respond with:**

> "To run the MMF to LP calculation, I need a current snapshot of your Citi checking account balances for the LP entities. Please share a screenshot of your CitiBusiness Online deposit account summary."

Once provided, extract balances for the three LP accounts only. **Ignore management company (MC) accounts entirely** — Fund I MC, Fund II MC, and any other MC-suffixed accounts are excluded from all calculations.

### Freshness gate (MANDATORY — before any calculation)

The snapshot must be **dated within 24 hours**, verified from the date/time in the Citi snapshot header. If the header shows an older date, or no date/time is visible, do NOT compute transfer amounts. Stop and respond:

> "This snapshot is dated [date] / has no visible timestamp — balances may be stale. Please share a fresh CitiBusiness deposit account summary (or confirm these balances are current) and I'll run the calc."

Exception: if Tom explicitly confirms in the same message that the balances are current ("these are as of this morning"), proceed and note the confirmation in the output footer.

## LP Account Mapping
Match Citi account names to entities as follows:

| Citi Account Name | Entity |
|---|---|
| Fund I LP | Fund I, a Series of Dash Venture Fund |
| Fund II LP | Dash Fund II, LP |
| Fund II-A LP | Dash Fund II-A, LP |

Use the **Current Available (USD)** column from the Citi snapshot as the current balance for each account.

## Calculation Logic
For each entity:

```
Transfer Needed = Target Amount − Current Available Balance
```

If Current Available ≥ Target Amount, transfer needed is $0 (no transfer required — flag this to Tom as a note).

## Output Format
Lead with the transfer needed section, followed by one bullet per entity. Use this exact structure:

**Transfers Needed (MMF → LP Checking)**
- Fund I LP: **$[transfer amount]** (Current Checking: $[X] | Target: $[X])
- Fund II LP: **$[transfer amount]** (Current Checking: $[X] | Target: $[X])
- Fund II-A LP: **$[transfer amount]** (Current Checking: $[X] | Target: $[X])

*Balances as of [date and time from Citi snapshot header]*

If any entity already meets or exceeds its target, replace the transfer amount with "— at/above target, no transfer needed" but still show the parenthetical context.

Do not include any prose before the transfer section. The header and bullets should be the first thing in the response.
