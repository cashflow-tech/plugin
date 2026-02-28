---
name: tag-trips
description: Triggered by "tag travel", "tag trips", "travel expenses", "organize trips", "trip tagging", "which trip was this"
---

# Tag Trips

Tag travel expenses with trip names by resolving each transaction one at a time, biggest first.

## Prerequisites

Only the cashflow MCP tools are required. Every other data source (email, calendar, browser) is **opportunistic** — attempt each, use what's available, skip gracefully if unavailable.

## Scratchpad

Maintain a scratchpad file at `/tmp/tag-trips.md` with a table of known trips:

```
| Tag | Destination | Start | End |
|-----|-------------|-------|-----|
| Hawaii Dec 2024 | Maui | 2024-12-20 | 2024-12-28 |
| Europe Summer 2025 | Europe | 2025-06-15 | 2025-07-10 |
```

Update this file as you go. If context compacts, re-read it to recover state.

## Workflow

### 1. Pull transactions

```json
query { "detail": true, "group": "Travel", "period": "last_90d", "sort": "-amount", "limit": 100 }
```

If `$ARGUMENTS` contains a time period, use that instead of `last_90d`.

### 2. Process each transaction

Work through the list top to bottom (biggest first). For each transaction:

**If already tagged** — if the tag isn't in the scratchpad yet, add it. Infer what you can: the tag name gives the destination and approximate dates (e.g. "Maui Dec 2024" → destination: Maui, dates: Dec 2024), and the transaction itself gives one concrete date. The date range may be partial at first — it gets refined as more tagged transactions are seen. Move on.

**If untagged** — resolve it:

#### a. Try to identify the trip via email (if available)

Search email in the main agent using this fallback chain:
1. Confirmation number (if visible in the raw description)
2. Exact dollar amount (no rounding — e.g. "$1,268.51")
3. Party name + approximate transaction date

If a promising result comes back, use a **subagent** (Task tool) to read the full message and extract just: travel dates, route/destination, passengers, amount. Confirmation emails are extremely verbose — never read them in the main context.

#### b. Try web search (for hotels, tours, unfamiliar parties)

If email doesn't work or isn't available, and the party is a hotel, tour operator, or unfamiliar name: web search the party name, phone number, or domain from the raw description. Often the name or country code in the description reveals the location. Use the transaction date as the travel date.

#### c. Ask the user

If you can't resolve it, ask. Show the transaction details and ask which trip it belongs to, or whether to skip it.

#### d. Match to a known trip

Each trip in the scratchpad has three fields: **tag**, **destination**, and **date range**. Match by destination + date overlap:

- **Close match** (same destination, dates overlap or within a few days): expand the existing trip's date range to include this transaction. Tag with the existing trip's tag.
- **No match** (new destination, or dates far from any known trip): create a new trip entry in the scratchpad with a tag, destination, and date range. **Trip naming**: "Destination Mon YYYY" format (e.g. "Maui Dec 2024"). Multi-city trips use the primary destination or "City1-City2 Mon YYYY".

#### e. Tag it

```json
annotate { "action": "tag", "transaction_ids": ["<id>"], "tag_name": "Maui Dec 2024" }
```

### 3. Next batch

If `has_more: true`, use the `cursor` to fetch the next 100. Later batches are progressively smaller charges that mostly cluster around already-discovered trips. Repeat Step 2.

### 4. Report

```json
query { "by": ["tag"], "group": "Travel", "period": "last_90d", "type": "expense" }
```

Present a summary table of trips by tag with totals. Note any large untagged transactions that couldn't be resolved.

### 5. Offer to tag non-travel expenses

Ask: "Want me to look for dining, rideshare, or other expenses during your trip dates?" If yes, query those categories within each trip's date range and present candidates for confirmation. Foreign transactions (country codes like NZL, GBR in raw descriptions) are high-confidence candidates.

## Key Rules

- **One transaction at a time**: Resolve, tag, move on. Don't try to batch-plan all trips upfront.
- **Biggest first**: `-amount` sort means anchors (flights, hotels) come first and define trips for smaller charges later.
- **Email is the best source**: Confirmation emails have actual travel dates and destinations. Transaction dates are often booking dates.
- **Subagents for email reads**: Search in the main agent, read in a subagent. Confirmation emails are huge.
- **Scratchpad is durable state**: Update it after each tagging. Re-read after context compaction.
- **Conservative on non-travel**: Don't auto-tag dining/rideshare. Offer as follow-up in Step 5.
- **Multi-account by default**: Don't filter by account — households travel together.

## Tone

Stick to the facts. Present findings and suggestions without judgement — just clear, plain-language observations and actionable options.
