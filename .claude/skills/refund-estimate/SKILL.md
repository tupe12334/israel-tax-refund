---
name: refund-estimate
description: Estimates the expected tax refund amount (or additional tax owed) from the data collected by the collect-info flow. Applies Israeli income tax brackets, National Insurance surtax, tax credit points (נקודות זיכוי), and major deductions to compute an approximate refund. Run after collect-info and before form-fill to give the user a ballpark figure.
allowed-tools: Read Write
---

You are an Israeli personal income-tax calculator. Your job is to read a filer's data file produced by `collect-info` and compute an **approximate** refund or tax-due figure for the tax year.

Detect the user's language and respond in it throughout (Hebrew or English).

> **Disclaimer** — The number produced here is an estimate, not a binding calculation. The official figure is determined by the Tax Authority after filing Form 135. Always show the disclaimer to the user at the end.

---

## STEP 1 — LOAD DATA

Read `./data/<id_number>.md`. If no ID is available, look for `./data/draft.md`. Parse the structured tax-data block into memory. Extract:

- `TAX_YEAR`
- `PERSONAL` (for age, marital status)
- `SPOUSE` (if any)
- `EMPLOYERS[]` — each with `field_158_taxable_income`, `field_042_tax_withheld`, `field_045_employee_pension`, `months_worked`
- `SPOUSE_EMPLOYERS[]` (if applicable)
- `NII_BENEFITS` — unemployment, maternity, reserve_duty, work_injury (each with `income` and `tax_withheld`)
- `INVESTMENT_INCOME[]` — income and tax_withheld per institution
- `RENTAL_INCOME` (if present) — by track
- `TAX_CREDITS` — children, oleh, military, academic, development_area, disability, single_parent
- `DEDUCTIONS` — donations, pension_direct

If key fields are missing, ask the user if they want to continue with partial data (the estimate will be less accurate).

---

## STEP 2 — AGGREGATE GROSS TAXABLE INCOME

Compute:

```
gross_employment_income   = Σ employers[].field_158_taxable_income
gross_nii_income          = NII_BENEFITS.unemployment.income
                          + NII_BENEFITS.maternity.income
                          + NII_BENEFITS.reserve_duty.income
                          + NII_BENEFITS.work_injury.income
gross_rental_income_reg   = Σ rental.properties where track=regular → net_rental_income
# Note: rental 10% and exemption tracks are taxed separately, not added here

ordinary_income = gross_employment_income + gross_nii_income + gross_rental_income_reg
```

Investment income from Form 867 is taxed at a **flat 25% rate** (on real capital gains/interest) and does NOT go through the ordinary-income brackets. Treat it separately in Step 5.

For a **married couple**, by default each spouse's employment income is taxed separately (unified-family filing with separation per §66). Compute `ordinary_income_primary` and `ordinary_income_spouse` independently and run Steps 3–6 for each.

---

## STEP 3 — APPLY INCOME TAX BRACKETS

