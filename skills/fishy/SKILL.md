---
name: fishy
description: Scan recent transactions for duplicates, unusual amounts, new expensive merchants, and data quality issues. Triggered by "anything fishy", "suspicious transactions", "unusual spending", "anomalies", "something look off", "anything weird", "check for fraud", "double charges". Use this skill whenever the user wants a health check on their transactions or suspects something might be wrong — even vague concerns like "something doesn't look right."
---

# Find Suspicious or Unusual Transactions

Scan recent transactions for anything that looks off.

## Why this matters

Most "fishy" transactions have innocent explanations — a merchant using a different billing name, a pre-authorization that posted twice, or a forgotten subscription. But catching the real problems (actual fraud, billing errors, zombie subscriptions) early saves money and hassle. The goal is to surface anything worth a second look, not to cause alarm.

## Arguments

`$ARGUMENTS` can specify a time period (e.g., "last week", "this month") or a focus area (e.g., "check for duplicates", "new charges"). Defaults to the last 30 days.

## Workflow

1. **Run health check.** Call the `query` MCP tool:
   ```json
   { "health": true }
   ```
   This surfaces: suspected misclassified income, uncategorized transactions, orphan parties, stale recurring series, broken bank connections.

2. **Check for anomalies in recent spending.** Fetch last 30 days with anomaly detection:
   ```json
   { "period": "last_30d", "type": "expense", "include": ["anomalies"], "by": ["party"], "top": 20 }
   ```

3. **Look for duplicates.** Fetch detail transactions sorted by amount to spot same-amount charges:
   ```json
   { "detail": true, "period": "last_30d", "sort": "-amount", "limit": 100 }
   ```
   **What counts as a possible duplicate:**
   - Same amount (within $0.01) on the same day from different descriptions — possible double charge
   - Same amount on the same day from the same party — possible processing error
   - Exclude known recurring items (e.g., a weekly $15 charge from the same party is normal)

4. **Check for unusually large transactions.** From the detail results, flag transactions that are:
   - More than **2x the party's typical amount** (e.g., a $200 charge at a restaurant where you usually spend $50)
   - Over **$500** from a party you've only seen once before
   - Any single transaction over **$1,000** from a non-recurring party

5. **Check for new merchants with large charges.** Look at the top spending parties and flag any that are new (only appeared in the last 30 days) with amounts over $100.

6. **Check recurring series for anomalies.** Fetch recurring data:
   ```json
   { "recurring": true }
   ```
   Flag:
   - **Amount changes**: recurring items where the latest charge differs significantly from the usual amount (possible price increase or billing error)
   - **Zombie subscriptions**: items marked inactive that still had a charge in the last 30 days
   - **Small recurring charges under $10** that the user may not recognize — these are a common pattern for unauthorized subscriptions

7. **Report findings.** Organize by severity:
   - **Possible duplicates**: same amount, same day, different descriptions
   - **Unusually large**: transactions significantly above the party's average
   - **New + expensive**: first-time merchants with large charges
   - **Recurring anomalies**: price changes, zombie subscriptions, suspicious small charges
   - **Data quality**: uncategorized transactions, misclassified income, broken connections
   - **Nothing found**: if everything looks clean, say so clearly — that's a good result

8. **For each flagged item**, show: date, amount, description, party, category, and a specific reason it was flagged. Be concrete — "2x your usual Uber amount" is better than "unusually large."

## Tone

Stay calm and factual. Most flagged items turn out to be legitimate — present them as "worth checking" rather than "problems." Don't cause alarm over a $5 discrepancy. If nothing looks off, say so plainly. The user wants reassurance as much as they want detection.
