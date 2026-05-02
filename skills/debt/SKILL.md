---
name: debt
description: Build a debt payoff plan — inventory all debts (Plaid-connected and not), put a number on what the debt is costing per year, and simulate snowball vs avalanche payoff strategies side-by-side so the user can pick. Triggered by "pay off debt", "debt payoff", "snowball", "avalanche", "when will I be debt free", "credit card debt", "student loan", "should I pay off X or Y first", "how much is my debt costing me", "debt strategy", "debt-free date", "extra principal", "best way to pay down", "I'm drowning in debt", "where do I start", and the slash invocation `/debt`. Use this skill whenever the user asks about paying off debt, comparing payoff strategies, or wants to understand what their debt actually costs them — even if they don't use the words "snowball" or "avalanche". Also the right skill when the user has consumer debt and asks "where do I start" or "make a plan" — the payoff plan is usually the answer. Trigger even when only some debts are connected via Plaid; this skill is built to handle partial coverage.
---

# Debt Payoff

Inventory the user's debts, headline what they cost per year, and simulate two payoff strategies (snowball and avalanche) side-by-side. The user picks. Don't pick for them.

## Why this matters

Debt has two costs the other skills already surface: monthly minimums (visible in `burnrate`) and interest charges that buy nothing (visible in `leaks`). What both miss is the **trajectory** — how long until this is gone, and what an extra hundred dollars a month actually does to that timeline. Without a payoff plan, every month is the same number. With one, every extra dollar has a visible effect on the debt-free date.

This skill exists to surface that trajectory and let the user act on it.

## How this fits with other skills

- `burnrate` shows minimum debt payments as part of the monthly fixed-cost floor.
- `leaks` calls out interest and finance charges as money bleeding out.
- `debt` is forward-looking: given today's balances and rates, what's the path out?

If the user asks "what am I paying every month," that's `burnrate`. If they ask "where is my money going wrong," that's `leaks`. If they ask "how do I get out from under this," it's this one.

## What we can see, what we have to ask

Cashflow sees transactions and account balances when Plaid is connected. It does not see the stated APR on every account, and the user's debts may not all be connected. Plan for the gaps:

| Signal | Source | What if missing |
|---|---|---|
| Balance | Plaid balance on `credit_card` / `loan` accounts | Ask the user. Hard to infer from payments alone for cards (revolving) or amortizing loans without the term. |
| Minimum payment | Recurring outflow series to the lender. Common misses: same-institution internal transfers (the CC and the checking are at the same bank), discretionary chunk payments, sub-monthly cadences (BNPL/EWA). | Ask the user. Plaid `liabilities.min_payment_amount` is a second source if the data layer exposes it. |
| Monthly interest | Description match on `INTEREST CHARGE` / `FINANCE CHARGE` / `PURCHASE INTEREST` first; `Interest Charges` category second. In practice the category is often empty even when the descriptions are present. | Use 0 for inference, but say so — could mean a 0% promo, paid-in-full, or simply no statements posted yet on a freshly connected card. |
| APR | Plaid liabilities (when exposed) → stated APR. Else inferred: `most_recent_month_interest × 12 / current_balance`. The 12-month-average version of this formula understates true APR when the user paid in full some months. | Ask the user. Confidence is low when fewer than 9 of the trailing 12 months had charges. |
| Whether a debt exists at all | Recurring outflow that looks like a payment to a lender (e.g., "STUDENT LN PMT", "SYNCHRONY BANK") even with no connected account | Ask the user — they may have a loan we can't see |

Treat this as a conversation. Inventory what the data shows first, confirm and fill gaps with the user, then simulate. A wrong APR on a focus debt makes the avalanche plan wrong; better to ask.

## Arguments

`$ARGUMENTS` is freeform. Common shapes:

