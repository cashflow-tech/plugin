---
name: recap
description: Generate a narrative financial review comparing spending, income, trends, and anomalies across any time period. Triggered by "monthly recap", "how did I do this month", "spending summary", "financial review", "weekly recap", "quarterly review", "year in review". Use this skill whenever the user asks about how they did financially over a period, wants a spending summary, or asks for a financial review — even casually like "how'd this month go?"
---

# Financial Recap

Generate a narrative financial review for any time period.

## Why this matters

A recap turns transactions into a story — what happened with the money, what changed, what's coming. The biggest risk is letting one bad data point — a tiny prior period, a one-off wire, a mass-duplicate row, a miscategorized cash deposit — drive the whole headline. Most of this skill is about not getting fooled.

## Arguments

`$ARGUMENTS` specifies the time period. Examples: "this month", "january", "Q1 2026", "last week", "2025". Defaults to the current month if no argument is given.

## Workflow

### 1. Determine the period
- "this week", "last week" → weekly
- "this month", "january", "jan 2025", "2025-01" → monthly (default)
- "this quarter", "Q1", "Q1 2025" → quarterly
- "this year", "2025", "year in review" → yearly
- Any explicit date range works too

### 2. Fetch the period summary with prior-period compare
```json
{ "period": "<detected>", "compare": "prior_period", "include": ["ratios", "anomalies", "accounts"] }
```
(Use `start`/`end` for explicit date ranges.) The `accounts` include returns balances and the cash forecast — both load-bearing later.

### 3. Fetch the year-ago summary, then decide if it's usable
```json
{ "start": "<same_period_last_year>", "end": "<same_period_last_year_end>", "include": ["ratios"] }
```
If the response returns `allZeroTotals`, or income+expenses < ~$50, or `ratios.caution` flags low income — there's no year-ago data to compare against. Drop the year-ago section entirely. Don't fabricate a sentence around empty data.

### 4. Fetch recurring bills + forecast
```json
{ "recurring": true }
```
Returns the bill series, frequency, next expected dates, and 30-day cash forecast (shortfall warnings, projected low balance).

### 5. Sanity-check before composing
Look for shapes that mean the headline is wrong:

- **Doubled rows.** If transaction count is way higher than expected, or `duplicates[]` is non-empty, fetch detail and check for identical (date, amount, description) pairs. Two Plaid items pointing at the same account, or a load-side dedup gap, can 2x every number. Don't report inflated totals as truth.
- **Improbably low income.** If headline income is < $50 but expenses are large, OR `top_cats` shows expense rows with negative amounts — real income is probably classified as Peer Payments / Cash & Checks (Zelle from a parent, family support, paycheck-via-P2P). Look at detail; describe what you see, don't report the math literally.
- **One-off pass-through transactions.** If a single transaction is >25% of period income/expenses, or a large inflow has a matching outflow within ~7 days, treat it as a one-off. Strip it from the headline and call it out separately. Don't let a $50K bonus or a $6M wire dominate the recap of a normal month.
- **Cash deposits at retailers.** Walgreens / 7-Eleven / branch "Cash Deposit" entries sometimes land in Pharmacy / Other Shopping. If the description says "Cash Deposit", treat as money in, not spending.

### 6. Synthesize
Lead with the answer. Tables for numbers, prose for context. Skip categories with trivial amounts.

**Always cover:**
- **Headline:** money in, money out, what's left. Dollars, not just percent.
- **vs. prior period:** dollar AND percent change — but **suppress the percent** when `|vs_prior.pct| > ~1000%` or the prior period was tiny/partial. The math is meaningless. Describe qualitatively instead ("March had a one-off $50K deposit; April returned to baseline").
- **vs. last year:** only if step 3 produced usable data. Otherwise skip silently.
- **Anomalies + recurring bills.** Unusual transactions; new, changed, or cancelled subscriptions.
- **Account balances.**

**Lead with this — don't bury it — when it applies:**
- **Forecast / runway.** If ending balance < ~$100, OR forecast shows any shortfall in the next 30 days, OR the user has 5+ cash-advance/EWA transactions in the period — the forecast IS the recap. Put it at the top. Name the date the balance turns negative and the bills driving it.
- **Borrowing churn.** "Cash Advance" lives in the Financing group, which is excluded from the `expenses` headline. With 5+ cash-advance transactions in the period, sum gross advances + fees and surface as its own line. Otherwise the headline number hides the user's biggest drain.

**Suppress:**
- Percent ratios (savings_rate, food_pct, discretionary_pct, etc.) whenever `ratios.caution` is set. The numbers are wrong. Use underlying dollar amounts.
- "Historical data still syncing" warnings when the recap period ended more than a week ago — the warning isn't time-bounded and is almost always stale for closed prior months.

(Voice, runway framing, EWA-as-annual-cost, gambling pattern + NCPG referral, and predatory-product APR math all live in the system prompt — don't restate them here.)
