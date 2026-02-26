---
name: tidy
description: Triggered by "tidy up", "clean up transactions", "categorize uncategorized", "organize my transactions"
---

# Tidy Up Uncategorized Transactions

Batch-categorize uncategorized transactions by clustering similar ones and applying categories in bulk.

## Workflow

1. **Fetch uncategorized transactions.** Call the `query` MCP tool:
   ```json
   { "detail": true, "is_uncategorized": true, "period": "last_90d", "limit": 200, "sort": "-amount" }
   ```
   If `$ARGUMENTS` contains a time period (e.g. "this month", "last 30 days"), use that instead of `last_90d`.

2. **Cluster by pattern.** Group the results by normalized description or party name. For each cluster, note the count and total amount.

3. **Suggest categorization.** For each cluster, propose:
   - A **category** (pick from the user's existing categories)
   - A **party** name (the clean merchant/counterparty name)
   - Whether a **rule** would help (if the pattern appears 3+ times)

4. **Present to the user.** Show a table or list of clusters with:
   - Pattern / merchant name
   - Count of transactions
   - Total amount
   - Suggested category
   - Whether you recommend creating a rule

   Ask the user to approve, modify, or skip each cluster.

5. **Apply approved categories.** For each approved cluster, call the `annotate` MCP tool:
   ```json
   { "action": "categorize", "filter": { "search": "<pattern>" }, "category_name": "<approved_category>" }
   ```
   Also set the party if one was approved:
   ```json
   { "action": "set_party", "filter": { "search": "<pattern>" }, "party_name": "<approved_party>" }
   ```

6. **Create rules for recurring patterns.** For clusters where the user approved a rule:
   - First preview: `admin { "entity": "rule", "action": "preview", ... }`
   - Show the preview (how many existing transactions would match)
   - If user confirms, create: `admin { "entity": "rule", "action": "create", ... }`

7. **Summarize.** Report how many transactions were categorized, how many rules were created, and how many uncategorized transactions remain.
