---
name: setup
description: One-time onboarding after connecting bank accounts. Pulls a wide read in parallel, surfaces what stands out, and asks the user what to do next. Read-only on first turn — applies changes only when the user confirms. Triggered by "set up my accounts", "just connected my bank", "first time setup", "onboarding", "get started", "configure cashflow", "now what", or by the system immediately after a successful Plaid link.
---

# First-Time Setup

Look at a user's freshly-connected accounts and tell them what you see. Run once after Plaid link. **Read-only on the first turn** — change nothing until the user says go.

## Why this matters

Raw bank data is messy. Money the user moved between their own accounts can look like spending. Paychecks hide behind generic descriptions. The same store shows up under three different names. The job is to make the picture cleaner — but the user knows their own money better than you do, so the first pass is just looking, not fixing.

## Step 1: Pull everything in parallel

Issue all six in a **single tool-use turn (parallel calls)**:

- `query { "health": true }`
- `query { "include": ["accounts"], "period": "last_90d" }`
- `query { "flows": true }`
- `query { "recurring": true }`
- `query { "by": ["group"], "type": "expense", "period": "last_90d", "top": 15 }`
- `admin { "entity": "rule", "action": "list" }`

If existing rules already cover the obvious gaps, focus the rest of the pass on what's still rough.

## Step 2: Reflect back what you see (one message)

Write a single message organized as three short sections, in this order:

1. **Accounts**
2. **Recurring**
3. **Worth a closer look**

Use these as section headings (`## Accounts`, etc.). Within each section, write flowing prose — paragraphs, not bulleted findings, no bolded inline labels. Aim for ~200–400 words total. Skip a section entirely if it has nothing worth saying. Each section should read like an observation, not a verdict.

### Section 1 — Accounts

Cover, in plain prose:

- How many accounts are connected, at which banks, which look active and which sit at zero. Names that are unclear ("CREDIT CARD", raw bank codes) are worth asking about — *what* is this account, not guessing what it is. Don't pester about names that are already clear ("Family Checking", "Joint Savings").
- Where money comes in and how it spreads out — does it land in one account and move to others, are there regular transfers between two specific accounts, etc.
- Banks or cards mentioned in your transfers, bill payments, or recurring charges that aren't connected here (Apple Card, Capital One, Fidelity, Schwab, Robinhood, the mortgage company, the student loan servicer). Mention as something they might want to add, not as a problem.

### Section 2 — Recurring

Cover the regular money in and out:

- Anything that looks like a paycheck — name the employer and the rough amount, then check the framing fits ("looks like a Trucker Huss paycheck twice a month — is that the right way to think about it?"). If the money coming in is irregular (one-off deposits, transfers in, advances), say so without trying to call it a paycheck.
- Anything that looks like rent or mortgage — the biggest regular Housing charge, or a large regular outflow that looks housing-shaped but isn't categorized. Flag it if Housing is empty.
- Big or notable regular bills (insurance, utilities, child support, loan payments) only if there's something worth saying — a recent price jump, something new, something unusually high. Don't list every bill; that's what `/burnrate` is for.

### Section 3 — Worth a closer look

Cover what needs the user's eyes. Pick the 2–4 most important items; don't bury them in twelve.

- **Rules worth adding.** Rules can mark transfers, set a category, give one name to a merchant that shows up under several (e.g. consolidating `AMZN MKTP US*1A2B3` and `AMAZON.COM*ABC` as "Amazon"), tag, or ignore. Roughly in this priority:
  - Money moving between the user's own accounts that isn't yet marked as a transfer (this throws everything else off)
  - One real merchant showing up under several different descriptions
  - A merchant the user buys from often where the transactions aren't sorted into a category yet (10+ in 90 days)
  - Obvious noise worth ignoring (micro-charges, placeholder rows)
- **Things that haven't been sorted yet** — large totals, the biggest single source of unsorted transactions.
- **Oddities** — anything from the health check that needs a human read: deposits that look like income but might not be, unusually large transactions, a recurring charge that suddenly jumped in price. Phrase as questions: "There's a $5K transfer from \<name\> on April 9 — was that a one-off?"

### Setup-specific tone notes

(General voice guidance — plain English, no jargon, no archetypes, hedge guesses — lives in the system prompt.)

- The three section headings (above) are the only structural signposts. No bolded inline labels inside sections — write prose.
- No money-flow diagrams unless the structure is genuinely complex (5+ interlinked accounts) AND a diagram makes it clearer than words would. Default: skip.
- Don't suggest rules for things Plaid usually gets right (the standard credit-card payment, the standard checking-to-savings transfer, well-known stores already in the right category).
- Don't restate dollar amounts the user can already see in the recurring list.

### Closing

End with an open question. Something like: *"Where would you like to start? I can [the one or two most useful things you raised], or you can poke around — `/recap` for a monthly summary, `/burnrate` for the regular bills picture, `/tidy` to clean up unsorted transactions."*

If the user replies "go", "yes", "apply", or names a specific part, move on to Step 3.

## Step 3: Apply (only when the user confirms)

For whatever subset they confirmed:

1. Issue all `admin rule preview` calls in a **single parallel batch**. Show the previews together.
2. If match counts look sane, call `admin rule create` for the batch in **one parallel turn**.
3. Run `annotate { "action": "apply_rules", "rule_ids": [...] }` once with all new rule IDs.
4. Issue all `admin account rename` calls in a **single parallel batch**.

Never preview-then-create one at a time. If a preview returns a count that's wildly different from what you described in Step 2, stop and tell the user before creating anything.

## Transfer rule patterns

### Multi-person household

Two layers:

**Layer 1 — Catch-alls (no account filter):** For each person, the SHORTEST pattern that uniquely identifies the flow. Plaid truncates pending transactions, so shorter is more resilient.

```json
admin { "entity": "rule", "action": "preview",
  "description_pattern": "TRANSFER FROM <LASTNAME>",
  "set_party_name": "Transfer from <FirstName>",
  "set_category_name": "Other Transfer",
  "set_is_transfer": true }
```

Both directions for each person — typically 4 rules for a two-person household.

**Layer 2 — Account-specific overrides:** When the same description means different things on different accounts. Common cases:
- Bill Pay funding: "Transfer to Bill Pay" on the source, "Transfer from Adam" on Bill Pay
- Kid stipends: "Transfer from Bill Pay" on the kid's account, "Transfer to Sophia" on Bill Pay

Use `from_acct_id` / `to_acct_id` from the flows query.

### Hub-and-spoke (envelope-style)

Identify the hub (most outgoing transfers in flows). Label transfers by purpose.

- Spoke side: rule with `account_id` pinned to the envelope, party "Fund from Hub".
- Hub side: rule with `account_id` pinned to the hub, party "Fund \<Envelope\>".

Patterns vary by bank — read actual descriptions from the flows data.

### Simple (1–3 accounts)

Often nothing to do. Plaid usually handles checking → savings and credit card payments correctly. Only propose rules where flows shows misclassified transfers.

### Hygiene

- **Transfer rules** must set both `set_is_transfer: true` AND `set_category_name: "Other Transfer"`. Without a category the transaction reads as uncategorized after the rule fires.
- **Party-consolidation rules** (different descriptions → one party) usually want `set_party_name` only, no category change. Let the existing category stand unless it's clearly wrong.
- **Category rules** with no party change are fine; just `description_pattern` + `set_category_name`.

## Re-running

If rules, account names, and transfer labels already look clean, say so in one or two sentences and stop. Don't manufacture work.
