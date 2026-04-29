---
name: tidy
description: Incrementally categorize uncategorized transactions, fix miscategorized items, consolidate duplicate merchant names, and create rules for recurring merchants. Triggered by "tidy up", "clean up transactions", "categorize uncategorized", "organize my transactions", "fix categories", "clean up merchants". Use this skill whenever the user wants to improve their transaction data quality — even if they just say something like "my transactions are messy" or "lots of uncategorized stuff."
---

# Tidy Up Transactions

Incrementally clean up uncategorized and miscategorized transactions. Designed to run many times — each run makes a small dent (~10 items per phase).

## Why this matters

Uncategorized and miscategorized transactions make every other skill less accurate — recaps miss spending, burn rates are wrong, and card optimization can't see the real category breakdown. Tidying is foundational work that pays dividends everywhere.

## Prerequisites

Only the cashflow MCP tools are required. Every other data source (email, browser) is **opportunistic** — attempt each, use what's available, skip gracefully if unavailable.

## Researching Unknown Merchants

When you can't identify a merchant from the description alone, try these in order. Stop as soon as one works.

**General knowledge** — start here, it's free:
1. Many raw descriptions contain recognizable merchant names, processor codes, or abbreviations. Try to identify the merchant from what you already know.

**Web searches** (if available) — cheap, no subagent needed:
2. Web search the **cleaned/display name** (e.g. "Tock Inc")
3. Web search the **raw description** — processor codes, location info, or abbreviations often reveal more
4. Web search any **phone numbers or domain names** embedded in the raw description — highly specific identifiers

**Email searches** (if available) — search in the main agent, read in a subagent:
5. Email search the **exact dollar amount** including cents (e.g. "$47.23", not "$47") with a date range near the transaction date. Receipts almost always include the exact amount.
6. Email search the **cleaned merchant name** with a date range near the transaction date

If a promising email result comes back, use a **subagent** (Agent tool) to read the full message and extract just: merchant name, what was purchased, amount. Order confirmation emails are extremely verbose — never read them in the main context.

**Last resort:**
7. **Ask the user** — show the transaction details and ask what it is

## Re-runs

This skill is designed to run repeatedly. On subsequent runs:
- **Phase 1**: Only uncategorized transactions without rules are shown — prior runs' work is already applied.
- **Phase 2**: Skip categories where all top parties already have rules (check with `admin { "entity": "rule", "action": "list" }` and compare party names). Focus on parties without rule coverage.
- **Phase 3**: Skip clusters where a rule already covers the canonical name. Focus on new or unresolved clusters.

## Workflow

Use the period from `$ARGUMENTS` (e.g. "this month", "last 30 days"), or `last_90d` as default.

### Phase 1: Categorize uncategorized (biggest first)

Start with the highest-value uncategorized transactions — they have the most impact on accuracy.

1. **Fetch top 10 uncategorized expenses by amount:**
   ```json
   query { "detail": true, "is_uncategorized": true, "type": "expense", "period": "<period>", "limit": 10, "sort": "-amount" }
   ```

2. **Research each transaction** using the merchant research steps above. For each, determine:
   - What the merchant/party is
   - What category it belongs to
   - Whether it's likely to recur (and thus warrants a rule)

3. **Present findings as a batch.** Show a table with: description, amount, date, proposed party name, proposed category, and confidence level. Ask: "Any corrections before I apply these?"

4. **Apply fixes:**
   - For recurring merchants (appears more than once, or clearly a subscription/bill): create a rule
     ```json
     admin { "entity": "rule", "action": "preview", "conditions": { "description_pattern": "<pattern>" }, "actions": { "category_name": "<category>", "party_name": "<party>" } }
     ```
     Show match count, then create if confirmed. After creating rules, apply them retroactively:
     ```json
     annotate { "action": "apply_rules", "rule_ids": ["<new-rule-id-1>", "<new-rule-id-2>"] }
     ```
   - For one-offs: annotate directly (use `start`/`end` to scope to the tidy period)
     ```json
     annotate { "action": "categorize", "filter": { "search": "<pattern>", "start": "<period-start>", "end": "<period-end>" }, "category_name": "<category>" }
     annotate { "action": "set_party", "filter": { "search": "<pattern>", "start": "<period-start>", "end": "<period-end>" }, "party_name": "<party>" }
     ```

### Phase 2: Audit top expense categories

Review the most populated expense categories for miscategorized or messy transactions.

5. **Fetch top 10 expense categories by amount:**
   ```json
   query { "by": ["category"], "type": "expense", "period": "<period>", "top": 10 }
   ```

6. **For each category, fetch top parties:**
   ```json
   query { "by": ["party"], "type": "expense", "category": "<category_name>", "period": "<period>", "top": 10 }
   ```

7. **Scan for issues** in each category:
   - **Miscategorized parties** — a party that clearly belongs in a different category (e.g. a grocery store in "Shopping")
   - **Messy/duplicate party names** — same merchant with different names (e.g. "AMAZON" and "Amazon.com")
   - **Generic names** — vague names like "PURCHASE" or "POS DEBIT" that should be resolved to real merchant names
   - Research unfamiliar parties using the merchant research steps above

8. **Present findings as a batch.** Show what you found: which parties look miscategorized, which names need cleanup, which are unidentifiable. Ask: "Any corrections before I apply these?"

9. **Apply fixes:**
   - For parties appearing more than once: create rules
   - For one-offs: annotate directly
   - For name cleanup: use `set_party` to fix the canonical name

### Phase 3: Clean up clusters

Use the built-in cluster detection to consolidate duplicate merchant names.

10. **Fetch top 10 clusters:**
    ```json
    query { "clusters": true, "top": 10 }
    ```
    Clusters group variant spellings/truncations of the same merchant. Focus on clusters where variants are clearly the same merchant — skip clusters where variants are different businesses.

11. **For each actionable cluster:**
    - Research unknown merchants using the steps above
    - Check that variants share the same category — flag any that differ
    - Determine the best canonical name

12. **Present findings as a batch.** For each cluster show: proposed canonical name, variant names, count, suggested category. Ask: "Any corrections before I apply these?"

13. **Preview and create rules** for confirmed clusters:
    ```json
    admin { "entity": "rule", "action": "preview", "conditions": { "description_pattern": "<pattern>" }, "actions": { "category_name": "<category>", "party_name": "<canonical_name>" } }
    ```
    Show match count, then create if confirmed. After creating rules, apply them retroactively:
    ```json
    annotate { "action": "apply_rules", "rule_ids": ["<new-rule-id-1>", "<new-rule-id-2>"] }
    ```

### Wrap up

14. **Summarize** what was done: transactions categorized, rules created, party names cleaned up, clusters consolidated.

15. **Report remaining work:**
    ```json
    query { "detail": true, "is_uncategorized": true, "period": "<period>", "limit": 1 }
    ```
    Report the remaining uncategorized count from the response metadata. If there's more to do, suggest running `/tidy` again.

## Arguments

`$ARGUMENTS` specifies the time period to tidy. Examples: "this month", "last 30 days", "last_90d". Defaults to `last_90d` if no argument is given.

## Tidy-specific format

Batch findings into tables for review — don't narrate each transaction individually. (General voice guidance lives in the system prompt.)
