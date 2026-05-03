---
name: drift
description: Compare the last 12 months to the 12 before them and show whether expenses grew faster than income — by how much, and in which categories. Triggered by "lifestyle inflation", "lifestyle creep", "am I drifting", "spending creep", "did the raise help", "where did my raise go", "am I keeping pace", "treading water", "expenses up year over year", "is my spending growing", "am I getting ahead", and the slash command `/drift`. Use this skill whenever the user asks about year-over-year change in spending, especially in relation to income — including casual phrasings like "did the raise actually help?" or "am I getting ahead?"
---

# Lifestyle Drift

Compare the last 12 months to the 12 before them. Show whether expenses outgrew income, and where.

## Why this matters

When income rises and spending rises by about the same amount, the user isn't getting richer — they're treading water. The system prompt's diagnose section calls this out directly: *"when income rose and savings rate didn't, say so."* This skill is how you say so. It puts a number on the gap and names the categories that drove it. It does not tell the user what to cut. Money writers disagree on that part — and the user knows their life better than you do.

## How this fits with other skills

- `recap` answers "how did this month/quarter go" against the period right before it.
- `burnrate` shows the fixed-cost floor, right now.
- `leaks` audits forgotten subscriptions, fees, and predatory products.
- `drift` is the year-over-year view: did the trend line tilt up faster than your income did?

If the user asks "where am I leaking money," that's `leaks`. If they ask "what do I pay every month," that's `burnrate`. If they ask "did my raise actually help," "am I getting ahead," or "is my spending creeping up" — that's this skill.

## Arguments

`$ARGUMENTS` can specify a custom comparison:

- (no argument) → trailing 12 months vs the 12 months before that (default)
- a year ("2025", "drift in 2025") → that calendar year vs the prior calendar year
- "last 6 months", "Q1", "this year so far" → parse the period and compare against the same window a year earlier

Default to trailing-12 vs prior-12. Same-period-last-year is the canonical compare. Last-month-vs-this-month doesn't catch seasonality (heating, holiday, vacation, taxes, insurance premiums).

## Workflow

### Step 1: Pull totals with year-over-year compare

```json
{ "period": "trailing_12m", "compare": "prior_year", "include": ["ratios"] }
```

This returns income, expenses, net, and a `vs_prior: {abs, pct}` block on each. Also returns `ratios.savings_rate` for the trailing-12 window.

Bow out if any of these hold:

- `vs_prior` is missing or empty (no prior-year window).
- `vs_prior.income.prior < 5000` OR `vs_prior.expenses.prior < 5000` — the prior year is degenerate. A $54 expense baseline turns any percent change into noise (a normal year would explode to 4,000%+).

In either case, tell the user how thin the history is and bow out: "I need at least a year of real activity in both windows. Come back when there's more." Don't compute a drift gap on a baseline that small — the percentages will mislead.

### Step 2: Pull per-group expense changes

```json
{ "period": "trailing_12m", "compare": "prior_year", "by": ["group"], "type": "expense", "top": 30 }
```

Each group result has its own `vs_prior: {abs, pct}` showing the year-over-year change.

Before going further, scan the response for two data-shape problems that make drift math unreliable:

- **Single-group dominance.** If one group is ≥90% of total expenses AND it's `Credit Payments`, `Cash & Checks`, or `Uncategorized`, the underlying spend isn't classified yet. Bow out: "Almost all your expenses landed in one bucket I can't see inside. Run /tidy first to sort categories, then come back." Drift across undifferentiated data is meaningless.
- **Sign-flipped groups.** If any group's `amt` is negative in expense mode (refunds outpacing real spending in that category), surface it as a data warning: "Transportation came back as -$X — refunds outran spending. The drift gap inherits that distortion." The gap can still be useful but the reader needs to know.

### Step 3: Compute the drift gap

- **drift gap** (in percentage points) = expense growth percent − income growth percent
- Positive = expenses outgrew income (drift)
- Near zero = pacing
- Negative = income outgrew expenses (savings rate climbing)

### Step 4: Triage what kind of story this is

The headline depends on the shape:

- **Income dropped >10%** — drift isn't the right frame. Acknowledge: "Income is down X% from last year. Let's talk runway, not drift." Suggest `recap` or `burnrate`. Stop here.
- **Drift gap < −3 pp** — the user moved the other way *on the gap*. Before declaring this a win, check `ratios.savings_rate` and `net.vs_prior.pct`. If savings rate is still negative, or net dropped year over year, the negative gap is misleading — they're moving slower toward the cliff but still moving toward it. Lead with the savings-rate problem first, *then* show the picture and call out where the apparent gain came from. Often one big change: paid off a car, downsized housing, sold a business, one-time event ended.
- **|drift gap| ≤ 3 pp** — pacing. Say so plainly. "You're keeping up with income." Then show the categories that moved most regardless, in case anything is surprising.
- **Drift gap > +3 pp** — the headline drift case. Lead with the number: "Expenses up X%. Income up Y%. Gap: Z percentage points."

