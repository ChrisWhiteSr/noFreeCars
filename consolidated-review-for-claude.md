# No Free Car — Final Calculation Review

Review date: 2026-05-05

## Verdict

The core arithmetic is correct: loan amortization, compound investment growth, fuel cost derivation, and remaining balance all check out. The net position headline number is right for the default paid-off-car scenario.

Two bugs need fixing before this is trustworthy: the tax formula deducts loan principal (overstating benefits by ~25% at 80% business use), and the results breakdown displays interest as a separate negative line item when it's already included in loan payments (making the visible rows not sum to the headline).

Beyond those bugs, the model has real gaps around current-car debt, negative equity, down payments, and transaction costs. These don't need to block a V1 ship, but they should be disclosed.

## Verified Correct

- Loan payment: standard PMT formula, handles 0% APR edge case
- Remaining loan balance: correct amortization schedule math
- Fuel cost: correctly derives monthly from annual miles / MPG or kWh, uses same mileage for both cars (intentional)
- Investment FV: standard compound growth
- Current car depreciation: 15%/year compound (code is better than spec, which says "linear" — update spec to match)
- Residual value presets: 20%/12%/7% annual depreciation rates are reasonable
- Net position formula (`switchW - keepW`): mathematically correct

## Bugs

### 1. Tax benefit deducts loan principal

`nofreecars.html` line 378:

```js
const taxBen = (totP + newOpsT) * bU * tR;
```

`totP` is total loan payments — principal plus interest. Only interest is deductible. At 80% business use / 30% tax rate with defaults, this reports $8,663 when the correct number is closer to $6,940.

**V1 fix:** Keep it conservative. Default business use to 0% (already the case). When enabled, deduct operating costs plus loan interest only — no principal, no depreciation:

```js
const taxBen = (intPaid + newOpsT) * bU * tR;
```

Leave depreciation / Section 179 out of V1 entirely. Economic depreciation (`purchasePrice - residualValue`) is not the same as tax depreciation, which depends on MACRS schedules, luxury auto caps ($20,200 first year for light vehicles), GVWR, and whether the user elects Section 179. Baking a simplified version into the default formula will give users false confidence in a number that's likely wrong. Build it as an explicit opt-in module in V2 when the UI can show the assumptions.

Add a disclaimer: "Tax estimates are not tax advice. Consult a tax professional."

**Also missing:** if the current car is business-used too, the user loses those deductions when they sell it. The relevant number is the tax benefit *delta* between keeping and switching, not the new car's benefit alone. Worth noting in the UI even if V1 doesn't model it.

### 2. Breakdown rows don't sum to the headline

The results breakdown shows:

```
+ Investment gain        +$8,570
+ Operating savings      +$5,953
+ Depreciation avoided  +$12,734
+ Tax benefit                 $0
─────────────────────────────────
- Loan payments         -$23,148
+ New car equity         +$7,178
- Interest paid          -$2,970   ← already inside "Loan payments"
```

These rows sum to $8,317. The headline says +$11,287. The difference is exactly $2,970 — the interest amount, counted twice.

The headline calculation on line 381 is correct. This is a display bug only, but it will make users think the calculator is broken.

**Fix:** Split "Loan payments" into "Principal paid" and "Interest paid" as two separate lines. Drop the combined total. This makes every row an independent component and they'll sum correctly:

```
- Principal paid        -$20,178
- Interest paid          -$2,970
+ New car equity         +$7,178
```

## Model Gaps

These don't break the calculator for the default scenario but limit its usefulness for real decisions.

### 3. Current-car loan is half-modeled

The UI accepts a loan balance (reducing freed capital) but has no APR, no term, and no monthly payment line in the current car cost card. If someone has a $500/mo payment on their current car, that's a real cost of keeping it that the model ignores. This makes switching look worse than it is for users with current auto debt.

**Fix:** Add current car APR and remaining term. Derive the monthly payment. Show it in the current car cost card. Include it in `keepW`.

### 4. Negative equity is silently zeroed out

```js
const freed = Math.max(0, cV - cL);
```

If the loan exceeds the car's value, the user needs cash to close the deal or rolls the deficit into the new loan. The calculator just shows $0 invested and moves on.

**Fix:** Show negative equity explicitly. When `cV < cL`, display a "Cash needed to exit" line and add the deficit to the new loan principal (or show it as a separate switch-path cost).

### 5. No down payment input

The model finances 100% of the purchase price. A down payment reduces financed principal (smaller loan payments, less interest) but also reduces investable capital (lower investment returns). These partially cancel out, but the ratio matters and users can't model it.

### 6. Transaction costs not modeled

Noted in the existing disclaimer. Sales tax alone on a $35k car can be $2k+. Title, registration, dealer fees, and immediate PPI/maintenance add up. A single "transaction costs" input with a sensible default ($2,000-$3,000) would be enough for V1.

### 7. Investment gains are pre-tax

The S&P return is shown as realized wealth, but selling $8,570 in gains triggers long-term capital gains tax (~15-20%). Acceptable for V1 if disclosed; add "investment returns shown pre-tax" to the footer.

### 8. "True monthly cost of switch" is mislabeled

The formula:

```js
trueMonthly = (totP + newOpsT - nEq - taxBen - invGain) / mo;
```

This is the net monthly cost of the new-car path after offsets. It's not the monthly advantage vs. keeping the current car (which would be `-net/mo`). Rename to "Effective monthly cost after offsets" or similar.

## Recommended V1 Output Structure

The cleanest fix for the breakdown problem (and the best foundation for lease mode / saved comparisons) is to show two explicit paths:

```
KEEP current car:
  Car value at end of period          $20,266
- Operating costs over period        -$18,900
- Loan payments (if any)                  $0
= Keep position                       $1,366

SWITCH cars:
  Invested capital at end of period   $41,571
+ New car equity                      +$7,178
- Operating costs over period        -$12,947
- Loan payments over period          -$23,148
+ Tax benefit                             $0
= Switch position                    $12,654

NET ADVANTAGE: +$11,287
```

Every number is visible. The rows sum. The user can audit both sides independently.

## Priority

1. Fix the breakdown display (trust issue — users will think the math is wrong)
2. Fix the tax formula (deducts principal, overstates by ~25% with business use)
3. Add current-car loan modeling (APR, term, monthly payment)
4. Handle negative equity explicitly
5. Add down payment input
6. Add transaction costs input
7. Rename "true monthly cost"
8. Disclose pre-tax investment returns in footer
9. Update spec: "compound" not "linear" depreciation
