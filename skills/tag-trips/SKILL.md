---
name: tag-trips
description: Triggered by "tag travel", "tag trips", "travel expenses", "organize trips", "trip tagging", "which trip was this"
---

# Tag Trips

Discover trips from transaction patterns and external sources, then tag related expenses with trip names. Trips are defined by **filter expressions** — declarative, idempotent, and context-compaction-safe.

## Prerequisites

Only the cashflow MCP tools are required. Every other data source (Gmail, Calendar, browser) is **opportunistic** — attempt each, use what's available, skip gracefully if unavailable. The workflow always progresses even if every external source fails.

## Workflow

### 1. Gather data

Run three queries in parallel, plus fetch existing tags:

```json
query { "detail": true, "group": "Travel", "period": "last_90d", "sort": "-amount", "limit": 200 }
query { "detail": true, "is_uncategorized": true, "amount_min": 50, "period": "last_90d", "sort": "-amount", "limit": 200 }
admin { "entity": "tag", "action": "list" }
```

If `$ARGUMENTS` contains a time period, use that instead of `last_90d`.

Then, for each existing tag that looks like a trip name (e.g. "Tokyo Mar 2026", "Hawaii Jan 2026"), fetch its tagged transactions:

```json
query { "detail": true, "tag": "Tokyo Mar 2026", "period": "all", "limit": 200 }
```

These tagged transactions define existing trips — their dates, destinations, and parties are the strongest signal for trip discovery.

### 2. Identify unknown parties

For expenses with unrecognized party names, web search the raw description to identify the business. Look for the business name, and check any domain names or phone numbers in the description. Often the party name alone is enough to categorize (e.g. "MARRIOTT WAIKIKI" → hotel in Honolulu). Skip this step for already-known parties (airlines, hotel chains, Airbnb, etc.).

### 3. Discover trips

Build the trip list from three sources, in priority order:

**Existing tags** (highest confidence): Each trip-named tag is an established trip. Pull date range and destination from its tagged transactions. These trips are already confirmed — the question is whether they have *untagged* expenses that should be added.

**Transaction clustering** (primary discovery): Among *untagged* Travel-category expenses:
- **Anchor transactions**: Round-trip flights and multi-night hotel stays define date ranges and destinations.
- **Clustering**: Group Travel expenses that fall within a few days of each other. A flight + hotel in the same week = a trip.
- **Date correction**: Flight transaction dates are often booking dates, not travel dates. If a flight doesn't cluster with hotels/other travel expenses, check Gmail for the airline + amount to find the actual travel dates. If ambiguous, present it to the user rather than guessing.
- **Foreign country codes**: Search raw descriptions for 3-letter ISO country codes (NZL, GBR, MEX, JPN, etc.) within trip date windows. These are high-confidence trip indicators even for non-Travel categories (coffee shops, groceries, restaurants abroad).

**External sources** (all optional — try each, skip gracefully if unavailable):
- **Gmail**: Search for receipts by dollar amount or party name. Extract actual travel dates and destinations from itineraries/confirmations. Especially useful for flights booked far in advance.
- **Google Calendar**: Search for travel-related events (flight numbers, hotel names, city names) to confirm or discover trip dates.
- **Kayak** (browser, if available): Check past and upcoming trips for aggregated trip info.

If none of these sources are available, proceed with tags + transactions alone.

### 4. Present trip list for confirmation

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

### 5. Match expenses to confirmed trips

For each confirmed trip (both existing and new), search across all accounts by default (households may have multiple people/accounts on the same trip). Find matching *untagged* Travel-group expenses:
- Flights, Hotels, Rental Car, Vacation category expenses whose travel dates fall within the trip window
- Allow a pre-booking buffer (flights can be months ahead) and post-travel buffer (+7 days for delayed charges)
- Include foreign transactions (identified by country codes in raw descriptions) as high-confidence candidates
- Do NOT auto-tag non-Travel-category expenses (dining, rideshare, etc.) — these are ambiguous

For expenses where the transaction date doesn't match any trip window, use email receipts (if available) to determine actual travel dates before giving up.

### 6. Tag with dry_run preview

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

### 6b. Verify & correct

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
