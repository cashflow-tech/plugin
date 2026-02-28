---
name: tag-trips
description: Triggered by "tag travel", "tag trips", "travel expenses", "organize trips", "trip tagging", "which trip was this"
---

# Tag Trips

Tag travel expenses with trip names. Two phases: silently cluster transactions into candidate trips, then present the full grouping to the user for review.

## Prerequisites

Only the cashflow MCP tools are required. Every other data source (email, calendar, browser) is **opportunistic** — attempt each, use what's available, skip gracefully if unavailable.

## Task list as working state

Use the **todo tools** (TaskCreate / TaskUpdate) to track progress. This state survives context compaction and is visible to the user.

During the clustering phase, create one task per candidate trip:
- **Subject**: "Tag: Morocco Mar 2026 (5 txns, $34k)"
- **Description**: tag name, destination, travel dates, transaction IDs, key charges, and any notes about confidence or open questions

Also create a task for unresolved transactions that couldn't be matched to any trip.

After the user reviews and approves groupings, work through the tasks — tag each trip's transactions and mark the task completed.

## Scratchpad

Optionally maintain `/tmp/tag-trips.md` with a table of confirmed (tagged) trips for quick reference:

```
| Tag | Destination | Start | End |
|-----|-------------|-------|-----|
| Hawaii Dec 2024 | Maui | 2024-12-20 | 2024-12-28 |
```

## Booking dates vs travel dates

Transaction dates are often **booking dates**, not travel dates. A hotel booked in December may be for a trip in March. Never assume the transaction date is when travel happened. Use these signals to determine actual travel dates:

- **Email confirmations** — the best source: they contain actual travel dates, routes, destinations
- **Location in raw description** — country codes (NZL, EGY, GRC), city names, state abbreviations
- **Activity type** — airport parking, in-flight wifi, and on-site charges (restaurants, gift shops) are same-day; flights, hotels, and rentals are often booked weeks ahead

Airline names are **not** destination signals — Alaska Airlines flies everywhere, not just Alaska.

## Domain knowledge

- **Turo trip codes**: Raw descriptions include a month code — `TRIP DE` = December, `TRIP JA` = January, `TRIP FE` = February. This tells you which month the rental was for, even if booked earlier.
- **Outdoorsy**: RV/camper rental platform. Similar booking-ahead pattern.
- **Country codes in raw descriptions**: `NZL`, `GBR`, `EGY`, `GRC`, `JOR`, `DEU` etc. — strong destination signals.
- **PayPal transfers**: `PAYPAL INST XFER` prefixed with a date (e.g. `260131`) — that's the transfer date, followed by the merchant name.

## Workflow

### 1. Pull transactions

```json
query { "detail": true, "group": "Travel", "period": "last_90d", "sort": "-amount", "limit": 100 }
```

If `$ARGUMENTS` contains a time period, use that instead of `last_90d`. Paginate with `cursor` until all transactions are fetched.

### 2. Cluster into candidate trips (silent — no user interaction)

Scan all transactions and group them into candidate trips. Work biggest first — large flights and hotels are the anchors that define trips.

For each transaction:

**If a refund** (positive amount) — pair it with the original charge.

**If already tagged** — add the tag to the scratchpad if not already there.

**If untagged** — identify the trip:

#### Researching a transaction

When you can't tell which trip a transaction belongs to from the description alone, try these in order. Stop as soon as one works.

**Email searches** (if available) — search in the main agent, read in a subagent:
1. Email search the **confirmation number** if one is visible in the raw description — highly specific
2. Email search the **exact dollar amount** including cents (e.g. "$1,268.51") with a date range near the transaction date
3. Email search the **party name** with a date range near the transaction date

If a promising email result comes back, use a **subagent** (Task tool) to read the full message and extract just: travel dates, route/destination, passengers, amount. Confirmation emails are extremely verbose — never read them in the main context.

**Web searches** (if available) — for hotels, tours, unfamiliar parties:
4. Web search the **cleaned/display name** (e.g. "Tock Inc")
5. Web search the **raw description** — country codes, location info, or abbreviations often reveal the destination
6. Web search any **phone numbers or domain names** embedded in the raw description

**Can't resolve** — mark it as unresolved for now. Don't ask the user yet.

#### Match to a known trip

Each trip in the scratchpad has three fields: **tag**, **destination**, and **date range**. Match by destination + **travel date** overlap (not transaction date):

- **Close match** (same destination, travel dates overlap or within a few days): add to that trip.
- **No match**: create a new candidate trip. **Trip naming**: "Destination Mon YYYY" format (e.g. "Maui Dec 2024"). Multi-city trips use the primary destination or "City1-City2 Mon YYYY".

### 3. Present to user for review

Show the complete proposed grouping — all candidate trips with their transactions. For each trip show: tag name, destination, travel date range, transaction count, total amount, and a few representative charges.

Also list any unresolved transactions with their details.

The user corrects, confirms, renames, or regroups in one pass. This is where all human input happens — **batch it into a single review** rather than asking about each transaction individually.

### 4. Tag

Apply tags based on the user-approved grouping:

```json
annotate { "action": "tag", "transaction_ids": ["<id1>", "<id2>", ...], "tag_name": "Maui Dec 2024" }
```

Tag in bulk per trip — not one transaction at a time.

### 5. Report

```json
query { "by": ["tag"], "group": "Travel", "period": "<same period as Step 1>", "type": "expense" }
```

Present a summary table of trips by tag with totals. Note any large untagged transactions that couldn't be resolved.

### 6. Offer to tag non-travel expenses

Ask: "Want me to look for dining, rideshare, or other expenses during your trip dates?" If yes, query those categories within each trip's date range and present candidates for confirmation. Foreign transactions (country codes like NZL, GBR in raw descriptions) are high-confidence candidates.

## Key Rules

- **Biggest first**: `-amount` sort means anchors (flights, hotels) come first and define trips for smaller charges later.
- **Batch human input**: Do all research silently, then present the full grouping for review. Don't ask about each transaction individually.
- **Email is the best source**: Confirmation emails convert booking dates into actual travel dates and destinations.
- **Subagents for email reads**: Search in the main agent, read in a subagent. Confirmation emails are huge.
- **Scratchpad is durable state**: Update it after each tagging. Re-read after context compaction.
- **Conservative on non-travel**: Don't auto-tag dining/rideshare. Offer as follow-up in Step 6.
- **Multi-account by default**: Don't filter by account — households travel together. But note that email confirmations may only be in one person's inbox.

## Tone

Stick to the facts. Present findings and suggestions without judgement — just clear, plain-language observations and actionable options.
