---
name: fishy
description: Triggered by "anything fishy", "suspicious transactions", "unusual spending", "anomalies", "something look off", "anything weird"
---

# Find Suspicious or Unusual Transactions

Scan recent transactions for anything that looks off.

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

3. **Look for duplicates.** Fetch detail transactions sorted by amount to spot same-amount charges on the same day:
   ```json
   { "detail": true, "period": "last_30d", "sort": "-amount", "limit": 100 }
   ```
   Scan for: same amount + same day from different descriptions (possible double charges), or unusually large single transactions.

4. **Check for new merchants with large charges.** Look at the top spending parties and flag any that are new (only appeared in the last 30 days) with amounts over $100.

5. **Report findings.** Organize by severity:
   - **Possible duplicates**: same amount, same day, different descriptions
   - **Unusually large**: transactions significantly above the party's average
   - **New + expensive**: first-time merchants with large charges
   - **Data quality**: uncategorized transactions, misclassified income, broken connections
   - **Nothing found**: if everything looks clean, say so

6. **For each flagged item**, show: date, amount, description, party, category, and why it was flagged.

## Tone

Stick to the facts. Flag what looks unusual without alarm — most "fishy" items turn out to be legitimate. Present findings neutrally and let the user decide what to investigate.