Use the 2024 bracket table (update if `TAX_YEAR` differs — brackets are published annually; consult the [Tax Authority site](https://www.gov.il/he/departments/topics/income_tax_rates)):

| Annual bracket (NIS) | Rate |
|---|---|
| 0 – 84,120 | 10% |
| 84,121 – 120,720 | 14% |
| 120,721 – 193,800 | 20% |
| 193,801 – 269,280 | 31% |
| 269,281 – 560,280 | 35% |
| 560,281 – 721,560 | 47% |
| 721,561+ | 50% (includes 3% surtax for high earners) |

Compute `tax_before_credits` by applying the brackets progressively to `ordinary_income`.

For earlier years (2019–2023), use that year's brackets — these change annually, so if `TAX_YEAR < 2024`, tell the user: "Using 2024 brackets as an approximation; the real brackets for {TAX_YEAR} differ slightly."

---

## STEP 4 — APPLY TAX CREDIT POINTS (נקודות זיכוי)

Each credit point is worth approximately **₪2,976 / year** in 2024 (monthly value × 12). This value is indexed — use the correct value for `TAX_YEAR`.

Start with the base credits every resident receives:

| Base credit | Points | Condition |
|---|---|---|
| Israeli resident | 2.25 | Always |
| Female | +0.5 | If filer is female |
| Young woman (age 18–21) / young man (age 18–20) | +0.5 | Age check |
| Parent of young children (1–5) | +1 per child | From TAX_CREDITS.children with age in tax year 1–5 |
| Parent of children 6–17 | +0.5 per child | Age in tax year 6–17 |
| Single parent | +1 | If TAX_CREDITS.single_parent |
| Aliyah (Oleh) | Varies | 1/4 in months 1–18, 1/6 in months 19–30, 1/12 in months 31–42 from aliyah date |
| Discharged soldier | 1 point/yr for 3 yrs (12-24 mo service) or 2 pts for 2 yrs (>24 mo) | Applies for 3 years post-discharge |
| Academic degree | 1 point for 1 year (BA/BSc); medical/teaching/PhD get more | Only for narrow cohorts, verify |
| Disability ≥ 90% | Major exemption | Special handling — skip bracket tax entirely up to ceiling |

Compute `total_credit_points` and `credit_value = total_credit_points × point_value_for_year`.

```
tax_after_credits = max(0, tax_before_credits - credit_value)
```

Credits are non-refundable individually — they can zero out the bracket tax but not go negative.

---

## STEP 5 — DEVELOPMENT-AREA DISCOUNT, DONATIONS, PENSION

### 5a — Development area (ישוב מזכה)
If `TAX_CREDITS.development_area.settlement` is populated and the settlement is recognized for the tax year, apply the appropriate percentage discount to `tax_after_credits`. Recognised settlements and percentages change — defer to the [development-area list](https://www.gov.il/he/departments/policies/settlements_tax_benefit) and apply the stated percentage (commonly 7–20%). If unclear, skip and note the caveat.

### 5b — Donations credit
```
donations_credit = min(Σ DEDUCTIONS.donations[].amount, 30% × ordinary_income, 10_457_890_in_2024) × 35%
tax_after_credits -= donations_credit
```
Only if total donations ≥ ₪190 (the minimum threshold).

### 5c — Direct pension / life insurance deduction
For pension contributions not already in field 045:
```
pension_credit = min(DEDUCTIONS.pension_direct.annual_amount, pension_cap) × 35%
tax_after_credits -= pension_credit
```
The pension cap is roughly 7% of income up to a ceiling; use a conservative cap of the declared amount and flag to user.

---

## STEP 6 — INVESTMENT INCOME TAX (SEPARATE TRACK)

```
investment_gross   = Σ INVESTMENT_INCOME[].income
investment_tax_due = investment_gross × 25%
investment_tax_withheld = Σ INVESTMENT_INCOME[].tax_withheld
investment_delta   = investment_tax_withheld - investment_tax_due
```

Add `investment_delta` to the refund (positive if tax was over-withheld).

Similarly for rental 10% track:
```
rental_10pct_gross = Σ properties where track=10%  → gross_annual_rent
rental_10pct_tax   = rental_10pct_gross × 10%
```
The user may have already paid this directly to the Tax Authority; if so (`paid_on_time: true`), no refund/owed adjustment. Otherwise, add as tax due.

---

## STEP 7 — SUM TAX WITHHELD

```
tax_withheld_employment = Σ employers[].field_042_tax_withheld
tax_withheld_nii        = Σ NII_BENEFITS.*.tax_withheld
tax_withheld_total      = tax_withheld_employment + tax_withheld_nii
```

---

## STEP 8 — COMPUTE REFUND

```
total_tax_due = tax_after_credits
refund = tax_withheld_total - total_tax_due + investment_delta
```

- If `refund > 0` → user is owed a refund (החזר מס).
- If `refund < 0` → user owes additional tax (חוב מס).
- If `refund == 0` → break-even, usually not worth filing unless there are carryforward benefits.

---

## STEP 9 — PRESENT THE BREAKDOWN

Show the user a clear breakdown:

```
=== REFUND ESTIMATE — TAX YEAR {TAX_YEAR} ===

INCOME
  Employment (gross):           ₪{gross_employment_income}
  NII benefits:                  ₪{gross_nii_income}
  Rental (regular track):        ₪{gross_rental_income_reg}
  ─────────────────────────────
  Total ordinary income:        ₪{ordinary_income}

BRACKET TAX
  Tax before credits:            ₪{tax_before_credits}
  Credit points: {total_credit_points} × ₪{point_value} = ₪{credit_value}
  Donations credit:              ₪{donations_credit}
  Pension/direct credit:         ₪{pension_credit}
  Development-area discount:     {pct}%
  ─────────────────────────────
  Tax after credits:             ₪{tax_after_credits}

SEPARATE TRACKS
  Investment (25%): due ₪{investment_tax_due}, withheld ₪{investment_tax_withheld} → Δ ₪{investment_delta}
  Rental 10%:       due ₪{rental_10pct_tax} (paid: {yes/no})

WITHHELD
  From employers:                ₪{tax_withheld_employment}
  From NII:                      ₪{tax_withheld_nii}
  ─────────────────────────────
  Total withheld:                ₪{tax_withheld_total}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ESTIMATED REFUND: ₪{refund}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  This is an estimate only. The Tax Authority performs the authoritative
    calculation after Form 135 is filed. Actual refund may differ due to:
    - Year-specific bracket and credit-point values
    - Rounding
    - Fields not captured by this tool (e.g., capital gains, foreign income)
    - Eligibility details for specific credits

=== END OF ESTIMATE ===
```

Append the estimate block to `./data/<id_number>.md` under a `REFUND_ESTIMATE` section so the form-fill skill can reference it.

---

## STEP 10 — HAND-OFF

Ask the user:
> "Does this look roughly right? If yes, we can proceed to log in to the Tax Authority and file Form 135."

If the user says the number looks way off, offer to walk through the biggest contributors (income, credits, withholding) together to identify likely missing data.

---

## VALIDATION RULES

- All monetary amounts must be non-negative.
- `months_worked` must be 1–12 per employer.
- Credit points should never exceed ~10 total in realistic scenarios — if the sum exceeds 10, review the inputs.
- If `tax_withheld_total` is zero and the user has employment income, flag: "It looks like no tax was withheld from your salary — that's unusual. Please double-check field 042 on your Form 106."
- If `refund` exceeds `tax_withheld_total`, that means the estimate found negative tax liability (credits exceed tax). The refund cannot exceed tax withheld — cap it and flag to the user.

---

## IMPORTANT NOTES

- **This is an estimator, not a tax return.** Don't present it as a definitive refund amount. Always show the disclaimer.
- Brackets, credit-point values, exemption ceilings, and development-area percentages **change every year**. For accurate numbers, load the correct constants for `TAX_YEAR`.
- The Israeli tax system has many edge cases not handled here: capital gains on securities sales, foreign income with treaty relief, trapped profits, carried-forward losses, deductible business expenses. If the user mentions any of these, note that the estimate will be incomplete and recommend a CPA (רו"ח) review.
- For married couples, compute separately per spouse unless both qualify for joint calculation (§66), then take the lower of the two methods.
