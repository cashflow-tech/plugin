---
name: optimize-cards
description: Triggered by "optimize my cards", "which card should I use", "card recommendations", "maximize points", "credit card optimizer", "best card for", "rewards optimization", "point optimization", "card strategy", "am I using the right card". Use this skill whenever the user asks about credit card rewards, points, cashback optimization, or which card to use for purchases — even if they don't say "optimize" explicitly.
---

# Credit Card Optimizer

Analyze spending patterns and recommend which credit card to use for each category to maximize rewards.

## Why this matters

Most people leave 1-3% on the table by swiping the wrong card. A person spending $5k/month on a 1x card instead of a 3x card for dining is losing $120/year on that category alone. This skill connects real spending data to card reward structures so the user can see exactly where the money is leaking and what to do about it.

## Workflow

### Step 1: Discover the user's cards

Fetch accounts to find credit cards:
```json
query { "include": ["accounts"] }
```

Filter to accounts with type `credit_card`. Present the list and ask the user to confirm the rewards structure for each card. To reduce back-and-forth, suggest the likely rewards structure based on the card name using the reference table below — the user just needs to confirm or correct.

If a card name doesn't match anything in the reference table, ask the user directly: "What rewards does [card name] earn? (e.g., 2% on everything, 3x on dining)"

Record each card's rewards as a simple map: `{ category: multiplier }` with a `default` entry for uncategorized spend.

**Household note:** If multiple checking/debit accounts appear (e.g., different family members), note which ones have significant spend. These are candidates for authorized user cards later.

### Step 2: Pull spending by category

Fetch trailing 12 months of spending grouped by category:
```json
query { "by": ["category"], "type": "expense", "period": "trailing_12m", "top": 30 }
```

Then fetch spending broken down by both account and category to see which card is currently being used for what:
```json
query { "by": ["account", "category"], "type": "expense", "period": "trailing_12m" }
```

### Step 3: Drill into top merchants

For any category where the user asks specifically, or for the top 3-5 categories by spend, fetch merchant-level detail:
```json
query { "by": ["party"], "category": "<category_name>", "type": "expense", "period": "trailing_12m", "top": 10 }
```

This step matters because merchant-level data unlocks insights that category-level analysis misses:
- **Co-brand card opportunities**: If >$10k/year goes to a single airline (e.g., United $75k → United Club Infinite) or retailer (Amazon $17k → Amazon Prime Visa), a co-brand card often beats a general travel/cashback card for that merchant.
- **Portal booking angles**: High Airbnb or hotel spend can sometimes be routed through card travel portals for bonus multipliers (e.g., Capital One portal for 10x on hotels).
- **International spend signals**: Merchants in foreign currencies or clearly international locations flag the need for no-foreign-transaction-fee cards. If you see international merchants, mention this.

### Step 4: Map Cashflow categories to reward tiers

Cashflow categories don't map 1:1 to card bonus categories. Use this mapping:

| Card reward tier | Cashflow categories |
|---|---|
| Dining | Restaurants, Coffee, Takeout, Bars, Fast Food |
| Groceries | Groceries |
| Travel | Flights, Hotels, Rental Cars, Rideshare, Public Transit, Parking, Tolls |
| Gas | Gas |
| Streaming | Streaming, Subscriptions (streaming portion) |
| Entertainment | Entertainment, Movies, Sports, Concerts, Hobbies |
| Online shopping | categories where the party is Amazon, Walmart.com, etc. |
| Transit | Public Transit, Rideshare, Parking, Tolls |
| Drugstores | Pharmacy |
| Everything else | All remaining categories |

Some categories (like Rideshare) may qualify under both "travel" and "transit" depending on the card issuer. Use the more favorable interpretation for the user.

### Step 5: Calculate the optimization opportunity

