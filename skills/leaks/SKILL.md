---
name: leaks
description: Comprehensive audit of money leaving the user's accounts in ways they're likely not tracking — subscriptions (active, zombie, price-creeped, free-trial converts), fees (overdraft, NSF, ATM, foreign-transaction, late, account, wire), interest charges and finance fees, predatory products (EWA, payday, BNPL stacking), gambling deposits with escalation patterns, and unintended leaks like duplicate charges and billing errors. Triggered by "where am I leaking money", "audit my subscriptions", "find hidden charges", "anything fishy", "what am I paying for", "check for duplicates", "fees", "interest", "subscriptions", "where's my money going wrong", "bank fees", "what's draining my account". Use this skill whenever the user asks about money leaks, recurring charges, fees, fraud, anomalies, suspicious activity, subscription audits, or wants a comprehensive look at what's bleeding from their accounts beyond their intentional spending.
---

# Find the Money Leaks

Audit money leaving the user's accounts in ways they probably aren't tracking. Subscriptions, fees, interest, predatory products, gambling, billing mistakes — anything that bleeds cash without a clear deliberate decision behind it.

## Why this matters

People notice the rent payment and the grocery bill. They don't notice the $4.99 forgotten free trial that converted last March, the $35 overdraft fee that hit twice last month, the $480/month in credit-card interest that buys nothing, the EWA fee on every paycheck, or the BNPL stack that's silently spreading across four providers. Each line is small. The annual total isn't.

This skill exists to surface the total. Not to lecture. Not to prescribe. To put a number on the bleeding so the user can decide what to do about it.

## Approach: prefer categories over keyword search

Most fee and interest activity is already classified into categories like "Bank Fees," "ATM Fees," "Interest Charges," "Late Fees." A category-grouped query gives you the right totals in one call and doesn't false-positive. Keyword search is only the right tool for things the user's category taxonomy doesn't cover — usually branded merchants like specific gambling sites, EWA apps, and BNPL providers.

So: **start by listing categories**. Then run grouped queries against the fee/interest categories. Only fall back to keyword search where categories don't exist.

## Workflow

The report has eight sections. Run them in order. Each is independent — if a section returns nothing, omit it (no "no fees found" filler).

### 0. Discover the user's category taxonomy (one query, used everywhere below)

```json
{ "entity": "category", "action": "list" }
```

(via the `admin` tool). Cache the result. You're looking for category names matching:

- Fees / Bank Fees / Bank Charges / ATM Fees / Foreign Transaction Fees / Late Fees / Service Charges
- Interest Charges / Finance Charges / Interest Expense
- Gambling
- BNPL & Installments / Cash Advance
- Anything that looks like a fee/interest classification in the user's specific taxonomy

### 1. Subscription leaks

One call does the finding. Cashflow decides what a subscription is, what kind it is, what competes, what is paid twice, what stopped and what looks new. Don't re-derive any of it from `recurring` or from detail rows: the read and this skill must agree.

```json
{ "subscriptions": true }
```

A subscription is a live recurring charge in Media (any) or Health › Gym. Its category decides, so the person moves one in or out by recategorizing. Bills — rent, utilities, insurance, loans — are never in it, so there is no rent-sized "leak" to explain away.

