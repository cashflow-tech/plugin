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
| **trips** | "tag travel", "organize trips", "trip tagging" |
| **burn** | "burn rate", "runway", "how long will my money last" |
| **fishy** | "anything fishy", "suspicious charges", "audit my spending" |
| **cards** | "optimize credit cards", "card rewards", "which card should I use" |
| **optimize-subs** | "audit subscriptions", "optimize subscriptions", "cancel subscriptions" |
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
- "Is there anything fishy in my transactions?"
- "Audit my subscriptions — what am I paying for?"
- "What's my burn rate?"
- "Tag my recent travel expenses"

## Links

- [cashflow.tech](https://cashflow.tech)
- [Documentation](https://cashflow.tech/docs)
- [Server discovery](https://cashflow.tech/.well-known/mcp.json)