### Step 5: Identify the drivers

From the per-group data, surface the 3–5 groups with the biggest year-over-year change. A group is worth surfacing if either:

- Absolute year-over-year change > $500/yr, or
- Percent change > 2× the income growth rate (catches small-but-fast-growing categories before they're large)

For each surfaced group, show: name, prior-year amount, current amount, dollar change, percent change. One-line table rows. Sort by absolute dollar change descending — that's almost always what the user cares about.

Skip groups with small change in both percent and dollars. Don't pad the table.

### Step 6: Flag one-time events and collapses

Two patterns to watch:

- **Jumps from near zero to a material number** — wedding, move, baby, medical event, one-time tax bill, vacation. Name it; let the user confirm rather than treating the spike as ongoing. (Note: a "jump from near zero" can also be a *new ongoing line item* — a subscription that just started, a rental that just began. If the new spending matches a recurring series, lean toward "ongoing" not "one-time.")
- **Collapses** — a single group's year-over-year drop is more than 80% AND that group was >10% of prior expenses (housing, transport, food, education, giving). A 90% drop in housing usually isn't real saving — it's a move, a paid-off mortgage, or a sync gap. **When you spot a collapse like this, also compute an "ex-this-group" drift gap** and surface it alongside the primary number so the structural picture isn't hidden behind the collapse. Example: *"Drift gap is −5 pp overall, but Transportation collapsed −90% (likely a sold car). Excluding Transportation, the rest of the budget is +8 pp — still creeping up."*

### Step 7: Translate the gap into monthly dollars

`(current_year_expenses − prior_year_expenses) ÷ 12` is the monthly drag in plain dollars. State it. If the increase is concentrated in groups that look ongoing (subscriptions, dining out, transport, housing), say so — the monthly drag is structural, not a blip.

### Step 8: Check the savings-rate signal (if relevant)

If the user got a raise (income up materially) but `ratios.savings_rate` is roughly flat or down vs prior periods, name it. This is the system prompt's exact language: *"income rose and savings rate didn't."* The drift gap captures it indirectly; the savings rate states it directly.

### Step 9: Cash-heavy household caveat

If the `Cash & Checks` group is more than 25% of total expenses (ATM withdrawals + peer payments dominating), prepend a one-line caveat at the top of the output: *"X% of your spend left as cash; what's underneath the ledger can't see."* The drift may still be directionally right, but the user should know the picture has gaps. The threshold isn't arbitrary: at 25%+, a single shift in cash behavior (using a card more, less, or differently this year vs last) can swing the drift gap by more than the structural drift itself.

## Output structure

Order matters. Caveats and data warnings come *before* the headline so the reader knows how to weight what comes next.

1. **Caveats first.** Cash-heavy share if >25%; sign-flipped expense groups; sync-incomplete warning if loud. One line each.
2. **Headline.** Income up X%, expenses up Y%, gap is Z percentage points. Or the appropriate triage variant (income drop, pacing, savings-rate-still-underwater override).
3. **Ex-collapse gap** if a one-time category collapse distorted the primary number.
4. **Short table.** 3–5 biggest movers — group, prior, current, dollar change, percent change.
5. **One-time event callouts.** Wedding, move, baby, paid-off debt, etc.
6. **Monthly-drag line** if drift is positive: *"$X/mo of new ongoing spending."*
7. **Savings-rate signal** if the raise-without-savings-bump pattern fired.

Don't recommend cuts. Don't pick categories the user should "trim." If they want that, hand off to `leaks` or `burnrate`.

## Drift-specific notes

- **Materiality threshold:** ±3 percentage points on the drift gap is the practical signal. Smaller is noise.
- **Inflation context:** general inflation runs ~3% per year. A 4% expense bump on flat income is real but small; an 18% bump is structural. Don't over-call drift in normal-inflation territory.
- **Raise misallocation is the canonical story.** If income rose materially and discretionary groups absorbed most of it, that's the drift the user came in suspecting. Name it in plain dollars: "Your raise added $X/mo. Restaurants and travel added $Y/mo. Net to savings: $Z."
- **A flat savings rate with rising income is itself drift**, even if total expense growth roughly matches total income growth — the new income should be bumping the savings rate, and it isn't. Surface this directly when it shows up.
- **Don't moralize.** Show the dollars. Where money writers disagree — Sethi says cut what you don't love, FIRE says cut everything, Ramsey says scrap the lattes — present the tradeoff only if asked. (General voice guidance lives in the system prompt.)
