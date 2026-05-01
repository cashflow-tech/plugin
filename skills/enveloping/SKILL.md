---
name: enveloping
description: Help the user set up or review an envelope budgeting system — physically separating non-discretionary bills from discretionary spending money so the bills are always covered and the rest is guilt-free. Use this skill whenever the user asks about envelope budgeting, paycheck splitting, automating bill payments, separating bills from spending, structuring multiple accounts, setting up a "bill pay" or "fixed costs" account, "what should I move to my bill account each paycheck", "how do I make sure my rent is covered", "how much can I actually spend", or references Ramit Sethi's Conscious Spending Plan or any three-bucket framework. Trigger even when the user doesn't know the term "enveloping" — phrases like "I keep accidentally spending my rent money", "I want to know what's safe to spend without worrying", "I want my bills to come out of a different account", or "I want to pay myself first" all mean this skill.
---

# Enveloping (Envelope Budgeting)

Help the user design a paycheck → accounts flow where fixed bills are always covered and the rest is theirs to spend without thinking about it.

## Why this matters

The simplest budgeting trick that actually works is physical separation: keep the money you owe (rent, utilities, insurance, loan payments) in a different account from the money you spend day-to-day. Once the "bill account" has enough to cover the month, whatever is in the spending account is genuinely yours — no anxiety about whether you can afford lunch, no surprise overdraft on rent day.

The math is unglamorous: figure out monthly non-discretionary spend, figure out monthly take-home pay, divide by paychecks per month, sweep that amount on payday. The hard part is doing it once, automating it, and trusting the system.

This is the practical core of Ramit Sethi's Conscious Spending Plan — his three-bucket model splits take-home into fixed costs, savings/investing, and guilt-free spending. Sethi suggests rough percentages (50–60% / 10–20% / 20–35%) but those are rules of thumb for someone starting from scratch, not a target every household should match. Some households save aggressively (savings rate well above 20%); others have lumpy variable comp that distorts any percentage. Use the framework, ignore the percentages when they don't fit.

Most people get the bulk of the benefit just from separating fixed costs out — call it tier 1. Multi-bucket sinking funds (vacation, gifts, car maintenance) are tier 2, only useful once tier 1 is running clean.

## Arguments

`$ARGUMENTS` is freeform. Common shapes:
- "set up enveloping" / "help me set this up" — assume no system in place, recommend from scratch
- "review my enveloping" / "is this right?" — assume something is in place, evaluate it
- "how much should I move per paycheck" — go straight to the sweep number
- (empty) — figure out where they are by looking at their accounts and flows first

## Workflow

### Step 1: Pull the data (single parallel turn)

Issue all five in one tool-use turn:

- `query { "recurring": true }` — non-discretionary obligations on the expense side
- `query { "paychecks": true }` — income side with cadence already labeled, plus trailing-90d non-recurring income (variable comp signal)
- `query { "account_summary": true, "period": "last_90d" }` — per-account inflow/outflow/net/balance/n in one shot
- `query { "flows": true, "period": "last_90d" }` — inter-account transfer matrix
- `query { "by": ["group"], "type": "expense", "period": "last_90d" }` — sanity-check what's actually flowing through fixed-cost categories

### Step 1.5: Triage — is this an enveloping question right now?

Before computing a sweep, scan the data for signals that enveloping isn't the right intervention yet. When any of these hold, **pivot, don't compute** — name the specific reason and what conversation to have instead.

- **Overdrawn or perpetually near-zero account.** Any checking account currently below zero, or repeatedly bumping zero across the period. Sweeping money into a "bill pay" account from a hub that's already negative just relocates the overdraft.
- **Recurring expense ≥ take-home.** When the obligations you'll identify in Step 2 add up to more than monthly take-home, no sweep number works — the math is broken before envelopes enter the picture. Surface the gap, name the largest offending recurring lines, and stop.
- **EWA / payday-loan churn.** Three or more of {Earnin, Albert, Brigit, Dave, Cleo, Bright Money, Klover, Grid, Advance America, Tilt, MoneyLion, ActiveHours, Empower, Possible Finance, Oasiscre, Atlas, Current advances} appearing in `flows` over 90d. The user is borrowing against their next paycheck multiple ways; the fix is exiting that cycle, not separating bills from spending money. Often pairs with high-interest installment lines (Affirm, Klarna, Uplift, GreenSky) draining the same account — count those toward the same signal.

When you pivot, be plain: "Before envelopes make sense, the thing eating the cushion is X." Don't soften it into a lecture, and don't refuse to engage — just be honest that sweep math doesn't help here and offer the conversation that does.

