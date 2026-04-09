---
name: burn
description: Triggered by "monthly burn", "recurring bills", "subscriptions", "what do I pay for", "fixed costs", "how much are my bills"
---

# Monthly Burn Rate

Calculate total recurring obligations and break them down by category.

## Workflow

1. **Fetch recurring series.** Call the `query` MCP tool:
   ```json
   { "recurring": true }
   ```

2. **Fetch forecast.** Get the next 30 days of expected charges:
   ```json
   { "forecast": true, "forecast_days": 30 }
   ```

3. **Compute the burn rate.** Sum all active recurring series by frequency:
   - Monthly items: use amount directly
   - Weekly items: multiply by 4.33
   - Annual items: divide by 12
   - Present the **total monthly burn** as a single headline number

4. **Break down by category group.** Show each group's contribution:
   - Group name → monthly total → % of burn → item count
   - Sort by monthly total descending

5. **Flag notable items:**
   - Largest single recurring charge
   - Any items that increased in amount recently
   - Inactive series that may still be charging (last seen recently but marked inactive)
   - Subscriptions vs. bills (subscriptions are discretionary, bills are not)

6. **Upcoming charges.** From the forecast, list the next 5 charges due with dates and amounts.

## Tone

Stick to the facts. Report what's being spent without judgement. Just clear, plain-language observations.
