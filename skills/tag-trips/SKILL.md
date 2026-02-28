---
name: tag-trips
description: Triggered by "tag travel", "tag trips", "travel expenses", "organize trips", "trip tagging", "which trip was this"
---

# Tag Trips

Discover trips from transaction patterns and external sources, then tag related expenses with trip names. Trips are defined by **filter expressions** — declarative, idempotent, and context-compaction-safe.

## Prerequisites

Only the cashflow MCP tools are required. Every other data source (email, calendar, browser) is **opportunistic** — attempt each, use what's available, skip gracefully if unavailable. The workflow always progresses even if every external source fails.

## Scratchpad

Maintain a scratchpad file at `/tmp/tag-trips.md` throughout the workflow. Write to it as you go and re-read it if context gets compacted. Track:

- **Discovered trips**: name, dates, destination, status (confirmed/pending)
- **Resolved anchors**: which big charges have been looked up, what trip they map to
- **Unresolved charges**: big transactions still needing email lookup or user input
- **Batch progress**: which cursor you're on, how many batches completed

Update the scratchpad after each major step (anchor resolution, trip confirmation, tagging batch). This is your durable state — if the conversation compacts, read the scratchpad to recover where you left off.

## Workflow

### 1. Gather data

Run these queries in parallel:

```json
query { "detail": true, "group": "Travel", "period": "last_90d", "sort": "-amount", "limit": 100 }
query { "detail": true, "is_uncategorized": true, "amount_min": 50, "period": "last_90d", "sort": "-amount", "limit": 100 }
admin { "entity": "tag", "action": "list" }
```

If `$ARGUMENTS` contains a time period, use that instead of `last_90d`.

**Work in batches of ~100 transactions.** The first query returns the 100 largest Travel transactions. Discover trips, tag, and verify this batch before fetching the next page. Sorting by `-amount` means each batch is progressively less important — the biggest anchors come first, and later batches are smaller charges that cluster around already-discovered trips. If there are more results (`has_more: true`), use the `cursor` to fetch the next batch after finishing the current one.

The tag list tells you which trips already exist. Tagged transactions in the current batch show you those trips' date ranges and destinations — no need to fetch them separately.

### 2. Discover trips

Build the trip list from three sources, in priority order:

**Existing tags** (highest confidence): Each trip-named tag is an established trip. Pull date range and destination from its tagged transactions. These trips are already confirmed — the question is whether they have *untagged* expenses that should be added.

**Anchor-first discovery** (primary method): Work the biggest untagged charges first — large airfares and multi-night hotel stays are trip anchors. They define the destination and travel dates for the entire trip.

1. From Step 1's transaction list, work down the biggest untagged charges — these are the anchors.
2. **Look up each anchor in email** (if available) — search email in the main agent for the airline/hotel name + approximate amount or confirmation number. Search results (subject lines, snippets) are small and help you pick the right message. Then use a subagent (Task tool) to *read* the matching message and return just the trip details (travel dates, route/destination, passengers). Confirmation emails are extremely verbose (legal boilerplate, baggage policies, etc.) and will bloat the main context if read directly. The subagent extracts just what's needed: e.g. "SFO→HNL Mar 12–19, Alice + Bob, $890." This is the fastest way to pin down a trip — the transaction date alone is often the *booking* date, not the travel date. Without email, cluster by transaction dates instead.
3. Once an anchor defines a trip (dates + destination), all smaller untagged Travel expenses in that window cluster naturally.

- **Clustering**: Group remaining Travel expenses that fall within a few days of an anchor. A flight + hotel in the same week = a trip.
- **Unknown parties**: For smaller charges with unrecognized party names, web search the raw description to identify the business. Look for the business name, and check any domain names or phone numbers in the description. Often the party name alone reveals the location (e.g. "MARRIOTT WAIKIKI" → hotel in Honolulu). Skip this for already-known parties (airlines, hotel chains, Airbnb, etc.).
- **Foreign country codes**: Search raw descriptions for 3-letter ISO country codes (NZL, GBR, MEX, JPN, etc.) within trip date windows. These are high-confidence trip indicators even for non-Travel categories (coffee shops, groceries, restaurants abroad).

**Other external sources** (all optional — try each, skip gracefully if unavailable):
- **Calendar**: Search for travel-related events (flight numbers, hotel names, city names) to confirm or discover trip dates.
- **Kayak** (browser, if available): Check past and upcoming trips for aggregated trip info.

If no external sources are available, proceed with tags + transaction clustering alone.

### 3. Present trip list for confirmation