### Step 2: Compute the numbers

**Monthly non-discretionary spend** — sum recurring obligations the user can't easily cancel:

Include:
- Rent / mortgage
- Utilities (electric, gas, water, internet, phone)
- Insurance (auto, home, life, health premiums)
- Loan payments (auto, student, credit card minimums)
- Childcare / tuition
- Genuinely required subscriptions (work software, etc.)

Exclude:
- Entertainment subscriptions (Netflix, Spotify, streaming) — discretionary
- Gym, meal kits, hobby memberships — discretionary
- Variable categories (groceries, gas, dining) — these belong in the spending account

Note: many users already pay streaming/subs out of their bill-pay account because autopay is convenient. That's fine — don't treat it as a problem to fix. The tier 1 question is whether the hub is funded enough to cover everything that drafts from it, not whether each draft is "really" non-discretionary. Flag the discretionary share so the user can see it (often $200–400/mo of subs hide in plain sight), but only recommend moving them out if they specifically want stricter separation.

Convert frequencies to monthly: weekly × 4.33, biweekly × 2.17, monthly direct, annual / 12.

**Negative line items are a data flag, not a windfall.** If the `by group` query shows a category total below zero, it's net of refunds, returns, or a Plaid sign mismatch — there's an anomaly worth investigating, not a category that "earned money this period." Don't subtract it from the sweep computation. Mention it once so the user can correct the data, then exclude it from the math.

**Monthly take-home income** — read directly from the `paychecks` response: `totals.monthly_avg` is the sum of regular paycheck streams. If `irregular_income_90d` is non-zero, the user has variable comp (1099, freelance, bonuses) — say so plainly and treat the recurring number as a floor, not the whole picture.

**When `monthly_avg` is zero but `irregular_income_90d` is non-zero**, the recurring detector didn't catch the user's actual income. Common patterns: 1099 / freelance with varying amounts, native-corp or pension dividends, cash or check deposits, and gig platforms (DoorDash, Uber) paying out at variable cadences. Don't fabricate a sweep number against unknown income — instead, surface what you see (`"~$X/mo of inflow over 90d that isn't categorized as a paycheck"`) and ask the user to confirm before running the math. The same applies when `monthly_avg` is suspiciously small (a $0.07 "Interest Earned" stream is not a paycheck).

**Paycheck cadence** — already labeled on each paycheck (`weekly`, `biweekly`, `semimonthly`, `monthly`, `quarterly`, `semiannual`, `annual`). Convert to paychecks-per-month for the sweep math:
- weekly → 4.33
- biweekly → 2.17
- semimonthly → 2
- monthly → 1

**Per-paycheck sweep** — `monthly_non_discretionary / paychecks_per_month`, plus a buffer:
- Steady salary: 5–10% buffer
- Variable income: 15–20% buffer, plus suggest building a one-month float before relying on the system

Round to a clean number ($50 or $100 increments).

### Step 3: Detect what's already in place

From the flows query and the `acct` field on each recurring series, look for these patterns:

- **Single-hub:** one account receives transfers from a paycheck-account and is the source of most bill payments. That's a classic bill-pay hub.
- **Multi-source hub:** two or more earner accounts both transfer into one shared bill-pay account. Common in two-earner households. Name both contributors and their share.
- **Multi-envelope:** several spoke accounts (kid stipends, savings, sinking funds) draw from a hub or directly from a paycheck account. Map each spoke to its purpose.
- **No structure:** transfers are zero, ad-hoc, or only credit-card payments. They don't have an enveloping system yet.

Name each account by its function, not just its label. "Bill Pay" might literally be called "Bill Pay" or it might be called "Joint Checking" or "BoA Advantage 1234" — what matters is what it does.

When two earners contribute to one hub at different cadences (Adam biweekly, Mary semimonthly is a real example), inflow lands unevenly across months — some months get an extra Adam pull. Worth flagging if the hub balance ever runs thin, harmless otherwise.

### Step 4: Present the plan

Lead with the headline number. Two shapes depending on what you found:

**No system yet (most users):**
> Your monthly non-discretionary spend is about **$X**. You take home roughly **$Y** every two weeks, so to keep bills covered you'd move **$Z per paycheck** into a separate "bill pay" account. Whatever stays in your main checking is then yours to spend without thinking.
>
> You don't have a bill pay account today. The simplest setup: open a second checking account at your existing bank (same login, no new app), set up an automatic transfer the day after payday for $Z, then move every bill autopay to draft from the new account.