- "make a plan" / "pay off my debt" — full inventory + simulation
- "snowball vs avalanche" — go straight to the comparison once the inventory is built
- "when will I be debt free" — emphasize the debt-free date at the user's current pace
- "how much is my debt costing me" — emphasize total annual interest as the headline; still build the inventory because they'll usually ask "what now"
- "what if I throw $X" — re-simulate at a new throw amount; reuse the prior inventory
- (empty) — start with the inventory and ask what they want to do with it

## Workflow

### Step 1: Pull the data (single parallel turn)

Issue these in one tool-use turn:

- `query { "include": ["accounts"] }` — find `credit_card` and `loan` accounts and their balances
- `query { "recurring": true }` — recurring outflows; many will be debt payments
- `query { "by": ["party"], "type": "expense", "period": "trailing_12m", "search": "INTEREST CHARGE" }` — interest hits per lender. **Description search is the primary path here.** The `Interest Charges` category is often empty in production data even when interest is being paid.
- `query { "period": "trailing_12m", "type": "budget", "include": ["ratios"] }` — surplus signal for Step 5. Use `trailing_12m` (not `trailing_3m`, which often collapses to a partial current month). `type: "budget"` excludes transfers and investment flows so the income figure isn't polluted by Schwab/Fidelity transfers for users with brokerage activity.

If the description-search returns nothing, also try `"FINANCE CHARGE"` and `"PURCHASE INTEREST"` (some lenders use different wording), then fall back to `query { "by": ["party"], "type": "expense", "period": "trailing_12m", "category": "Interest Charges" }`. If everything's empty, accept that interest isn't measurable for this user and say so in the headline.

**Read the `warnings` field on every response.** A `sync_incomplete` warning means the user has accounts still loading historical data — the picture you're computing on is partial. Surface that to the user before quoting numbers ("X is still syncing; these numbers may shift over the next day"), and weight your inventory accordingly: a debt that hasn't shown a payment yet may just not have synced.

### Step 2: Triage — is this the right conversation right now?

Before computing a plan, scan the data for signals that a payoff simulation isn't the right intervention. When any of these hold, **pivot, don't compute**:

- **No consumer debt.** All `credit_card` accounts at zero or near-zero, no loan accounts, no debt-shaped recurring outflows. Say so plainly: "I don't see consumer debt in your accounts. The plan you're asking about isn't needed." If they have a mortgage and want to discuss accelerating it, note that mortgage paydown depends on investment alternatives the agent doesn't see — that's a different conversation.
- **Clearable this month.** The debt is small enough that a lump payment is obviously the right move. Trigger when **either** total debt balance ≤ one month of trailing-12m surplus, **or** the balance is a small fraction of liquid (non-CD) savings — roughly 10% or less. Don't simulate a ten-month plan; say "you can clear this with $X from savings, this month" and stop. A multi-month projection on a balance the user could wipe tomorrow is theater.
- **Active debt-settlement program.** Recurring escrow outflows to a debt-settlement firm. Match specific signatures, not loose substrings — `CROSSROADS(FDR)`, `FREEDOM DEBT RELIEF`, `NATIONAL DEBT RELIEF`, `ALLEVIATE FINANCIAL`, `PACIFIC DEBT INC`, `FORTH ADVANCE`, etc. Plain "Crossroads" matches a retail chain, and "FDR" appears inside bank reference tokens; require the full program name or the parenthetical `(FDR)` tag. The user has chosen a path; a payoff sim presupposes they haven't, and can actively work against it (settlements often require keeping accounts delinquent). Surface what you see, ask whether they want a sim alongside the program (rare) or just to track the program's progress.
- **Can't cover minimums.** Sum of debt minimums approaches or exceeds monthly take-home — even with no extra. The fix isn't ordering the payoff; it's stabilizing cash flow first. Surface the gap, name the largest minimums, and stop.
- **Active EWA / payday churn.** Three or more EWA or payday providers (Earnin, Albert, Brigit, Dave, MoneyLion, etc.) appearing in `flows` over 90d — same heuristic as `enveloping`. The user is rolling the same paycheck multiple ways; sequencing other debts doesn't help until that exits.
- **Thin-data + chronic overdraft.** Trailing-12mo income < expenses, recurring NSF/overdraft fees, and salary-advance or micro-EWA draws — even if below the 3-provider threshold. Same conclusion as the EWA pivot: stabilize cash flow first.

