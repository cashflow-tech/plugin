---
name: setup
description: One-time onboarding after connecting bank accounts. Detects household archetype, drafts transfer rules, identifies paycheck and rent/mortgage, flags unconnected accounts. Read-only first — applies changes only when the user confirms. Triggered by "set up my accounts", "just connected my bank", "first time setup", "onboarding", "get started", "configure cashflow", "now what", or by the system immediately after a successful Plaid link.
---

# First-Time Setup

Map a freshly-connected user's financial topology and propose a small batch of fixes. Designed to run once after Plaid link. **Read-only on the first turn** — apply changes only when the user explicitly confirms.

## Why this matters

Raw bank data is messy. Transfers between a user's own accounts show up as expenses if not marked. Paychecks hide in generic descriptions. The same person appears under different names. This skill turns raw data into a clean, labeled financial picture.

## Step 1: Pull everything in parallel

Issue all six of these queries in a **single tool-use turn (parallel calls)**. Do not run them one at a time.

- `query { "health": true }`
- `query { "include": ["accounts"], "period": "last_90d" }`
- `query { "flows": true }`
- `query { "recurring": true }`
- `query { "by": ["group"], "type": "expense", "period": "last_90d", "top": 15 }`
- `admin { "entity": "rule", "action": "list" }`

If existing rules already cover transfers, paychecks, or housing, treat those as done and focus only on gaps.

## Step 2: Infer and report (one message)

From the data above, in **one message** back to the user, present:

### Archetype

Pick one from the flow topology:

- **Family household** — transfers between accounts with different surnames; multiple checking accounts named for different people; small recurring inflows to kid-named accounts.
- **Envelope budgeter** — one person, 5+ accounts at the same institution; round-number transfers from a hub account to spokes; star topology in flows.
- **Individual / simple** — 1–3 accounts (checking, savings, credit card); functional transfers only.
- **Couple, pooled finances** — two people on accounts but money pools into shared accounts; few inter-person transfers.

State the inference and the signals you used. Don't ask for confirmation — if it's wrong, the user will correct it.

### Money flow diagram

ASCII diagram of the topology. This is the most valuable thing you produce — it shows the user their plumbing at a glance.

```
Paycheck (ACME) → Adam checking → Mary checking → Bill Pay → kid accounts
                               → Savings
                               → Visa payment
```

### Recommendations (do not apply yet)

Present as a numbered list. Describe each in prose, with the pattern and approximate match count from the flows/recurring data already in hand. **Do not call `admin rule preview` per rule** — that comes in Step 3.

1. **Account renames** — list any accounts with generic bank names (e.g. "EVERYDAY CHECKING ...2226"). Suggest a name based on archetype:
   - Household: by person ("Adam", "Mary", "Bill Pay", "Sophia")
   - Envelope: by purpose ("Bills", "Groceries", "Fun Money")
   - Individual/Couple: by type ("Main Checking", "Savings", "Visa")
2. **Transfer rules** — see *Transfer rule patterns* below. e.g. "Add a rule for `TRANSFER FROM POWELL` → 'Transfer from Mary'. Would match ~14 transactions in last 90d."
3. **Paycheck** — largest, most regular income from `recurring`. Skip for retirees (look for pension/Social Security) and students (irregular income). State the inferred employer and amount.
4. **Rent/mortgage** — top recurring in the Housing group. If Housing is empty but a large recurring expense looks housing-shaped, flag it.
5. **Unconnected accounts** — institutions referenced in transfers/recurring that aren't connected (Schwab, Fidelity, Vanguard, Ameritrade, mortgage servicers, student loan servicers, unfamiliar credit cards).

Skip any branch with no signal. Empty `flows` → no transfer rules. Empty `recurring.income` → no paycheck section. Don't generate zero-match recommendations.

### Closing prompt

End with: *"Cashflow is set up and ready. You can apply the rules I drafted (reply 'go'), or jump straight in: try `/tidy` to clean up uncategorized transactions, `/recap` for a monthly summary, or `/burn` to see your recurring obligations."*

## Step 3: Apply (only when the user confirms)

On confirmation ("go", "yes", "apply", or a subset like "rules only"):

1. Issue all `admin rule preview` calls in a **single parallel batch**. Show the previews together.
2. If match counts look sane, call `admin rule create` for the batch in **one parallel turn**.
3. Run `annotate { "action": "apply_rules", "rule_ids": [...] }` once with all new rule IDs.
4. Issue all `admin account rename` calls in a **single parallel batch**.

Never preview-then-create one rule at a time.

## Transfer rule patterns

### Family household

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

### Envelope budgeter

Identify the hub (most outgoing transfers in flows). Label transfers by envelope purpose.

- Spoke side: rule with `account_id` pinned to the envelope, party "Fund from Hub".
- Hub side: rule with `account_id` pinned to the hub, party "Fund \<Envelope\>".

Patterns vary by bank — read actual descriptions from the flows data.

### Individual

Often minimal. Plaid usually handles checking → savings and credit card payments correctly. Only propose rules where flows shows misclassified transfers.

### Hygiene (all archetypes)

- Always set `set_is_transfer: true` AND `set_category_name: "Other Transfer"`. Without a category, the transaction shows as uncategorized.

## Re-running

If rules, account names, and transfer labels already look clean, say so and stop. Don't redo finished work.

## Tone

Welcoming but compact. Don't over-explain. The archetype, the diagram, and the recommendation list are the product — everything else is scaffolding. If something already looks clean, skip it without ceremony.
