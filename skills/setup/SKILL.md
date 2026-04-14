---
name: setup
description: One-time onboarding after connecting bank accounts. Maps household money flows, creates transfer rules with human-readable names, identifies paycheck and rent/mortgage, detects life stage and budgeting approach, flags unconnected accounts. Triggered by "set up my accounts", "just connected my bank", "first time setup", "onboarding", "get started", "configure cashflow", "now what". Use this skill whenever a user has just connected or asks to set up Cashflow for the first time.
---

# First-Time Setup

Walk a new user through initial configuration after connecting bank accounts. This one-time process makes everything else work — accurate recaps, clean spending reports, and correct transfer handling all depend on the foundations laid here.

## Why this matters

Raw bank data is messy. Transfers between a user's own accounts show up as expenses if not marked. Paychecks hide in generic descriptions. The same person appears under different names across accounts. This skill turns raw data into a clean, labeled financial picture.

## Step 1: Gather data and detect patterns

Pull everything needed to understand the user's financial topology:

```json
query { "health": true }
query { "include": ["accounts"], "period": "last_90d" }
query { "flows": true }
query { "recurring": true }
query { "by": ["group"], "type": "expense", "period": "last_90d", "top": 15 }
admin { "entity": "rule", "action": "list" }
```

**Check existing rules first.** If transfer rules, paycheck rules, or housing rules already exist, acknowledge them and focus only on gaps. Don't propose rules that duplicate existing ones.

From the flows query, identify the **archetype** — this determines the entire setup approach:

### Archetype: Family household

**Signal:** Transfers flow between accounts with different surnames in the description. Multiple checking accounts named for different people or with different masks. Possibly kid-named accounts with small regular inflows.

**Flow topology:** Chain or mesh — money moves between people (A → B → C), not just between account types.

**Example:** Adam's paycheck → Adam's checking → Mary's checking → Bill Pay → kid accounts. Descriptions contain surnames like "TRANSFER FROM POWELL", "TRANSFER TO MESSINGER".

### Archetype: Envelope budgeter

**Signal:** One person with many accounts (5+) at the same institution. Frequent round-number transfers from one hub account to several others. Star topology in flows.

**Flow topology:** Star — one hub distributes to many spokes. The hub receives the paycheck and funds each envelope.

**Example:** Hub checking receives $5k paycheck → $1.5k to "Bills" account, $800 to "Groceries", $500 to "Fun Money", $200 to "Emergency Fund". All accounts are the same person at the same bank.

### Archetype: Individual / simple

**Signal:** 1-3 accounts (checking, savings, credit card). Transfers are functional (checking → savings, checking → credit card payment). No multi-person flows.

**Flow topology:** Simple — checking → savings, checking → credit card. Maybe one external transfer (brokerage).

### Archetype: Couple, pooled finances

**Signal:** Two people's names appear on accounts but money pools into shared accounts. Fewer inter-person transfers than family household — more like two incomes into one pot.

**Draw the flow diagram.** Present the money topology as a simple ASCII diagram — this is the most valuable output of setup because it shows the user their financial plumbing at a glance:

```
Paycheck (Employer) → Main Checking → Savings
                                    → Credit Card payments
                                    → Bill Pay → utilities
                                    → Kid accounts
```

Present your read to the user:
> "Here's what I see: [X accounts at Y institution(s)]. [Diagram]. [Life stage guess]. Does that sound right?"

**Trust the data, not the user's self-description.** If the user says "I'm single" but the data shows transfers between accounts with different surnames and kid accounts, flag the discrepancy gently: "You mentioned being single, but I see accounts and transfers that suggest a household — want to clarify?" The data is ground truth; the user may have been imprecise.

Let the user correct before proceeding. Getting the archetype right determines which steps matter.

## Step 2: Name the accounts

Check for generic bank names ("EVERYDAY CHECKING ...2226", "TOTAL CHECKING ...4891"). Ask the user what they call each account.

```json
admin { "entity": "account", "action": "rename", "id": "<id>", "display_name": "<name>" }
```

**By archetype:**
- **Household:** Name by person — "Adam", "Mary", "Bill Pay", "Sophia"
- **Envelope:** Name by purpose — "Bills", "Groceries", "Fun Money", "Emergency"
- **Individual:** Name by type — "Main Checking", "Savings", "Visa"
- **Couple:** Could be by person or by purpose — ask

## Step 3: Create transfer rules

The flows query shows every inter-account transfer with amounts and account IDs. Use it to create rules that label transfers with human-readable names.

### Household transfer rules

Banks use surnames in transfer descriptions (e.g., "TRANSFER FROM POWELL"). Users want first names ("Transfer from Mary"). The rule architecture:

**Layer 1 — Catch-alls (short patterns, no account filter):** For each person, create rules matching just the surname. Use the SHORTEST pattern that uniquely identifies the flow — Plaid truncates pending transactions, so shorter is more resilient.

```json
admin { "entity": "rule", "action": "preview",
  "description_pattern": "TRANSFER FROM <LASTNAME>",
  "set_party_name": "Transfer from <FirstName>",
  "set_category_name": "Other Transfer",
  "set_is_transfer": true }
```

