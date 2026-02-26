---
name: recap
description: Triggered by "monthly recap", "how did I do this month", "spending summary", "financial review"
---

# Monthly Financial Recap

Generate a narrative monthly financial review.

## Workflow

1. **Determine the period.** If the user provided a month in `$ARGUMENTS` (e.g. "january", "jan 2025", "2025-01"), convert it to `start` / `end` date range params. Otherwise use `period: "this_month"`.

2. **Fetch summary data.** Call the `query` MCP tool:
   ```json
   { "period": "this_month", "compare": "prior_period", "include": ["ratios", "anomalies", "accounts"] }
   ```
   (Replace `period` with `start`/`end` if a specific month was requested.)

3. **Fetch recurring bills.** Call the `query` MCP tool:
   ```json
   { "recurring": true }
   ```

4. **Synthesize a narrative recap** covering:
   - **Headline numbers**: total income, total expenses, net cash flow, savings rate
   - **vs. last month**: notable changes (call out anything that moved more than ~15%)
   - **Anomalies**: any flagged unusual transactions or spending spikes
   - **Recurring bills**: new, changed, or cancelled subscriptions/bills
   - **Key ratios**: any ratios returned in the summary (e.g. expense-to-income)
   - **Account balances**: if returned, note current balances and changes

5. **Keep it conversational** — this is a personal finance review, not a spreadsheet. Use plain language, highlight what matters, and skip categories with trivial amounts.