Surface, in order (skip what's absent):

- **`twice`** — one service paid on two series this cycle (two Audible memberships, Spotify Family and Individual, Google One on two cards). The clearest leak. Name both charges and accounts.
- **`overlap`** — competing services side by side, with the combined monthly and yearly cost ("Netflix, Hulu and Peacock: $46 a month"). Ask whether they still use each. Never tell them to cancel; keeping both is often deliberate. `bundled` names one they pay for that another already includes ("Amazon Prime includes Prime Video").
- **`looks_new`** — two or three month-apart charges that detection hasn't confirmed yet: often a trial that became paid. Ask whether it's theirs. A charge they tag `one-time` drops out.
- **Price changes** — any item with `price { was, now, since }`.
- **`stopped`** — listed but no charge in two cycles. Usually cancelled already; offer to mark it ended.
- **`check_category`** — a known subscription service filed under another category. Offer to recategorize it so it's counted.
- **`billed_through.apple` / `.google`** — see "Apple bills" below.

**The headline number for this section** is the yearly cost of `twice` plus anything they say they don't use. `total.yearly` is context ("you pay $1,140 a year in subscriptions"), not a leak.

#### Apple bills

Apple bills every App Store subscription — and one-off purchases — under one name, so the bank can't say which apps. `billed_through.apple.not_in_list` is the Apple money no listed subscription explains.

**If you can search the person's email:**

1. Search `from:no_reply@email.apple.com subject:"Your receipt from Apple."` over the last 120 days.
2. Match a receipt to a charge in `billed_through.apple.recent` by **the same total on the same day** (allow one day for a card that posts late).
3. Read the items. A line reading **"Renews \<date\>"** is a subscription; its plan line says the cadence ("(Monthly)", "(7 Days)"). Anything else — usage credits, rentals, in-app purchases — is a one-off. One receipt can hold both.
4. "Apple Account" on the receipt says whose it is. That is the only way to see two people's subscriptions on one card.
5. Receipts come in two formats. The newer one reads fine as plain text. The older table format loses its line items in plain-text conversion: read its HTML, and only for receipts that match a charge you need to explain (they run ~11k tokens each).
6. About half of Apple charges have no receipt email: renewal receipts are opt-in. Say so, and send them to iPhone Settings › [name] › Subscriptions (or reportaproblem.apple.com) for the rest.
7. With their OK, remember what you found, using service names only — never an Apple account email:
   - a receipt names the app behind an Apple series in the list → `admin { entity: "recurring_series", action: "rename", id, new_name: "Canva Pro (Apple)" }`;
   - an app billed alone on its renewal day each time, with no series → `admin { entity: "recurring_series", action: "create", name, frequency, transaction_ids }` from the matched `recent[].id`s. A subscription that only ever arrives inside a mixed receipt can't be declared — say what it costs and stop.

**Without email:** say how much Apple bills that the list can't explain, and where to look (iPhone Settings › [name] › Subscriptions; Android: play.google.com → Subscriptions). Don't guess app names from amounts.

#### Cancelling

Only when the person asks, one service at a time, with their explicit go-ahead for each — never because two services overlap.

- `via: "apple"` or `"google"` means it's cancelled at Apple or Google, **not on the service's own site**. This is the commonest way a cancellation silently fails.
- If you can use a browser, cancel and report exactly what the site confirmed. Otherwise tell them where to cancel, and to do it before `next`.
- Afterwards, give the yearly saving (the item's `yearly`) and when it should stop, then `admin { entity: "recurring_series", action: "update", id, active: false }`. If it charges again, it comes back by itself on the next review.

### 2. Fee leaks — categories first

**Fees and interest are different problems.** Fees (overdraft, ATM, late, foreign-transaction) are friction — the goal is zero. Interest is the cost of carrying debt — the goal is paying down balances, not switching cards. Keep them in separate queries so the report can speak to them differently. Don't bundle `Interest Charges` / `Finance Charges` into the fees query below; those belong in section 3.

Use the category list from step 0. Pick the categories that look fee-related, then one grouped query:

```json
{
  "period": "trailing_12m",
  "by": ["category"],
  "category": ["Bank Fees", "ATM Fees", "Foreign Transaction Fees", "Late Fees"],
  "include": ["accounts"]
}
```

(Use the user's actual category names from step 0 — they may differ.)

**Then add a per-account breakdown.** Different cards have different fee policies (e.g., some cards charge foreign-transaction fees, others don't). The biggest actionable insight in a fees audit is often "you got hit on the WF cards but not the X1 — use the X1 next time you travel."

```json
{
  "period": "trailing_12m",
  "by": ["account"],
  "category": ["Foreign Transaction Fees"]
}
```

(Repeat per fee type if it's worth splitting — usually only foreign-transaction and ATM fees vary meaningfully by account.)

**If a category mixes fee types** (a "Bank Fees" category often contains foreign-tx + service + wire + replacement card fees all bundled), add a party-level breakdown to separate them:

```json
{
  "period": "trailing_12m",
  "by": ["party"],
  "category": ["Bank Fees"]
}
```

That tells you whether the $1,200 in "Bank Fees" is one chronic problem (e.g., monthly maintenance) or a one-trip cluster of foreign-tx hits.

**Fall back to keyword search ONLY for fee types the category taxonomy missed.** If the user has no "Foreign Transaction Fees" category, search the description directly:

```json
{ "detail": true, "period": "trailing_12m", "search": "FOREIGN TRANSACTION", "limit": 100 }
```

But before you do, check if the category list shows similar coverage under a different name. **When a category covers the fee, never use detail-mode for the headline number** — high-volume fee categories (foreign-tx easily produces 500+ tiny rows on a single trip) waste tokens iterating per-transaction. Group by party within the category instead.

**Output for this section:**
- **Total fees, last 12 months** — the headline
- **By type** — overdraft, NSF, ATM, foreign-transaction, late, etc., with counts and totals
- **By account** when meaningful — cards with high fees vs cards with none
- **Frequency trend** — last 6 months vs prior 6, are fees rising or falling?
- **One-time vs chronic** — a one-trip cluster of foreign-tx fees ≠ ongoing problem; a steady drip of overdraft fees does.

### 3. Debt-cost leaks — categories first

Same playbook. From step 0, look for "Interest Charges" / "Finance Charges" / similar.

```json
{
  "period": "trailing_12m",
  "by": ["account"],
  "category": ["Interest Charges"]
}
```

Per-account breakdown matters here too — interest on a specific card identifies that card as the high-rate culprit.

**Fall back to keyword search if no category covers it:**

```json
{ "detail": true, "period": "trailing_12m", "search": "INTEREST CHARGE", "limit": 100 }
{ "detail": true, "period": "trailing_12m", "search": "FINANCE CHARGE", "limit": 100 }
```

When using keyword search, check returned rows for false positives — skip rate-disclosure notices, $0 rows, "INTEREST RATE INFORMATION," "0% INTEREST PROMO," etc. Look at amounts, not just description matches.

**Reversals and refunds**: interest months may include reversals (negative entries) when a charge was disputed or refunded. Report the **net total** for the year. If reversals exceed ~20% of gross charges, mention it — that suggests something unusual happened (a successful dispute, a billing error refund) worth a glance.

**For the net total, use the monthly time-series view, not by-account totals.** A by-account aggregate may collapse signs in a way that surfaces gross-positive figures. The monthly time-series:

```json
{
  "period": "trailing_12m",
  "granularity": "monthly",
  "category": ["Interest Charges"]
}
```

shows positive months (charges) and negative months (reversals) separately. Sum them to get the true net.

**Output:**
- **Total interest paid in the last 12 months** — net of reversals, the headline
- **By account** — "you paid $X on the Citi card, $Y on the Discover card" (use this for which-card identification, not for the headline)
- **Monthly trend** — is the interest cost rising or falling?
- **Effective monthly leak** — total ÷ 12, framed as "money you spent buying nothing"

### 4. Predatory-product leaks — categories first, then party breakdown

Cashflow's seed taxonomy includes two categories that capture most predatory-product activity:

- **`BNPL & Installments`** (under Credit Payments) — Klarna, Affirm, Afterpay, Sezzle, Zip, Perpay
- **`Cash Advance`** (under Financing) — EWA apps (Earnin, Dave, Brigit, MoneyLion, Klover, FloatMe, Empower, B9), payday loans

If the user's transactions are categorized, both groups are detectable in one call:

```json
{
  "period": "trailing_12m",
  "by": ["party"],
  "category": ["BNPL & Installments", "Cash Advance"]
}
```

That returns each provider with annual total and transaction count. No keyword search, no shotgun.

**Inspect the rows before reporting the headline number.** Both categories regularly catch things that aren't actually predatory:

- **Paybacks, not borrowing.** Rows like `Payment Thank You-Mobile`, `Payment Thank You - Web`, or `Direct Debit: <Bank>, Payment` are the user *paying back* a card or BNPL plan. These are not new loans and don't belong in the leak total. Filter them out of the BNPL & Installments figure or note the split. (Big credit-card autopays sometimes land in `Cash Advance` too — same rule.)
- **Personal/installment loans aren't BNPL or EWA.** Lendmark, Mariner, OneMain, Plain Green, World Finance, RISE Credit — these are traditional installment loans, not the small-bore predatory stack the section is meant to surface. Worth flagging if the user is carrying one (that's a debt they may want to discuss), but don't lump them in with EWA/payday APR-shaming.
- **Inbound EWA disbursements vs. outbound fees.** EWA apps deposit the advance (inbound) and later debit the payback (outbound). The advance principal isn't a leak. The *fees and tips* on top are. If the user's data has clean inflow/outflow signs, headline outflows minus inbound principal; if not, just split the rows by direction in the report.

**For BNPL specifically: stacking signal.** If 3+ distinct BNPL providers show activity in the last 90 days, flag it. Stacking correlates strongly with overdraft and impulsivity research. Run a 90-day version of the same query to check.

**True-cost math when fees are detectable** — for EWA/payday, a $5 fee on $100 advanced for 5 days is ~365% APR. If fee amounts are visible (sometimes EWA apps charge a separate "tip" or "expedited" fee row), show the math.

**Keyword search is a fallback, not the primary path.** Only run keyword searches if (a) the categories above returned nothing AND (b) the user has signals suggesting predatory product use anyway (chronic overdrafts, very low balances, paycheck-correlated big outflows). If you do fall back, the `search` field takes one literal phrase, not OR — run one query per pattern in parallel. Use specific strings: `EARNIN.COM` not `EARNIN`, `DAVE INC` not `DAVE`. Skip generic short matches like `ALBERT` (matches city names) or bare `NSF` (matches "Visa Direct NY"). Always inspect returned rows to confirm the brand is the merchant identity, not a substring.

### 5. Gambling leaks — category first, top-parties as fallback

Cashflow seeds a `Gambling` category under `Entertainment`. The Plaid PFC mapping routes `ENTERTAINMENT_CASINOS_AND_GAMBLING` directly there, so for any user with reasonably modern data the category query is the primary path:

```json
{
  "period": "trailing_12m",
  "by": ["party"],
  "category": ["Gambling"]
}
```

That returns each gambling counterparty (sportsbook, casino, lottery, daily-fantasy site) with annual total and transaction count.

**Sanity-check the rows before headlining a number.** The Gambling category routes off Plaid's PFC, which is broad. Watch for:

- **Vegas-vacation false positives.** A single large charge with `LAS VEGAS NV` in the description and no other gambling activity is more likely a hotel/restaurant on a trip than a casino-floor loss. If one row dominates the total and lives geographically (Vegas, Atlantic City, Reno), call it out as "looks like a single trip — could be lodging or dining mis-tagged" rather than reporting it as ongoing gambling spend.
- **Sweepstakes / skill-gaming apps.** Plaid PFCs apps like Papaya Gaming, AviaGames, Bingo Cash, Solitaire Cash, Funzpoints, Jackpota, and Blitz Win Cash as gambling. They are technically gambling-adjacent (sweepstakes-casino model), but most users wouldn't describe them as "gambling." Report them in their own bucket — "casual gaming/sweepstakes" — and don't apply the escalation/NCPG framing to them. Reserve that framing for clear sportsbook, casino, or daily-fantasy activity.
- **Obvious mis-tags.** If the party name doesn't fit any gambling pattern at all (e.g., a foreign bakery, a generic merchant), flag it as likely Plaid-PFC error and exclude from the total. Don't pretend the data is right just because the category says so.

**Fallback — top-parties scan.** Use this only when the category query returns empty. Some historical transactions predate the Gambling category seed; some merchants get mis-tagged by Plaid; some users may have gambling activity flowing through a peer-payment app (Venmo to a bookie) that won't carry a gambling PFC.

```json
{
  "period": "trailing_12m",
  "by": ["party"],
  "type": "expense",
  "top": 50
}
```

Eyeball the returned party list for known sportsbook / casino / lottery names. Common signatures:

- **Sportsbooks**: DraftKings, FanDuel, BetMGM, Caesars Sports, Barstool Sports, PointsBet, HardRock Bet, ESPN BET
- **Casinos**: Most casinos have the word "Casino" in the merchant name (be careful with restaurant chains like "Casino Pizza" — verify by amount pattern)
- **Lottery**: State lottery operators (varies by state); look for "LOTTERY" in the party name
- **Daily fantasy / prediction markets**: Underdog, Kalshi, PredictIt, Polymarket

**For high-frequency low-amount activity** (e.g., $5 lottery tickets a few times a week — won't crack top 50 by spend), also check by transaction count:

```json
{
  "period": "trailing_12m",
  "by": ["party"],
  "type": "expense",
  "top": 50,
  "sort": "n"
}
```

If both the category query and the top-50-by-count list show nothing gambling-shaped, the user is clean on this. Don't run further keyword searches.

**If activity surfaces**, compute escalation/clustering signals across all the gambling parties combined:

- **Last 90 days vs prior 90 days** — escalation signal (zero before, daily now is the highest-concern pattern)
- **Day-of-week distribution** — Thu–Sat clustering is the sports-betting tell
- **Paycheck correlation** — deposits within 0–3 days of recurring inflow events. Use `query { recurring: true, type: "income" }` to identify paycheck dates, then check whether gambling spikes follow.

Reporting rules (from the system prompt):

- Always state the pattern factually — annual total, frequency, escalation curve
- **For severe escalation OR clear paycheck-correlation OR daily activity**, mention the NCPG helpline once: 1-800-MY-RESET, ncpgambling.org/chat. Don't repeat. Don't lecture. Don't diagnose.
- For modest, infrequent activity (e.g., occasional lottery ticket), just report the number — no helpline.

### 6. Unintended leaks — anomalies and double charges

Anomalies appear automatically inside summary-mode output (in the `anomalies` field with `large_unusual`, `large_new_merchant`, `price_changes` sub-fields). No `include` parameter needed.

```json
{ "period": "last_30d", "type": "expense" }
```

Read the `anomalies` field from that response.

For double charges (one purchase charged twice — not a subscription paid twice, which is section 1's `twice`), fetch detail rows sorted by amount:

```json
{ "detail": true, "period": "last_30d", "type": "expense", "sort": "-amount", "limit": 100 }
```

Flag:
- **Possible double charges** — same amount, same day, different descriptions OR same party charged twice on the same day. Exclude known recurring (a weekly $15 charge from the same coffee shop is not a duplicate).
- **Unusually large from a familiar merchant** — appears in the `large_unusual` anomaly field; or detect manually by comparing detail rows to recent history of the same party
- **Large charges from new merchants** — appears in the `large_new_merchant` anomaly field
- **Price changes on recurring** — appears in the `price_changes` anomaly field

Tone: these are "worth a second look," not "fraud." Most have innocent explanations. Stay calm.

### 7. Data-quality issues that hide leaks

```json
{ "health": true }
```

The health response covers a lot of things (uncategorized, orphan parties, broken connections, suspected misclassified income, empty categories, stale recurring). For this skill, only surface the items that affect leak totals:

- **Suspected misclassified income** — large outflow miscoded as inflow could hide a real leak
- **Broken Plaid connections** — missing data means invisible leaks
- **Uncategorized large charges** — if there's an uncategorized $2,000 charge, the user should look at it before trusting the report

Skip the rest (orphan parties, empty categories — those are tidiness, not leaks). Frame the surfaced items as: "These might affect what I'm reporting. Worth fixing before trusting the totals."

### 8. Top-line summary

Lead the final report with:

- **Total annualized leak** — one number, in dollars. Sum these and only these:
  - Section 1's `twice` plus any subscription the person says they don't use (NOT `total` — the subscriptions they use are not a leak)
  - Section 2 fees, 12m
  - Section 3 interest, 12m net of reversals
  - Section 4 BNPL/Cash Advance fees and net outflows (after excluding paybacks and personal loans, per section 4's filters)
  - Section 5 gambling, after excluding sweepstakes apps and any flagged Vegas-vacation false positives
- **Three biggest line items** by annual cost
- **The single highest-leverage move** — usually one thing dominates (paying off the high-interest card, switching cards for foreign-tx fees, the subscription paid twice, a streaming overlap they confirm they don't need). Surface it explicitly so the user doesn't have to scan the whole report to find it. **Follow the numbers — if discretionary subs are bigger than fees + interest combined, the biggest move is in the stack, not the cards. Don't lead with fees just because that's section 2.**
- **Anything urgent** — escalating gambling pattern, BNPL stacking, chronic overdrafts, very high interest cost

## Output format

Lead with the summary number. Then sections in priority order: biggest leak first. Use compact tables, not paragraphs. Plain English, not jargon — see the system prompt for voice rules. Annual cost is the unit. The user knows their life better than you do.

Skip sections that returned nothing. Don't pad the report with "no overdraft fees found" — silence on a category means it's clean.

## Arguments

`$ARGUMENTS` may narrow the scope:

- "subscriptions only" / "fees only" / "interest only" / "gambling only" / "BNPL only" — run that section, skip the rest
- "last quarter" / "this year" / specific date range — narrow the period from the default trailing-12-months
- Anything else — run the comprehensive audit

If unclear, run the comprehensive audit. The headline number with a clear breakdown is almost always more useful than guessing what the user wanted to focus on.

## Performance notes

- **Cache the category list** from step 0 — don't re-fetch it for every section
- **Run parallel searches in batches** — the subscriptions query, the category-grouped fee query, and the keyword-search batch (predatory + gambling merchants) can all run in parallel
- **Prefer grouped queries over detail+sum** — when you need totals, ask for totals, don't pull rows and sum them yourself
- **Watch for high-volume categories** — foreign-transaction fees can produce hundreds of micro-charges. Use the grouped total; don't iterate the detail list
- **Keyword search with care** — short patterns false-positive. Inspect returned rows to confirm brand match before counting

## A note on tone

This skill exists because most users have no idea how much fees, interest, and forgotten subscriptions are costing them per year. Surfacing the number is the work. Don't moralize. Don't suggest specific cuts unless asked. The user will know what to do with the number.

For gambling specifically: bank data can show patterns but cannot diagnose addiction. Report the pattern, mention NCPG once where the pattern warrants it, then move on. Diagnosis is not your job.
