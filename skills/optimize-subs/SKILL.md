---
name: optimize-subs
description: Triggered by "optimize subscriptions", "audit subscriptions", "subscription audit", "cancel subscriptions", "review subscriptions", "which subscriptions do I have", "subscription review", "how much am I spending on subscriptions", "find hidden subscriptions", "Apple subscriptions". Use this skill whenever the user wants to understand, review, or reduce their recurring subscription spending — even if they just mention "subscriptions" casually.
---

# Subscription Audit & Optimization

Discover all subscriptions — including ones hidden inside Apple's bundled billing — analyze the total cost, and surface savings opportunities.

## Why This Skill Exists

Subscription creep is real. People often don't realize how much they're spending on recurring charges because:
- **Apple bundles everything**: App Store subscriptions, movie rentals, audiobook purchases, and AppleCare all appear as a single "apple.com/bill" or "APPLE.COM/BILL" charge on the bank statement. A $75 charge might be a $39 WSJ subscription + five $4 movie rentals. The only way to break these apart is through Apple's receipt emails or the user's Apple Account purchase history.
- **Multiple Apple accounts per household**: Each family member may have their own Apple ID with their own subscriptions, all billing to the same card. Sophia's Canva and Strava subscriptions show up as "Apple" on Adam's credit card statement — completely invisible as individual line items.
- **Google Play does the same thing**: Android users see "GOOGLE*[service]" charges that can also bundle multiple subscriptions.
- Small monthly charges ($5-20) are easy to forget about
- Free trials silently convert to paid subscriptions

This skill cross-references transaction data with email receipts to build a complete picture. It works with or without email access — email just unlocks the Apple breakdown.

## Workflow

### Step 1: Gather subscription data from Cashflow

Call the `query` MCP tool with recurring mode:

```json
{ "recurring": true, "type": "expense" }
```

This returns all detected recurring expense series with fields: `name`, `party`, `freq` (frequency), `amt` (amount in dollars), `monthly` (normalized monthly cost), `last` (last seen date), `next` (next expected date), `active` (false if inactive, omitted if active), `acct` (account name).

Also fetch Apple-specific transactions to understand the bundled charges:

```json
{ "detail": true, "search": "apple", "period": "last_90d", "limit": 50 }
```

### Step 2: Search for Apple receipt emails (if email tools are available)

Check whether email search tools exist. Look for tools with names containing `gmail`, `mail`, `email`, or `outlook` — specifically a search/list function. Don't assume Gmail specifically; the tool could be from any email provider.

**If no email tools are available, skip to Step 3.** You can still produce a useful report from Cashflow data alone — you just won't be able to break apart Apple's bundled charges. Note this limitation clearly in the output.

If email search IS available, search for Apple subscription receipts from the last 6 months:

```
from:no_reply@email.apple.com subject:"Your receipt from Apple"
```

Read 10-15 recent receipts to build a picture. Also search for subscription-related emails from other services:

```
subject:("subscription renewal" OR "subscription confirmation" OR "recurring payment")
```

#### How Apple receipt emails work

Apple sends receipts from `no_reply@email.apple.com` with subject "Your receipt from Apple." — note the period at the end. There are two formats:

**Format 1 — Newer receipts (2026+)**:
```
Receipt March 28, 2026
Order ID: MM9NZB7JJ1
Document: 700112062489
Apple Account: amaryllis.messinger@gmail.com
Runna: Running Plans & Coach
Runna Premium (Monthly)
Renews April 24, 2026
SOPH!
$19.99
```

**Format 2 — Older/bundled receipts**:
```
Receipt
APPLE ACCOUNT adam@bitmechanic.com
BILLED TO Apple Card
DATE Jan 12, 2026
ORDER ID MSSWB4MXWX
App Store
The Wall Street Journal. News
WSJ Twelve Month Subscription (Monthly)
Renews Feb 12, 2026
Report a Problem
$38.99
Apple TV
Hey Good Lookin'
Comedy Movie Rental
Living Room TV
Report a Problem
$2.99
TOTAL $74.93
```

#### Key extraction rules