When you pivot, be plain about what you saw and what conversation would help instead. Don't refuse to engage. Don't lecture.

### Step 3: Build the inventory

Combine the signals into one row per debt:

| Lender | Type | Balance | Min payment | Monthly interest | APR | Source |
|---|---|---|---|---|---|---|
| (e.g., Capital One) | credit card | $2,140 | $45 | $52/mo | 28.9% (inferred) | Plaid |
| (e.g., Discover) | credit card | $4,820 | $98 | $115/mo | 28.6% (inferred) | Plaid |
| (e.g., Sallie Mae) | student loan | unknown | $310 | unknown | unknown | recurring outflow only |

For `Source`, distinguish "Plaid" (connected, balance + interest visible), "recurring only" (we see the payment but not the account), and "user-provided" (filled in by hand).

Show the inventory to the user before simulating. Ask for any unknown balances and APRs that matter — start with the largest debts. A 5-minute conversation here is worth more than a more elaborate report. If the user doesn't know an APR off the top of their head, suggest checking the most recent statement; rough estimates are fine.

Drop Plaid-connected accounts at zero balance. Don't include them in the report.

**APR inference**: prefer the most-recent-month interest charge / current balance × 12. The 12-month average understates true APR when the user paid in full some months (no charge) or had a 0% promo period. If fewer than 9 of the trailing 12 months had charges, mark APR confidence as "low" and say so plainly — there's likely a promo period or paid-in-full month skewing the inference.

**Filter out leases.** Recurring outflows whose raw description contains `LEASE` or `RENT` are leases, not amortizing debt. Drop them from the inventory before simulating. (Toyota / Honda Financial / BMW Financial often appear as "TOYOTA ACH LEASE" — the lender name handles both leases and loans, so the description is the tell.)

**Flag loans with no observable repayment.** A connected loan account with a non-zero balance and zero recurring outflows toward the lender is worth surfacing — either the user isn't paying it (deferred or delinquent), or the payment goes through a route the agent doesn't see (payroll deduction, internal transfer). Don't silently include it in the simulation. Ask.

### Step 4: Headline the cost

State the total annual cost of debt as a single number, plain.

> "Last 12 months: $2,140 in interest. That's $178 a month going to nothing."

This is often the most motivating line in the whole report. Don't bury it under the inventory table — put it right after.

If a meaningful portion of debt is unconnected and we don't know its interest cost, say what we measured and what we couldn't: "$1,380 in measured interest over the past year, plus the student loan interest we can't see — actual total is higher."

### Step 5: Ask the throw amount

One question:

> "Above the minimums, how much can you put toward debt each month?"

If they don't know, use the surplus already pulled in Step 1 — `trailing_12m` with `type: "budget"` so investment transfers don't pollute it. Float it: "Over the past year you've had about $X/mo left after expenses. Want to use that, a different number, or run both?" Don't insist on a number.

Sanity-check the income figure before quoting. If trailing-12m income looks impossibly large (e.g., millions), there are likely investment transfers leaking past the `budget` filter — surface that and ask the user what number to use instead of fabricating a throw amount.

If the answer is zero, the simulator still runs — it just shows the minimums-only path with no acceleration. That's still useful: it gives the debt-free date as a function of doing nothing differently.

### Step 6: Simulate the payoff

**If only one debt has known balance + APR, skip the strategy comparison.** With one debt, snowball and avalanche converge — the order of one debt is irrelevant. Compute one paydown trajectory at the user's throw amount and output months-to-debt-free + total interest. Don't waste the user's attention on a side-by-side that has no delta.

