# No Free Car -- Calculation Evaluation & Recommendations

## Summary

The core financial math (loan amortization, compound investment growth, fuel cost derivation) is correct and well-implemented. However, there are two significant calculation issues and several smaller gaps that affect accuracy, especially for users with business use deductions.

---

## Calculation Issues

### 1. Tax Benefit Formula Overstates Deductions (HIGH impact)

**Location:** `nofreecars.html` line 378

```js
const taxBen = (totP + newOpsT) * bU * tR;
```

**Problem:** `totP` is total loan payments (principal + interest), but only interest is tax-deductible, not principal repayment. The formula treats every dollar of loan payment as a deductible expense.

**Impact:** At 80% business use / 30% tax rate with default values, the formula reports **$8,663** in tax benefit when the correct number is **$6,940** -- an overstatement of **$1,723** (25% too high). The error scales with loan size and business use percentage.

**Fix:** Replace with:

```js
const newCarDep = nP - nRes; // simplified depreciation of new car
const taxBen = (intPaid + newOpsT + newCarDep) * bU * tR;
```

This deducts interest (not principal), operating expenses, and depreciation -- the actual IRS-deductible items for business vehicle use.

---

### 2. Breakdown Display Appears to Double-Count Interest (MEDIUM impact)

**Location:** `nofreecars.html` lines 291-293

The results breakdown shows:

```
- Loan payments      -$23,148
+ New car equity     +$7,178
- Interest paid      -$2,970    <-- already included in loan payments
```

**Problem:** Interest paid is a subset of total loan payments. Showing it as a separate negative line makes the breakdown items sum to **$8,317** while the actual displayed net is **$11,287**. The "big number" net calculation (line 381) is correct -- this is purely a display/presentation issue. But users who mentally add up the breakdown will get a different number.

**Fix:** Either:
- (A) Remove the "Interest paid" line entirely (it's informational but misleading as a line item), or
- (B) Split "Loan payments" into two lines: "Principal paid" (negative) and "Interest paid" (negative), and remove the combined "Loan payments" line, or
- (C) Keep it but visually indent/gray it with a label like "of which interest:" to show it's a sub-item, not a separate component

---

## Accuracy Gaps (Lower Priority)

### 3. Current Car Loan Interest Not Modeled

There's a "Loan balance" input for the current car, but no APR. The loan balance correctly reduces freed capital (and thus investment returns), but if the user is actively paying interest on that loan while keeping the car, that cost isn't captured. This understates the cost of keeping the current car.

**Recommendation:** Add a "Current car APR" slider, or add a note that the model assumes the current car loan is interest-free.

### 4. No Down Payment Input for New Car

The model finances 100% of the purchase price. A user putting $10k down on a $35k car should finance $25k, but there's no way to express this. The down payment also represents capital that can't be invested, which affects the opportunity cost calculation.

**Recommendation:** Add a down payment slider. Adjust the loan principal to `nP - downPayment` and reduce `freed` capital by the down payment.

### 5. Capital Gains Tax on Investment Returns

The S&P returns are pre-tax. A user who invests $33k and earns $8,571 over 3 years would owe long-term capital gains tax (~15-20%) when selling. The model overstates the realized investment gain.

**Recommendation:** Either add a capital gains tax rate input, or note this assumption in the disclaimer.

### 6. Spec Says "15%/year Linear" but Code Uses Compound

The spec describes depreciation as "15%/year linear" but the code uses `cV * 0.85^y` (compound). For 3 years: compound gives $12,734 depreciation vs. linear which would give $14,850. The compound approach is actually more realistic for car depreciation curves, so the code is arguably better than the spec. Just a consistency note.

### 7. Current Car Monthly Card Missing Loan Payment

The spec shows a "Loan payment" line in the current car monthly breakdown, but the HTML doesn't include one. If the user has a current car loan, they're making payments on it -- those should appear as a cost of keeping the current car.

---

## What's Working Well

- **Loan amortization formula** (line 313): Standard PMT formula, correctly implemented, handles 0% APR edge case.
- **Remaining balance calculation** (line 314): Correct amortization schedule math.
- **Fuel cost derivation** (lines 316-320): Correctly derives monthly fuel from annual miles, MPG/kWh, and price. Uses `curMiles` for both cars (intentional -- same driver, same miles).
- **Investment compound growth** (line 370): Standard FV formula, correct.
- **Depreciation avoided** (line 373): Compound depreciation, realistic curve.
- **Residual value presets** (line 400): Heavy/avg/light depreciation rates (20%/12%/7% per year) are reasonable ranges.
- **Net position calculation** (lines 379-381): The `keepW` vs `switchW` comparison is correct -- the "big number" is right even though the breakdown display is misleading.
- **Hard vs. soft number UX**: Blue/amber distinction with calc/est tags is well-executed and solves a real problem in financial calculators.
- **Edge cases**: Handles analysis period > loan term correctly (zeroes out remaining balance).

---

## Suggested Priority Order

1. Fix tax benefit formula (changes dollar output, affects users with business use)
2. Fix breakdown display (users will notice the math doesn't add up)
3. Add down payment input (common real-world scenario)
4. Add current car APR (affects accuracy for users with existing loans)
5. Note capital gains tax assumption in disclaimer
6. Sync spec language on depreciation model