For each spending category:
1. Look up the annual spend from the category query
2. Determine which card is currently being used (from the account+category breakdown — use the card with the highest spend in that category as the "current" card)
3. Look up that card's reward rate for this tier
4. Find the best card in the user's wallet for this tier
5. Calculate the annual difference

**Point valuation** — when comparing points across ecosystems, use these conservative estimates:
- Chase Ultimate Rewards: 1.7 cents per point
- Amex Membership Rewards: 1.5 cents per point
- Capital One Miles: 1.4 cents per point
- Citi ThankYou Points: 1.4 cents per point
- Cashback: 1 cent per cent (face value)

To compare: multiply `spend x multiplier x point_value` for each card option.

The **baseline** is a flat 2% cashback card. Any category earning less than 2% effective return is underperforming even a basic no-fee card.

### Step 6: Present recommendations

**Lead with the headline**: "You're leaving ~$X/year on the table" (sum of all optimization gaps).

Then show a table sorted by opportunity size (biggest savings first):

| Category | Annual Spend | Current Card | Rate | Best Card | Rate | Extra $/Year |
|---|---|---|---|---|---|---|

After the table, organize recommendations in priority order:

#### Quick wins (no new cards)
Categories where the user already has the right card but isn't using it — just needs to change the card on file or swipe differently. Also flag household members with high debit spend who should get authorized user cards on the best rewards card.

#### If you get one card
The single highest-impact new card given this user's specific spending. Show the math: annual rewards minus annual fee minus what the current best card already earns = net incremental value. This is the "start here" recommendation — don't overwhelm with a 5-card portfolio upfront.

#### Full portfolio (optional)
If the user's spend volume justifies multiple specialty cards, show the full optimized portfolio. Include co-brand cards surfaced from the merchant drill-down (e.g., a United card if United is the dominant airline). Include annual fee math for each card.

#### Annual fee check
For each card with a fee (existing or recommended), calculate whether the user's actual spend justifies the fee vs. downgrading to a no-fee alternative.

### Step 7: Wallet cheat sheet

End with a simple reference the user can screenshot or memorize:

```
Your card playbook:
  Dining      → [Card X]
  Groceries   → [Card Y]
  Gas         → [Card Z]
  Travel      → [Card W]
  Everything  → [Card V]
```

Only include tiers where the user has meaningful spend (>$500/year).

## Common card rewards reference

Use this to suggest reward structures when the user confirms their cards. These are approximate — always let the user correct.

**Chase**
- Sapphire Reserve: 3x dining, 3x travel, 1x other
- Sapphire Preferred: 3x dining, 2x travel, 1x other
- Freedom Flex: 5x rotating quarterly, 3x dining, 3x drugstores, 1x other
- Freedom Unlimited: 1.5x all, 3x dining, 3x drugstores

**Amex**
- Gold: 4x dining, 4x groceries ($25k cap), 3x flights, 1x other
- Platinum: 5x flights booked direct, 1x other
- Blue Cash Preferred: 6% groceries ($6k cap), 6% streaming, 3% transit, 1% other

**Capital One**
- Venture X: 2x all, 10x hotels/cars via portal
- Savor: 4% dining, 4% entertainment, 3% groceries, 1% other

**Citi**
- Custom Cash: 5% top category ($500/mo cap)
- Double Cash: 2% flat

**Wells Fargo**
- Active Cash: 2% flat
- Autograph: 3x restaurants, 3x travel, 3x gas, 3x transit, 3x streaming, 1x other

**Other**
- Apple Card: 3% Apple/select merchants, 2% Apple Pay, 1% other
- Amazon Prime Visa: 5% Amazon/Whole Foods, 2% restaurants/gas/transit, 1% other
- Discover It: 5% rotating quarterly, 1% other

## Optimize-cards-specific notes

Give clear numbers and specific actions. Don't caveat everything to death. If a card swap saves $15/year, say so and let the user decide whether it's worth the hassle; if there's a $500/year opportunity, make sure it stands out. (General voice guidance lives in the system prompt.)
