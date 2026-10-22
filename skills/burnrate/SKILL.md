---
name: burnrate
description: Calculate total recurring financial obligations, break them down by category, and flag changes. Triggered by "burn rate", "monthly burn", "recurring bills", "subscriptions", "what do I pay for", "fixed costs", "how much are my bills", "monthly obligations", "what are my recurring charges". Use this skill whenever the user asks about their recurring spending, fixed costs, or wants to understand their baseline monthly obligations.
---

# Monthly Burn Rate

Calculate total recurring obligations and break them down by category.

## Why this matters

Burn rate is the floor — the minimum you spend each month before any discretionary purchases. Understanding this number is the foundation of cash management: it tells you how much runway you have, whether your fixed costs are creeping up, and where the non-negotiable money goes. A $50/month subscription increase is invisible in a monthly recap but adds up to $600/year.

## Arguments

`$ARGUMENTS` can specify a focus area (e.g., "subscriptions only", "bills only") or a comparison period. Defaults to showing all recurring obligations with a prior-period comparison.

## Workflow

1. **Fetch recurring series and the subscription review.** Call the `query` MCP tool twice, in one batch:
   ```json
   { "recurring": true }
   { "subscriptions": true }
   ```

2. **Fetch forecast.** Get the next 30 days of expected charges:
   ```json
   { "forecast": true, "forecast_days": 30 }
   ```

3. **Fetch prior-period comparison.** Get last month vs. the month before to spot changes:
   ```json
   { "recurring": true, "compare": "prior_period" }
   ```

4. **Fetch take-home for % framing.** Pull paychecks so you can express burn as a share of income:
   ```json
   { "paychecks": true }
   ```

5. **Compute the burn rate.** Use `recurring.totals.monthly_expenses`, plus `monthly_financing` when present (loan principal). It is already normalized to a month and counts live series only — don't re-sum the `series` list, which also lists series that stopped charging.
   - Present the **total monthly burn** as a single headline number
   - **Frame as % of take-home**: divide total monthly burn by `paychecks.totals.monthly_avg` and present as "$X/mo = Y% of your take-home." Sethi's Conscious Spending Plan targets Fixed Costs at 50–60% of take-home — over 60% means the user is structurally over-committed; under 50% means there's slack for savings/investing. **Skip the % line entirely** if `paychecks.totals.monthly_avg` is missing, near zero, or `irregular_income_90d` is large relative to it (variable income makes the ratio meaningless).

6. **Classify obligations.** Split the burn in two, using Cashflow's definition rather than your own:
   - **Subscriptions** (could cancel tomorrow): `subscriptions.total.monthly` — every live subscription, by `kinds`
   - **Fixed obligations** (rent/mortgage, utilities, insurance, loan payments, property tax, car payment): the burn minus that
   - Present each group's total and percentage of burn

7. **Break down by category group.** Within each classification, show each group's contribution:
   - Group name → monthly total → % of burn → item count
   - Sort by monthly total descending

8. **Flag notable items:**
   - Largest single recurring charge
   - Subscriptions whose price changed: `price { was, now, since }` on `subscriptions` items
   - Subscriptions listed but no longer charging: `subscriptions.stopped`
   - New ones: `subscriptions.looks_new` (charges detection hasn't confirmed yet)
   - Paid twice (`subscriptions.twice`) and competing services (`subscriptions.overlap`) — ask, don't tell them to cancel

9. **Upcoming charges.** From the forecast, list the next 5 charges due with dates and amounts.

10. **Month-over-month delta.** If comparison data is available, report:
   - Net change in monthly burn vs. prior period
   - Which specific items drove the change (new additions, cancellations, price increases)

## Burn-specific notes

Distinguish between fixed bills the user can't easily change (rent, mortgage, insurance, utilities) and discretionary subscriptions they could cancel — that distinction is the most useful thing this skill produces. Make the numbers clear; let the user decide what to cut. Lead with the headline burn number, then the breakdown. (General voice guidance lives in the system prompt.)
