---
name: benefits
description: Screen for unclaimed safety-net benefits (EITC, SNAP, LIHEAP, WIC, Medicaid) using the user's Cashflow data, then generate a prep packet for a free Community Action Agency appointment. Triggered by "am I missing any benefits", "do I qualify for SNAP", "food stamps", "EITC", "help with bills", "help with rent", "I can't afford", "money is tight", "community action", "211", "free tax prep", "VITA". Use this skill whenever the user mentions financial hardship, asks about any government assistance program, mentions struggling to make ends meet, OR — when invoked from a recap/check-in — when their data shows likely eligibility (income under ~200% FPL, heavy debt service, utility-bill stress). Don't wait for the user to know the right name for the program — surface it when the data calls for it.
---

# Benefits Screening + Community Action Prep Packet

Help a user figure out what safety-net programs they likely qualify for, and prepare them to walk into a free Community Action Agency appointment with everything they need.

## Why this matters

Roughly one in four Americans eligible for federal benefits doesn't claim them — that's thousands of dollars per household per year left on the table. The biggest barriers are knowing what exists, knowing if you qualify, and getting through complicated applications. **Cashflow can see whether someone's income makes them likely-eligible. A free Community Action Agency visit can enroll them in everything they qualify for in one sitting. The gap between the two is just preparation.**

Not every user benefits from this. A $150k household has different problems. The skill should diagnose first — if the user is clearly out of the income range for these programs, say so and gracefully exit (offering tax-prep tips or other targeted help if relevant).

## Step 1: Pull a financial snapshot

Get the last 12 months of activity:

```json
query { "period": "trailing_12m" }
query { "by": ["category"], "period": "trailing_12m", "top": 30 }
```

From the response, capture:
- **Annual income** — the `income` field (this includes Wage Advance, Other Income, Salary, Refunds, Interest)
- **Salary** vs **other inflows** — separate W-2 wages from EWA, benefits, gigs, deposits
- **Annual debt service** — sum of `Fees & Charges/Interest Charges` + `Credit Payments/BNPL & Installments` + DTC payday-loan repayments visible in any category
- **Utility spend** — `Bills & Utilities/Electric`, `/Gas`, `/Water` (LIHEAP relevance scales with this)
- **Housing spend** — `Housing/Rent` + utilities (rent-burden calculation)

If annual income is **above $60,000**, this skill is almost certainly not the right tool. Tell the user: "Based on your income, the safety-net programs I screen for probably don't apply — they're income-tested. If you're looking for help with something specific, tell me what's going on and I can point you in a better direction."

If income is between $60k and $80k, screen anyway but flag clearly that eligibility is borderline.

## Step 2: Conversational interview

There are six things you need to ask about that aren't visible in transactions. Ask them naturally, not as a survey. Combine related ones. **Explain why you're asking** — these questions feel intrusive without context.

**The six:**

1. **Household size and dependents.** Who lives with you, and how old are any kids? (Programs are sized by household; kids under 5 unlock WIC; school-age kids unlock free meals + Head Start eligibility.)

2. **ZIP code.** Programs vary state-by-state and county-by-county. Need this to find the local Community Action Agency.

