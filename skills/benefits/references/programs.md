# Benefits Programs Reference

Detailed eligibility rules + value estimates for each program the `benefits` skill screens for. Read this when running the eligibility step.

## Federal Poverty Level (2026, as published by HHS)

The 100% FPL row is in `SKILL.md`. Multipliers most-used in benefits screening:

- **100% FPL** — TANF, some emergency assistance
- **130% FPL** — SNAP gross income test
- **138% FPL** — Medicaid (in expansion states)
- **150% FPL** — many state-level programs, EITC starts phasing out around here for childless filers
- **185% FPL** — WIC, free + reduced-price school meals (free at 130%, reduced at 185%)
- **200% FPL** — most Community Action Agency programs, LIHEAP in many states

**Compute**: `fpl_ratio = annual_income / fpl_for_household_size`. Express as a percentage.

## SNAP (Supplemental Nutrition Assistance Program)

**What it is.** Monthly EBT-card benefit for food purchases. Average benefit is ~$185/person/month (2024); a household of 3 averages ~$540/month.

**Federal income limits:**
- **Gross income** ≤ 130% FPL
- **Net income** (after deductions for housing, childcare, etc.) ≤ 100% FPL
- **Asset limit**: $2,750 ($4,250 if elderly/disabled) — but most states have eliminated the asset test via "Broad-Based Categorical Eligibility"

**State variation:** ~40 states use BBCE which raises the gross income limit to 150-200% FPL. Notable strict states (130% only): Wyoming, Idaho, Mississippi.

**Citizenship:** U.S. citizens + lawfully-present non-citizens after 5 years (some categories qualify sooner — refugees, asylees, certain children).

**Special cases:**
- Households with elderly (60+) or disabled members get different rules and higher deduction limits
- College students have very strict rules (must work 20+ hrs/wk OR have dependents OR be in work-study)
- Households receiving TANF or SSI are categorically eligible

**Estimating value:** Use this rough table (2024 averages):
| Household size | Avg monthly SNAP | Annual |
|---|---|---|
| 1 | $200 | $2,400 |
| 2 | $370 | $4,440 |
| 3 | $540 | $6,480 |
| 4 | $700 | $8,400 |
| 5+ | $900+ | $10,800+ |

These are AVERAGES — actual benefit depends on net income; very-low-income households get the maximum (~$300/month/person), higher-income eligible households get less.

**Trigger to flag SNAP:** income < 130% FPL (federally) OR < 200% FPL (in BBCE states). Always flag if children in household + income < 200% FPL.

## EITC (Earned Income Tax Credit)

**What it is.** A refundable tax credit that effectively boosts the income of low/moderate-income working households. Maximum credit (TY 2025): ~$8,000 for a family with 3+ kids; ~$632 for childless workers.

**Eligibility (TY 2025 income limits, single filer):**
| Children | Max AGI | Max credit |
|---|---|---|
| 0 | $19,104 | $632 |
| 1 | $50,434 | $4,213 |
| 2 | $57,310 | $6,960 |
| 3+ | $61,555 | $7,830 |

For married filing jointly, add ~$7,000 to each AGI cap.