Both directions (FROM and TO) for each person. Typically 4 rules for a two-person household.

**Layer 2 — Account-specific overrides:** When a catch-all gives the wrong label because the same description appears on multiple accounts with different meanings. Common cases:
- Bill Pay funding: "Transfer to Bill Pay" on the source, "Transfer from Adam" on Bill Pay
- Kid stipends: "Transfer from Bill Pay" on the kid's account, "Transfer to Sophia" on Bill Pay
- Use `account_id` from the flows query — it provides both `from_acct_id` and `to_acct_id`

### Envelope budgeter transfer rules

Transfers are the budgeting system — label them by the envelope's purpose, not by person.

**Identify the hub** — the account that receives the paycheck and funds the envelopes. It has the most outgoing transfers in the flows.

**For each spoke:** Create rules labeling the funding transfer. Descriptions might look like "CHASE TRANSFER TO CHECKING ...4891" or "TRANSFER BETWEEN ACCOUNTS REF#..." — the patterns vary by bank. Look at actual descriptions from the flows data to determine the right patterns.

```json
admin { "entity": "rule", "action": "preview",
  "description_pattern": "<pattern matching this envelope's transfers>",
  "account_id": "<envelope-account-id>",
  "set_party_name": "Fund from Hub",
  "set_category_name": "Other Transfer",
  "set_is_transfer": true }
```

The hub outbound side:
```json
admin { "entity": "rule", "action": "preview",
  "description_pattern": "<pattern>",
  "account_id": "<hub-account-id>",
  "set_party_name": "Fund <Envelope Name>",
  "set_category_name": "Other Transfer",
  "set_is_transfer": true }
```

### Individual transfer rules

Often minimal — Plaid usually handles checking → savings and credit card payments correctly. Verify by checking the flows query: if transfers already show as `xfer: 1` with reasonable names, no rules needed.

If not, create simple rules for:
- Credit card payments ("Transfer to [Card Name]")
- Savings transfers ("Transfer to Savings")

### Rule hygiene (all archetypes)

1. **Always set both** `set_is_transfer: true` AND `set_category_name: "Other Transfer"`. Plaid sometimes misclassifies transfers as general services; without a category the transaction appears "uncategorized" in reports.
2. **Always preview before creating.** Check match count and scan for false positives.
3. **Present previews in batches** — don't ask for confirmation one rule at a time.

## Step 4: Verify credit card payments

Check whether credit card payments are correctly marked as transfers. Look at the flows query for any flow TO a credit card account. Common patterns:
- "AUTOPAY" or "PAYMENT" in the description
- Plaid PFC `LOAN_PAYMENTS_CREDIT_CARD_PAYMENT` usually handles this automatically

If credit card payments show up as expenses instead of transfers, create a rule:
```json
admin { "entity": "rule", "action": "preview",
  "description_pattern": "<CARD NAME> PAYMENT",
  "set_party_name": "<Card Name> Payment",
  "set_category_name": "Credit Card Payment",
  "set_is_transfer": true }
```

Also check for Apple Card payments (Goldman Sachs) — these often have unusual descriptions like "APPLECARD GSBANK PAYMENT" and Plaid sometimes misclassifies them.

## Step 5: Identify the paycheck (if working)

Skip for retirees (look for pension/Social Security instead) and students (may have irregular income).

```json
query { "recurring": true, "type": "income" }
```

The paycheck is the largest, most regular income — same amount, same frequency, same party. For households, identify whose account it lands in — this is usually the top of the money flow chain.

Confirm with the user and create a rule if the party name is messy or category is wrong.

## Step 6: Identify rent/mortgage

Skip if not visible in the data (student living at home, housing paid by other means).

```json
query { "recurring": true, "group": "Housing" }
```

If nothing in Housing, check top recurring expenses — might be miscategorized. Confirm and fix.

## Step 7: Flag unconnected accounts

Look for references to accounts not in the system:
- Investment: "SCHWAB", "FIDELITY", "VANGUARD", "AMERITRADE" in transfer descriptions
- Credit cards: payment descriptions mentioning issuers not connected
- Loans: mortgage servicer, student loan, car payment going to unknown entities

Don't push — mention what you found and let the user decide:
> "I see transfers to [institution] that aren't connected. Want to add it for a more complete picture?"

## Step 8: Summary and next steps

Present what was configured:

```
Setup complete:
  Accounts named: X
  Transfer rules created: Y (covering Z transactions)
  Paycheck: $A/mo from [employer]
  Housing: $B/mo to [payee]
  Unconnected accounts flagged: [list or "none"]
```

Suggest next steps:
- **"/tidy"** — clean up remaining uncategorized transactions
- **"/recap this month"** — first financial summary
- **"/burn"** — see recurring monthly obligations

## Re-running setup

If run on an already-configured account, check existing rules and account names first. Focus on what's missing or broken rather than redoing everything. Acknowledge what's already in place.

## Tone

Be welcoming and patient — this is the user's first interaction with Cashflow. Explain what you're doing and why. Don't overwhelm — batch questions logically (what are we looking at → how does money flow → what should things be called → what else should we look for). If something looks clean already, skip it.