3. **Citizenship/status.** U.S. citizen, permanent resident, or other status? (Some programs require citizenship or specific lawful presence categories. Don't ask intrusively — just enough to know what to screen.)

4. **Pregnancy or kids under 5.** Anyone in the household pregnant or postpartum? Kids under 5? (WIC eligibility.)

5. **Disability or veteran status.** Anyone disabled, on SSDI/SSI, or a veteran? (SSDI/SSI eligibility check, VA programs.)

6. **Already enrolled?** Already getting any of these — SNAP / EBT, Medicaid, LIHEAP, WIC, SSI, Section 8 housing? (Avoid recommending things they're already on.)

7. *(If income range suggests EITC eligibility):* **Tax filing.** Did you file taxes for last year? If not, that's worth knowing — EITC requires filing.

Be conversational. **Bad:** "Question 1: household size?" **Good:** "To figure out what you might qualify for, I need a few quick details. First — who lives in your household? Just you, or are there kids or a partner?"

## Step 3: Eligibility screening

For each program, calculate likely eligibility from income, household size, and the answers above. The Federal Poverty Level (FPL) table is the foundation:

### 2026 Federal Poverty Level (annual income)

| Household size | 100% FPL | 138% FPL (Medicaid) | 150% FPL (EITC w/o kids guideline) | 185% FPL (WIC) | 200% FPL (most CAA programs) |
|---|---|---|---|---|---|
| 1 | $15,650 | $21,597 | $23,475 | $28,953 | $31,300 |
| 2 | $21,150 | $29,187 | $31,725 | $39,128 | $42,300 |
| 3 | $26,650 | $36,777 | $39,975 | $49,303 | $53,300 |
| 4 | $32,150 | $44,367 | $48,225 | $59,478 | $64,300 |
| 5 | $37,650 | $51,957 | $56,475 | $69,653 | $75,300 |
| 6 | $43,150 | $59,547 | $64,725 | $79,828 | $86,300 |
| 7 | $48,650 | $67,137 | $72,975 | $90,003 | $97,300 |
| 8 | $54,150 | $74,727 | $81,225 | $100,178 | $108,300 |

For each additional person beyond 8, add $5,500.

Compute the **FPL ratio**: annual income ÷ (100% FPL for household size). E.g. a single adult earning $20,000 has an FPL ratio of $20,000/$15,650 = **128% FPL**.

### Per-program eligibility rules

Detailed program-by-program rules are in `references/programs.md`. Read it when you're ready to do the screening — it covers SNAP, EITC, LIHEAP, WIC, Medicaid, and supporting programs (free/reduced school meals, Lifeline phone, weatherization, ACP internet, EITC state add-ons, child care subsidy).

For each program, output:
- **Status**: "Likely qualifies" / "May qualify — depends on state details" / "Doesn't qualify based on income" / "Already enrolled"
- **Estimated annual value** — concrete dollars where possible (EITC has lookup tables; SNAP averages $185/mo per household member; LIHEAP $300-$1,000/yr depending on state)
- **One-sentence why** — "Your income at $X is under the X% FPL threshold for household size Y"

**Be honest about uncertainty.** Use "likely qualifies" rather than "qualifies" — only certified counselors and the agencies themselves can make a final determination. The skill's job is surfacing possibilities, not certifying enrollment.

## Step 4: Resolve the local Community Action Agency

For v1, we don't have a comprehensive CAA directory embedded. Use this fallback:

1. **2-1-1** is universal — every state has a 211 line that routes to local services. Always recommend it.

2. **Community Action Partnership locator** — point user to https://communityactionpartnership.org/find-a-cap/ (their public locator covers ~99% of US counties).

3. If the user is in one of the major metros below, you can give a starting suggestion, but always tell them to confirm via the locator since CAA service areas don't perfectly map to metros:

| Region | Possible starting point |
|---|---|
| New York City | Community Service Society of NY, or call 311 |
| Los Angeles County | Community Action Partnership of Orange County or LA |
| Chicago | Community and Economic Development Association |
| Bay Area | Community Action of Napa Valley, Bay Area Community Services |
| Seattle | Solid Ground or YWCA Seattle / King County |
| DC area | Alexandria Office of Community Services, DC Department of Human Services |

Otherwise: "Call 2-1-1 or visit communityactionpartnership.org/find-a-cap/ and search by your ZIP."

## Step 5: Generate the prep packet

This is the highest-value output. Generate as Markdown, in the chat. Tell the user they can copy/save/print it and bring it to their appointment.

Use this exact structure:

```markdown
# Benefits Screening & Community Action Prep Packet
*Generated by Cashflow on [today's date]*

## Your Income (last 12 months)

| Source | Annual | Monthly Avg | Frequency |
|---|---|---|---|
| [Salary / employer name] | $X | $Y | [Bi-weekly / monthly / variable] |
| [Wage Advance / Tapcheck etc.] | $X | $Y | Variable |
| [Other Income / SSA / etc.] | $X | $Y | [Monthly / annual / variable] |
| **Total** | **$X** | **$Y** | |

Income volatility: [low / moderate / high — based on monthly variance].

## Your Expenses (last 12 months)

| Category | Annual | Monthly Avg | Notes |
|---|---|---|---|
| Housing (rent/mortgage) | $X | $Y | [details] |
| Utilities (electric/gas/water) | $X | $Y | [LIHEAP-relevant] |
| Food (groceries) | $X | $Y | |
| Phone/Internet | $X | $Y | [Lifeline/ACP-relevant] |
| Childcare | $X | $Y | [If present] |
| Healthcare | $X | $Y | |
| Debt service (interest, payday loans) | $X | $Y | **[Flag if >15% of income]** |

## Programs You Likely Qualify For

For each likely-eligible program:

### [Program Name]
- **Estimated value**: $X/year
- **Why you likely qualify**: [one sentence based on income/household]
- **What it covers**: [one sentence on what the program does]

## What to Bring to Your Appointment

[Customized to likely-eligible programs. Include items based on which programs to enroll in. Always include the basics:]

**Identity & status:**
- [ ] Government photo ID (driver's license or state ID)
- [ ] Social Security cards for everyone in household
- [ ] Birth certificates for children
- [ ] Proof of citizenship/lawful presence (passport, green card) — if applicable

**Income proof (last 30 days):**
- [ ] 4 most recent paystubs (or 1099 / SSA award letter / pension statement)
- [ ] Most recent tax return (1040)
- [ ] Bank statements for last month

**Address proof:**
- [ ] Lease or mortgage statement
- [ ] Recent utility bill in your name

**Expense proof (only what applies):**
- [ ] Most recent electric/gas/water bill — for LIHEAP
- [ ] Childcare provider bill — for childcare subsidy
- [ ] Medical bills — for Medicaid spend-down
- [ ] Disability documentation — if applicable
- [ ] DD-214 — if veteran

## Programs to Ask About at Your Appointment

[Bulleted list — exactly the programs identified above as likely-eligible, plus general "ask about" items:]

- [ ] [Program 1] — for [what]
- [ ] [Program 2] — for [what]
- [ ] **Emergency rent or utility assistance** if you're behind
- [ ] **VITA free tax preparation** (deadline April 15) — for unclaimed EITC
- [ ] **Weatherization Assistance Program** — free home insulation, lowers utility bills permanently

## Useful Follow-up Questions to Ask

- "Is there an emergency assistance fund I might qualify for if I'm behind on rent or utilities?"
- "Can you help me apply for SNAP today, or do I need to do it separately?"
- "Do you offer free tax preparation through VITA?"
- "Are there any other programs I should know about for my situation?"
- "Can I schedule a follow-up if I have more questions?"
- "Is there a food pantry I can use today?"

## Where to Go

**Your local Community Action Agency:** [If known, name + phone + address. Otherwise: "Call 2-1-1 or visit https://communityactionpartnership.org/find-a-cap/ and search by your ZIP code."]

**2-1-1** — Free 24/7 line that connects you to local services in any state. Just dial 2-1-1 from any phone.

**Tips for the call/appointment:**
- Most CAAs are open M-F business hours; some have extended hours
- Appointments are usually 1-2 hours
- Bring everything on the checklist above — missing documents is the #1 reason appointments don't fully succeed
- If you can't make it in person, ask if they offer phone or virtual appointments
- Bring a notebook to write down what they enroll you in and follow-up dates

---
*This packet is generated from your Cashflow transaction data and answers you provided. It's a screening tool, not a final eligibility determination — only the agency can confirm what you qualify for. Information accurate as of [generation date].*
```

## Step 6: Tone and follow-up

After delivering the packet, ask the user:

- "Want me to also draft an email or phone script you could use to introduce yourself when you call the agency?" — offer this; it removes a barrier.
- "Anything specific you're worried about asking?" — sometimes people feel embarrassed to ask about benefits; address it head-on.
- Don't lecture or moralize. Don't say things like "you should really apply for SNAP." Say "based on your income, you'd likely qualify for SNAP — that's around $X/month for your household. The agency can sign you up in one visit."

If the user reports back later that they got the appointment / got enrolled / got denied, that's valuable. Acknowledge wins specifically. If denied, ask what the agency said and offer to help them appeal or try a different program.

## What this skill is NOT

- **Not financial advice.** We're surfacing public eligibility information.
- **Not a benefits counselor.** Final eligibility comes from the agency, not from us.
- **Not for users above ~200% FPL** for most programs. EITC has higher cutoffs but most safety-net programs don't.
- **Not a replacement for a human appointment.** The packet *prepares* the user for the appointment; it doesn't replace it.
- **Not for emergencies.** If the user is currently being evicted, in food insecurity TODAY, or facing utility shutoff, the response is "Call 2-1-1 right now — they have emergency-response specialists. Then we can also work through this packet for the longer-term enrollments."

## When the data clearly doesn't fit

If income is way above the threshold, household composition doesn't match the programs (e.g. single high-earner with no dependents), or the user is clearly already getting everything they're eligible for, say so cleanly:

> "Based on what I see, you're either above the income range for these programs or you're already enrolled in what you qualify for. If something specific is going on — a job loss, an unexpected expense, helping a family member — tell me what it is and I can suggest something more targeted."

Then exit gracefully. Don't force the workflow on someone it doesn't fit.