**System exists:**
> You're already running a bill pay setup — **\<Account Name\>** is the hub. \<Funded by X from \<source\>, Y from \<source\>\>. Recurring outflow is **$X/mo**, total inflow is **$Y/mo**, so the hub is \<over/under/right\>-funded by **$Z/mo** on the recurring view.
>
> [If sweep is too low: name the gap, suggest a new amount.]
> [If sweep looks high but the hub balance is still flat: non-recurring stuff (variable bills, anomalies) is consuming the apparent surplus — the system is matching real outflow even if recurring sums don't show it. Reconcile against the actual current balance before recommending a change.]
> [If amount is right: say so plainly. Don't manufacture work.]

**Fixed-income or very-low-income (single SSI/SSA stream, take-home under ~$2k/mo):**

Skip the percentage-of-take-home framing entirely — Ramit's 50/30/20 (or any similar split) doesn't apply when there isn't slack to allocate. Tier 1 mechanics (separating the bill draft from the spending account) are still useful as a discipline if the user wants the structure, but the more useful conversation first is what's quietly draining the cushion: typically credit-card interest, EWA fees, or a recurring line the user has stopped seeing. Surface that line by name, give the actual dollar impact, and let the user decide whether to engage. Only return to envelope mechanics if they ask.

A common mistake here is to recommend cutting the sweep based on `inflow − recurring_outflow > 0`, when in fact non-recurring charges (anomalies, variable utility bills, occasional larger insurance hits) are eating that surplus. The actual hub balance is the truth — if it's not accumulating despite the recurring math saying it should, the recurring view is incomplete.

To quantify the gap, run a follow-up:
- `query { "account": "<hub>", "is_recurring": false, "period": "last_90d", "by": ["category"] }` — non-recurring drafts hitting the hub, grouped by category. The total here usually explains the apparent surplus.
- Cross-check with `anomalies.by_account` from any summary query — gives a per-account total of large/unusual drafts in one number.

Always show a short breakdown of which recurring items you counted as non-discretionary so the user can correct you. Common disputes:
- Phone bill — usually non-discretionary, but ask if it's a family plan they could cut
- Gym, music streaming — borderline; let the user decide
- Insurance premiums — non-discretionary even if quarterly/annual

### Step 5: Offer tier 2 (only if asked or tier 1 is already humming)

Don't push the multi-envelope version on someone who hasn't gotten tier 1 going. Once they have, or if they ask:

> If you want to go further, the next layer is splitting the *non-bill* money:
> - **Savings envelope**: 5–10% of take-home, swept the same way as the bill account, untouched until you have a goal
> - **Sinking funds**: separate sub-accounts for predictable-but-irregular costs (vacation, gifts, car maintenance, holidays, annual insurance). Sweep a monthly amount, draw down when the bill arrives, never get blindsided.
> - **Investing**: out of scope for Cashflow, but Ramit's plan suggests routing 10% to long-term investing automatically once fixed costs and savings are running.

The biggest win by far is the first split. Five envelopes with leaks beat by one bill account that works.

## Enveloping-specific notes

- **Show the math.** People don't trust the sweep number unless they can see what went into it. Always include the per-line breakdown of non-discretionary items.
- **Don't recommend specific banks or account types.** Cashflow doesn't know what's available to the user. Say "second checking account at your existing bank" generically. The point is separation, not the brand.
- **Don't move money for the user.** Cashflow is read-mostly here. The output is "go set this up at your bank," not "I'll do it." Be clear about that.
- **Variable income changes the math.** Wider buffer (15–20%), and recommend building a one-month float in the bill account before depending on the sweep. Otherwise a slow week breaks rent.
- **Resist over-engineering.** If the user is anxious about bills, give them tier 1 and stop. Tier 2 is only useful when tier 1 is boring.
- **Reconcile against actual balances, not just the recurring sum.** Recurring detection misses irregular bills (variable utilities, quarterly anomalies, one-off larger draws). If the hub is receiving more than the recurring math says it spends but the balance is flat, the recurring view is incomplete — don't recommend a sweep cut based on the gap.
- **Two earners with different cadences is normal.** Biweekly + semimonthly into a shared hub gives uneven monthly inflows (some months get an extra biweekly pull). Note it if the hub balance ever runs thin; otherwise leave alone.
- **Reference Ramit lightly.** The Conscious Spending Plan is a useful framing if it fits the conversation, but most users just want the number to move per paycheck. Don't lecture.
- **Don't double-count income.** If a paycheck splits across two recurring inflows (employer + bonus, or two jobs), make sure you're summing actual dollars in, not duplicating.
- **Tone**: matter-of-fact, no judgement. People come to enveloping when something feels wrong; the job is to make the picture concrete and the next move obvious.