**If you have neither a minimum payment nor a usable throw amount, don't simulate.** Even a single-debt trajectory needs at least one number to project forward. Output the inventory and the cost-of-debt headline, then ask the user for a minimum payment off the most recent statement, or a throw amount they're comfortable with. Better to stop honestly than fabricate a number.

**If two or more debts have known balance + APR**, run both side-by-side using simple month-by-month simulation:

- **Snowball**: order debts smallest balance first. Pay minimums on every debt. Add the throw amount to debt #1. When #1 clears, roll its minimum payment plus the throw onto debt #2. Repeat.
- **Avalanche**: same mechanic, ordered highest APR first.

For each strategy, output:

- **Months to debt-free**
- **Total interest paid over the plan**
- **Order of payoff** — which debt clears first, second, etc., with the month each clears

Then compute the delta. Avalanche typically saves some amount of interest. State it as a fact, not a recommendation:

> "Snowball: 31 months, $4,200 interest. First clear: Capital One ($2,140) in month 9.
> Avalanche: 29 months, $3,800 interest. First clear: Discover (28.9% APR, $4,820) in month 14.
> Avalanche saves $400 over the plan."

Skip debts with `unknown` balance from both simulations and note them: "Sallie Mae isn't in either plan — I don't have the balance. Once you have it, I can include it."

### Step 7: Present the tradeoff (only when there are two or more debts)

If Step 6 produced two strategies, present them side-by-side and explain:

Both strategies work. Avalanche is the math — it minimizes total interest. Snowball is the momentum — clearing a small debt fast feels like progress, and feeling like progress is what keeps people on the plan. The right answer depends on the person.

When the strategies are close (within a few hundred dollars over multiple years), say so: at that point the math difference is small enough that snowball's behavioral edge is probably worth more.

The system prompt commits to "where money writers disagree, present the tradeoff." This is the skill where that line gets used. Lay it out and let the user choose.

If Step 6 produced a single trajectory (one debt), skip this step entirely. Quote the months-to-zero, the total interest at the chosen throw amount, and ask whether the user wants to try a different throw to see how the date moves.

### Step 8: Re-runs and check-ins

When the user comes back later — "how am I doing on debt" — recompute the inventory and compare against the prior plan. Look for:

- **Balance went up** on a debt that should be falling. Fresh charges defeating the plan. Flag it plainly.
- **Ahead of schedule** — paid down faster than projected. Recompute the debt-free date.
- **Behind** — paid down slower (or made minimums only). Recompute and note what changed.

You don't need to persist the plan in the database. Recompute it each session from current balances and the user's stated throw amount. The data tells you the trajectory.

## Voice notes

- Lead with the dollars-per-year cost. That's the headline.
- Quote APRs as numbers, not adjectives. "28.9% APR," not "high-interest."
- Don't moralize. The user knows debt is a drag. Putting a number on it is the value-add.
- For BNPL stacks, name each provider with its own line — "Affirm $340, Klarna $180, Afterpay $95," not lumped as "BNPL debt."
- Skip "you should." Frame as "here's what the math says, here's what most people find sticks."

## What this skill deliberately does not do

- **No promo-rate / 0% APR teaser tracking.** If the user mentions a 0% intro period, factor it into the simulation by hand. Don't try to detect it.
- **No balance-transfer or consolidation recommendations.** Product-specific and outside scope. If the user asks: "That's outside what I track. The numbers above are what you'd need to evaluate any offer."
- **No HELOC or 401k-loan strategies.** Same reason.
- **No mortgage paydown modeling.** Mortgage interest is fine to mention if the user has it. Aggressive mortgage paydown depends on investment alternatives the agent doesn't see — that's its own conversation.
- **No "good debt vs bad debt" framing.** All debt has a balance, an APR, and a payoff date. Compute those.