- **Subscriptions have "Renews [date]"** — this is the definitive marker. Items without "Renews" are one-time purchases (movie rentals, audiobook purchases, app purchases).
- **Apple Account email** identifies whose subscription it is. In a household, you'll see different Apple IDs (e.g., adam@bitmechanic.com vs amaryllis.messinger@gmail.com) — this tells you which family member owns the subscription.
- **App name appears on two lines**: the full app name (e.g., "Runna: Running Plans & Coach") then the subscription tier (e.g., "Runna Premium (Monthly)").
- **Dollar amount** follows each item, like `$19.99`.
- **Bundled receipts** contain multiple items under one order. A single receipt might have a WSJ subscription renewal + five movie rentals + an audiobook purchase, all charged as one "apple.com/bill" amount.
- **"Receipt & Renewal Notice"** in the header means it's a subscription renewal with an upcoming charge.
- **Apple Store receipts** (from `baystreet@email.apple.com` or similar store addresses) are physical hardware purchases — ignore these.

#### Also look for cancellation/expiry signals

Search for:
```
from:no_reply@email.apple.com subject:"subscription"
```

Apple sends separate emails for:
- "Your subscription is expiring" — user turned off auto-renew, it will lapse
- Price increase notices — "the price of [app] is increasing"

These are valuable signals for the audit — they reveal recently cancelled subscriptions and price changes.

### Step 3: Build the complete subscription inventory

Merge findings from all sources into a single list:

1. **Cashflow recurring series** — the primary source for most subscriptions
2. **Apple email receipts** — reveals individual subscriptions hidden inside "Apple" or "App Store" charges
3. **Other email receipts** — catches subscriptions that Cashflow may not have detected as recurring yet

For each subscription, normalize to:
- Service name
- Monthly cost (annualize yearly charges, etc.)
- Source (Cashflow recurring / Apple receipt / email)
- Last charged date
- Status (active / possibly cancelled / unknown)

### Step 4: Analyze and present findings

Present a clear, scannable report:

#### Summary
- Total monthly subscription spend
- Total annual projection
- Number of active subscriptions
- How much is hidden inside Apple billing

#### Full Subscription List

Organize by monthly cost (highest first). For each:
- Name, monthly cost, frequency, last charged
- Source: whether detected by Cashflow or found in email

#### Apple Breakdown

If Apple receipts were found, show a separate section:
- What Cashflow sees: "Apple" charge of $X
- What's actually inside: individual subscriptions with amounts
- One-time purchases (rentals, etc.) that inflated the charge

#### Potential Issues

Flag anything notable:
- **Duplicate or overlapping services** — e.g., paying for both Spotify and Apple Music, or ChatGPT Plus and Claude Pro
- **Forgotten/unused subscriptions** — charges still active but not recently used (especially old free-trial conversions)
- **Price increases** — if the recurring amount has changed (Cashflow's `stddev` field can hint at this)
- **Multiple Apple accounts** — receipts from different Apple IDs may mean family members have their own subscriptions

#### Recommendations

For each potential saving, note:
- The subscription name and monthly cost
- Why it's flagged (duplicate, unused, etc.)
- How to cancel: for App Store subscriptions, go to Settings > [your name] > Subscriptions on iPhone/iPad, or appleid.apple.com on the web

### When email is NOT available

If no email tools are connected, the report still works — just without the Apple breakdown. Present all Cashflow-detected subscriptions and analysis, but add a note like:

> I found [N] recurring charges in your bank data, including [M] charges to Apple. Apple bundles multiple subscriptions into single charges, so there may be additional subscriptions hidden inside. To see what's inside:
> - **iPhone/iPad**: Settings > [your name] > Subscriptions
> - **Web**: appleid.apple.com > Sign In > Subscriptions
> - **Or** connect your email so I can read Apple's receipt emails next time

Also note that without email, you can't detect:
- Subscriptions billed through Google Play (similar bundling)
- Subscriptions billed directly through apps that don't show up as distinct bank charges
- Price increases on Apple-billed subscriptions

## Optimize-subs-specific format

Keep the report scannable — use tables, not paragraphs. Lead with the summary numbers, then the details. If something looks like a duplicate, say so plainly. (General voice guidance lives in the system prompt.)