Present trips in batches of 5-8. For each trip, show the **filter expression** that defines it (not raw transaction lists) so the user can review and adjust the definition. Use AskUserQuestion for structured yes/no confirmations.

**Existing trips with new candidates:** For each tag-seeded trip, show:
- Trip name (already established)
- Date range and destination (from tagged transactions)
- Count of already-tagged transactions
- Filter expression for new untagged candidates
- Ask: tag these new ones too?

**Newly discovered trips:** For each transaction-clustered trip, show:
- Proposed trip name (format: "Destination Mon YYYY", e.g. "Tokyo Mar 2026")
- Date range (travel dates, not booking dates)
- Anchor transactions (the flights/hotels that defined the trip)
- Filter expression that would capture the trip's expenses

Ask the user to confirm, adjust names, merge/split trips, or add trips the skill missed.

### 4. Match expenses to confirmed trips

Step 2 discovered trips from the biggest charges. Now do a comprehensive sweep for *all* untagged Travel expenses that fall within each confirmed trip's date window — including smaller charges that weren't part of the initial discovery.

Search across all accounts by default (households may have multiple people/accounts on the same trip):
- Flights, Hotels, Rental Car, Vacation category expenses within the trip window
- Allow a post-travel buffer (+7 days for delayed charges)
- Include foreign transactions (identified by country codes in raw descriptions) as high-confidence candidates
- Do NOT auto-tag non-Travel-category expenses (dining, rideshare, etc.) — these are ambiguous

For expenses where the transaction date doesn't match any trip window, use email receipts (if available) to determine actual travel dates before giving up.

### 5. Tag with dry_run preview

Use filter-based tagging with `dry_run` to preview matches before committing:

```json
annotate { "action": "tag", "filter": { "group": "Travel", "start": "2025-12-15", "end": "2026-01-05" }, "tag_name": "Home for Christmas", "dry_run": true }
```

Review the `preview` array and `matched` count. If correct, re-run without `dry_run`:

```json
annotate { "action": "tag", "filter": { "group": "Travel", "start": "2025-12-15", "end": "2026-01-05" }, "tag_name": "Home for Christmas" }
```

For foreign transactions, use search filter:

```json
annotate { "action": "tag", "filter": { "search": "NZL", "start": "2025-09-01", "end": "2025-09-13" }, "tag_name": "NZ Sep 2025", "dry_run": true }
```

For curated lists (individual transactions cherry-picked after review), use `transaction_ids` directly — no dry_run needed since the list is already deliberate.

### 6. Verify & correct

After tagging each trip, query it back:

```json
query { "detail": true, "tag": "NZ Sep 2025", "period": "all" }
```

If any transactions were mistagged, untag them by name:

```json
annotate { "action": "untag", "transaction_ids": ["<id>"], "tag_name": "NZ Sep 2025" }
```

### 7. Report

Use tag grouping for the summary:

```json
query { "by": ["tag"], "period": "last_year", "type": "expense" }
```

For per-trip detail with category breakdown:

```json
query { "tag": "Tokyo Mar 2026", "by": ["category"], "period": "all" }
```

Also present:
- List of large travel expenses that couldn't be matched to any trip (for manual review)
- Any newly discovered trips the user might want to investigate further

### 8. Offer to tag non-travel expenses

After the main tagging is done, ask: "Want me to look for dining, rideshare, or other expenses during your trip dates that might be travel-related?" Include foreign transactions (identified by country codes in raw descriptions) as high-confidence candidates worth including. If yes, query those categories within each trip's date range and present candidates for manual confirmation via dry_run preview.

## Key Rules

- **Trip naming**: "Destination Mon YYYY" format. Multi-city trips use primary destination or "City1-City2 Mon YYYY".
- **Existing tags are trip seeds**: Re-running extends existing trips with newly discovered expenses, making it incremental.
- **Filter expressions over UUIDs**: Prefer filter-based tagging. Filters survive context compaction, can be reviewed and re-applied. Only fall back to `transaction_ids` for individually curated picks.
- **dry_run before commit**: Always preview filter-based tags with `dry_run: true` first.
- **Conservative on non-travel categories**: Don't auto-tag dining/rideshare during trip dates. Offer it as a follow-up in Step 8.
- **Foreign currency**: Receipt amounts may differ from USD transaction amounts. Search email by party name + approximate amount.
- **Multi-account by default**: Don't filter by account unless the user asks to.

## Tone

Stick to the facts. Present findings and suggestions without judgement — just clear, plain-language observations and actionable options.
