---
name: burn
description: Calculate total recurring financial obligations, break them down by category, and flag changes. Triggered by "monthly burn", "recurring bills", "subscriptions", "what do I pay for", "fixed costs", "how much are my bills", "monthly obligations", "what are my recurring charges". Use this skill whenever the user asks about their recurring spending, fixed costs, or wants to understand their baseline monthly obligations.
---

# Monthly Burn Rate

Calculate total recurring obligations and break them down by category.

## Why this matters

Burn rate is the floor — the minimum you spend each month before any discretionary purchases. Understanding this number is the foundation of cash management: it tells you how much runway you have, whether your fixed costs are creeping up, and where the non-negotiable money goes. A $50/month subscription increase is invisible in a monthly recap but adds up to $600/year.

## Arguments

`$ARGUMENTS` can specify a focus area (e.g., "subscriptions only", "bills only") or a comparison period. Defaults to showing all recurring obligations with a prior-period comparison.

## Workflow

1. **Fetch recurring series.** Call the `query` MCP tool:
   ```json
   { "recurring": true }
   ```

2. **Fetch forecast.** Get the next 30 days of expected charges:
   ```json
   { "forecast": true, "forecast_days": 30 }
   ```

3. **Fetch prior-period comparison.** Get last month vs. the month before to spot changes:
   ```json
   { "recurring": true, "compare": "prior_period" }
   ```

4. **Compute the burn rate.** Sum all active recurring series by frequency:
   - Monthly items: use amount directly
   - Weekly items: multiply by 4.33
   - Annual items: divide by 12
   - Present the **total monthly burn** as a single headline number

5. **Classify obligations.** Break recurring items into two groups:
   - **Fixed obligations** (non-discretionary): rent/mortgage, utilities, insurance, loan payments, property tax, car payment — things you can't easily cancel
   - **Discretionary recurring** (subscriptions): streaming, gym, software, meal kits, memberships — things you could cancel tomorrow
   - Present each group's total and percentage of burn

6. **Break down by category group.** Within each classification, show each group's contribution:
   - Group name → monthly total → % of burn → item count
   - Sort by monthly total descending

7. **Flag notable items:**
   - Largest single recurring charge
   - Any items that **increased in amount** recently (compare current amount to historical average — Cashflow's recurring series tracks amount variance)
   - Inactive series that may still be charging (last seen recently but marked inactive)
   - New recurring charges that appeared in the last 30-60 days
   - Charges that seem duplicative (e.g., two streaming music services)

8. **Upcoming charges.** From the forecast, list the next 5 charges due with dates and amounts.

9. **Month-over-month delta.** If comparison data is available, report:
   - Net change in monthly burn vs. prior period
   - Which specific items drove the change (new additions, cancellations, price increases)

## Burn-specific notes

Distinguish between fixed bills the user can't easily change (rent, mortgage, insurance, utilities) and discretionary subscriptions they could cancel — that distinction is the most useful thing this skill produces. Make the numbers clear; let the user decide what to cut. Lead with the headline burn number, then the breakdown. (General voice guidance lives in the system prompt.)
