---
name: tidy
description: Triggered by "tidy up", "clean up transactions", "categorize uncategorized", "organize my transactions"
---

# Tidy Up Uncategorized Transactions

Categorize uncategorized transactions in two phases: cluster duplicates first (high leverage), then research remaining one-offs.

## Prerequisites

Only the cashflow MCP tools are required. Every other data source (email, browser) is **opportunistic** — attempt each, use what's available, skip gracefully if unavailable.

## Researching Unknown Merchants

When you can't identify a merchant from the description alone, try these in order. Stop as soon as one works.

**Web searches** (if available) — cheap, no subagent needed:
1. Web search the **cleaned/display name** (e.g. "Tock Inc")
2. Web search the **raw description** — sometimes has processor codes, location info, or abbreviations that reveal more
3. Web search any **phone numbers or domain names** embedded in the raw description — these are very specific identifiers

**Email searches** (if available) — search in the main agent, read in a subagent:
4. Email search the **exact dollar amount** including cents (e.g. "$47.23", not "$47") with a date range near the transaction date. Receipts almost always include the exact amount.
5. Email search the **cleaned merchant name** with a date range near the transaction date

If a promising email result comes back, use a **subagent** (Task tool) to read the full message and extract just: merchant name, what was purchased, amount. Order confirmation emails are extremely verbose — never read them in the main context.

**Last resort:**
6. **Ask the user** — show the transaction details and ask what it is

## Workflow

Use the period from `$ARGUMENTS` (e.g. "this month", "last 30 days"), or `last_90d` as default.

### Phase 1: Cluster duplicates (highest leverage)

One rule covers many transactions now and in future imports — find clusters first.

1. **Find clusters.** Use the built-in clusters query mode:
   ```json
   query { "clusters": true, "top": 25 }
   ```
   This returns party name clusters sorted by transaction count, with variants showing different spellings/truncations of the same entity. Focus on clusters where variants are clearly the same merchant (case differences, truncation, trailing punctuation) — skip clusters where variants are different businesses that happen to have similar names.

2. **For each actionable cluster:**
   - Research unknown merchants (see above)
   - Propose a category and party name to the user
   - Show: canonical name, variant names, count, suggested category
   - If user approves: preview the rule (`admin { "entity": "rule", "action": "preview", ... }`), show match count, create if confirmed
   - If user skips: move on

### Phase 2: Research the leftovers (one-offs)

What remains after clustering is genuinely unique — worth individual attention.

3. **Fetch remaining uncategorized.** Top 100 by amount:
   ```json
   query { "detail": true, "is_uncategorized": true, "period": "<same period>", "limit": 100, "sort": "-amount" }
   ```

4. **For each transaction:**
   - Research unknown merchants (see above)
   - If likely to recur: create a rule
   - Otherwise annotate directly:
     ```json
     annotate { "action": "categorize", "filter": { "search": "<pattern>" }, "category_name": "<category>" }
     annotate { "action": "set_party", "filter": { "search": "<pattern>" }, "party_name": "<party>" }
     ```

### Wrap up

5. **Summarize.** Report how many transactions were categorized, how many rules were created, and how many uncategorized transactions remain.

## Tone

Stick to the facts. Present findings and suggestions without judgement — no commentary on spending habits. Just clear, plain-language observations and actionable options.
