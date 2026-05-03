# Cashflow — Claude Code Plugin

Personal finance ledger for AI agents. Connect your bank accounts, then talk to your money.

## Install

```
/plugin install cashflow
```

Or from the marketplace:

```
/plugin marketplace add https://github.com/cashflow-tech/plugin.git
/plugin install cashflow@cashflow
```

## What you get

### MCP Tools

Three tools connected to [cashflow.tech](https://cashflow.tech) via OAuth:

| Tool | Purpose |
|------|---------|
| **query** | Spending summaries, time series, recurring bills, transaction detail, forecasts, anomalies, income statements, health checks |
| **annotate** | Categorize transactions, set parties, tag, mark transfers, detect recurring |
| **admin** | Manage categories, rules, accounts, parties, tags, bank sync |

### Skills

| Skill | Trigger phrases |
|-------|-----------------|
| **recap** | "monthly recap", "how did I do this month", "spending summary", "quarterly review" |
| **tidy** | "tidy up", "categorize uncategorized", "clean up transactions" |
| **leaks** | "where am I leaking money", "audit my subscriptions", "anything fishy", "find hidden charges", "fees" |
| **burnrate** | "burn rate", "recurring bills", "what do I pay for", "monthly obligations" |
| **enveloping** | "envelope budgeting", "paycheck splitting", "separate bills from spending", "what's safe to spend" |
| **debt** | "pay off debt", "snowball vs avalanche", "when will I be debt free", "how much is my debt costing me" |
| **cards** | "optimize credit cards", "card rewards", "which card should I use" |
| **trips** | "tag travel", "organize trips", "trip tagging" |
| **benefits** | "am I missing any benefits", "do I qualify for SNAP", "money is tight" |
| **setup** | First-time onboarding and account setup |

## How it works

1. First use triggers OAuth sign-in at cashflow.tech
2. Link your bank accounts via Plaid (Wells Fargo, Chase, BofA, and 12,000+ institutions)
3. Ask Claude about your finances in natural language

## Examples

- "How much did I spend on dining this month?"
- "Give me a monthly recap"
- "What are my recurring bills?"
- "Tidy up my uncategorized transactions"
- "Show me my spending by category for the last 3 months"
- "Where am I leaking money?"
- "Audit my subscriptions — what am I paying for?"
- "What's my burn rate?"
- "Snowball vs avalanche — what's the fastest way to pay off my debt?"
- "Help me set up envelope budgeting"
- "Tag my recent travel expenses"

## Links

- [cashflow.tech](https://cashflow.tech)
- [Documentation](https://cashflow.tech/docs)
- [Server discovery](https://cashflow.tech/.well-known/mcp.json)