**Other requirements:**
- Must have earned income (wages, self-employment, EWA wages count)
- Investment income must be ≤ $11,950 (TY 2025)
- Must file a tax return (so people who don't file MISS the credit — this is huge)
- Must have valid SSN (not ITIN) for filer + qualifying children
- U.S. citizen or resident alien for full year

**Critical insight:** ~25% of eligible filers don't claim EITC. The biggest reason: they don't file a tax return because they're below the filing threshold. **For Cashflow users with income $5k-$25k who haven't filed taxes, this is potentially the single biggest unclaimed benefit.**

**State EITC supplements:** 30+ states have their own EITC, typically 5-30% of the federal amount. Notable: California (CalEITC + Young Child Tax Credit), New York, Massachusetts, Maryland, Minnesota.

**Trigger to flag EITC:** earned income > $1 AND income < $62k AND has qualifying children, OR earned income $1-$19k childless. Strongly flag if user mentions not filing taxes.

**Where to get help filing for free:**
- **VITA** (Volunteer Income Tax Assistance) — free for incomes < ~$60k. IRS locator: https://irs.treasury.gov/freetaxprep/
- **MyFreeTaxes** — online, sponsored by United Way, no income limit
- **GetYourRefund** — virtual VITA service
- **Most CAAs offer or refer to VITA** during tax season (Jan-Apr)

## LIHEAP (Low Income Home Energy Assistance Program)

**What it is.** Federal grants to states for utility bill assistance (heating + cooling). Some states also fund weatherization separately.

**Eligibility (federal baseline):** household income ≤ 150% FPL OR ≤ 60% state median income (whichever is higher). Many states cap at 200% FPL.

**Benefit size:** $300-$1,500/year typical, depending on state, household size, and energy burden. Cold-climate states (MN, NY, MA, ME) tend to give more; warm-climate states (FL, AZ, TX) give cooling assistance with smaller amounts.

**Special programs within LIHEAP:**
- **Crisis assistance** — emergency funds when shutoff is imminent
- **Weatherization referral** — free home insulation, weather stripping, window replacement, sometimes furnace repair (saves money permanently — typically $300-$700/year reduction in utility bills after weatherization)

**Application:** Through state LIHEAP office or local CAA. Most CAAs handle LIHEAP enrollment directly.

**Trigger to flag LIHEAP:** income < 200% FPL AND user has visible electric/gas/water bills > $50/month. Always flag if utility costs > 10% of income (energy-burdened).

## WIC (Women, Infants, and Children)

**What it is.** Monthly supplemental food benefits + nutrition counseling + breastfeeding support for pregnant/postpartum women and children under 5.

**Eligibility:**
- Income ≤ 185% FPL (some states use higher thresholds)
- Pregnant, postpartum (up to 6 months for non-breastfeeding, 12 months breastfeeding), OR has a child under 5
- Resident of state where applying
- Adjunctively eligible if already enrolled in SNAP, Medicaid, or TANF

**Benefit size:** Variable by participant; typical monthly value $50-$150 in food vouchers (specific items: milk, formula, cereal, fruits/vegetables, eggs, etc.). Plus the nutrition counseling and breastfeeding peer support are independently valuable.

**Trigger to flag WIC:** household income < 185% FPL AND (pregnant OR postpartum OR child under 5). Underutilized — many eligible families don't enroll because they think SNAP is better; both can be received.

## Medicaid

**What it is.** State-administered health insurance for low-income people.

**Eligibility (varies dramatically by state):**

**Expansion states (40+ states as of 2026):** adults at ≤ 138% FPL qualify. Just need to apply.

**Non-expansion states (10): TX, FL, GA, AL, MS, SC, TN, WI (partial), KS, WY** — adults only qualify if they're parents/caretakers, pregnant, disabled, or in specific categories. Childless adults may have NO Medicaid path until they hit 100% FPL and the federal marketplace subsidies kick in (the "coverage gap").

**Children always qualify (CHIP)** at higher income thresholds — typically 200-300% FPL depending on state.

**Pregnant women** qualify in all states at higher thresholds (typically 138-200% FPL).

**Asset tests:** Most states have eliminated for non-elderly adults. Still apply for elderly/disabled long-term care eligibility.

**Trigger to flag Medicaid:** income < 138% FPL (in expansion states) OR has children/pregnant/disabled (anywhere). Strongly flag if user has medical bills visible in transactions but no apparent insurance premium.

## Free + Reduced-Price School Meals

**What it is.** Free or reduced-price breakfast and lunch at K-12 schools.

**Eligibility:** Family income ≤ 130% FPL (free) OR ≤ 185% FPL (reduced price; ~$0.40/meal). Some districts have universal free meals via Community Eligibility Provision.

**Application:** Through child's school, usually one form per family per year. Often automatic if family is on SNAP or TANF.

**Trigger to flag:** household income < 185% FPL AND school-age children.

## Supporting Programs (mention in packet)

### Lifeline Phone/Internet
- $9.25/month off phone bill (or qualifying internet service)
- Eligible: SNAP/Medicaid/SSI/Federal Public Housing recipients, OR ≤ 135% FPL
- Multiple providers: SafeLink, Assurance, Q Link, etc.

### ACP (Affordable Connectivity Program)
- *NOTE*: ACP funding ended June 2024; check current status. If revived, $30/month off internet ($75 on tribal lands). Verify status at fcc.gov/acp before recommending.

### Weatherization Assistance Program (WAP)
- Often bundled with LIHEAP referral
- Free home insulation, weather stripping, sometimes new HVAC
- Saves $300-$700/year on utility bills permanently
- Eligibility tracks LIHEAP

### Child Care Subsidy (state-administered, federally funded via CCDF)
- Helps with childcare costs while parents work or attend school
- Eligibility usually < 200-250% FPL (varies by state)
- Apply through state child care assistance office (CAAs typically have referrals)

### TANF (Temporary Assistance for Needy Families)
- Cash assistance for families with kids
- Strictest income limits (typically 50-100% FPL depending on state)
- Has work requirements + lifetime limits (60 months federal, less in some states)
- CAAs help with applications

### SSI (Supplemental Security Income)
- Federal cash benefit for low-income disabled, blind, or 65+ adults
- Strict asset test ($2,000 individual / $3,000 couple)
- Distinct from SSDI (which requires work history); SSI is needs-based
- Not a CAA program per se but they often help with applications

### Emergency Rental Assistance
- Often available through CAAs, typically funded via state ERA programs (post-pandemic some funds remain)
- Covers back rent + sometimes utilities
- Eligibility: at risk of homelessness + income < 80% area median income

### Section 8 Housing Choice Voucher
- Long waitlists (years in most cities) but worth applying anyway
- 30% of income capped; voucher covers the rest up to fair market rent
- Apply through local Public Housing Authority (PHA)

### Head Start / Early Head Start
- Free preschool for kids 0-5 from low-income families
- Eligibility ≤ 100% FPL (typically); some slots for foster kids, homeless families regardless of income
- Apply through local Head Start agency (often part of or affiliated with CAA)

## State-Specific Notes (highest-priority states)

### California
- Has CalEITC + Young Child Tax Credit (boosts federal EITC ~50%+)
- Medi-Cal expansion + covers undocumented adults under 50 statewide
- CalFresh (SNAP) uses BBCE — 200% FPL gross income limit
- LIHEAP via CSD: https://csd.ca.gov

### New York
- NY EITC at 30% of federal
- Medicaid expansion + Essential Plan (138-200% FPL alternative coverage)
- HEAP (NY's LIHEAP)

### Texas
- NO Medicaid expansion — large coverage gap
- SNAP federal limits only (130% gross / 100% net)
- LIHEAP via CEAP (Comprehensive Energy Assistance Program)
- Children's Health Insurance Program (CHIP) up to 201% FPL

### Florida
- NO Medicaid expansion — large coverage gap
- SNAP federal limits
- LIHEAP via state community services

### Iowa
- Medicaid expansion via "IA Health Link" (138% FPL)
- SNAP at 160% FPL (BBCE)
- LIHEAP at 200% FPL

### Arkansas (relevant to Saudí)
- Medicaid expansion via "Arkansas Health and Opportunity for Me" (AR HOME)
- SNAP federal limits
- LIHEAP at 60% state median income
- Has "ARKids First" CHIP for children up to 211% FPL

### Pennsylvania (relevant to TL Queen)
- Medicaid expansion (138% FPL)
- SNAP at 200% FPL (BBCE)
- LIHEAP cash + crisis component

### Georgia
- NO Medicaid expansion
- SNAP federal limits
- LIHEAP via Georgia DHS

### Oklahoma (relevant to Amanda)
- NO Medicaid expansion (note: Oklahoma did expand in 2021 — verify current)
- SNAP at 130% gross / 100% net
- LIHEAP via OKDHS

For other states, default to federal baselines and note that state details may improve eligibility.

## Putting it together

When screening a specific user:

1. Compute their FPL ratio
2. For each program, apply the eligibility logic above + any state-specific rule
3. Categorize each program: "Likely qualifies" / "May qualify — depends on state details" / "Doesn't qualify" / "Already enrolled"
4. Estimate annual value where possible
5. Sum the total potential dollar value of unclaimed benefits — this is THE headline number for the user

**Be honest about uncertainty.** If you're not sure whether a state has BBCE for SNAP, say "depends on state details — the agency can confirm." If you don't know whether ACP funding has been restored, say "ACP eligibility was paused in 2024; ask the agency if it's resumed."

The goal is to give the user a credible, useful starting point — not to perfectly model every benefits program in the country.
