# No Free Car - Calculation Review & Recommendations

Review date: 2026-05-05

## Overall Assessment

The project goal is strong and coherent: the calculator is trying to compare the wealth impact of keeping a current car versus selling it, investing freed equity, and financing a replacement. The HTML prototype matches the V1 product direction well: single-file, mobile-first, visible assumptions, and clear separation between calculated and estimated inputs.

The basic arithmetic for fuel cost, new-car loan payment, remaining loan balance, investment growth, and residual-value depreciation is mostly correct. With the current default scenario, where the current car has no loan balance, the app's net position calculation reconciles with the formula in the spec.

The main accuracy issue is not arithmetic. It is the financial model. The current code mixes asset values, cash flow, tax effects, and debt payoff in ways that are reasonable for a quick prototype but can mislead users in common real-world cases.

## Default Scenario Check

Using the defaults in `nofreecars.html`:

- Current car value: `$33,000`
- Current loan balance: `$0`
- Current fuel: `$175/mo`
- Current insurance: `$180/mo`
- Current maintenance: `$170/mo`
- New car price: `$35,000`
- APR / term: `3.9% / 60 months`
- New fuel: about `$130/mo`
- New insurance: `$150/mo`
- New maintenance: `$80/mo`
- New residual: `$22,000`
- S&P return: `8%`
- Analysis period: `3 years`

Independent recalculation:

- New loan payment: about `$643/mo`
- Investment gain on `$33,000`: about `$8,570`
- Operating savings: about `$5,953`
- Depreciation avoided: about `$12,734`
- New loan payments during period: about `$23,148`
- Remaining loan balance after 36 months: about `$14,822`
- New car equity at period end: about `$7,178`
- Net position versus keeping current car: about `+$11,287`
- True monthly cost of switch: about `$565/mo`

For this zero-current-loan default case, the app's math is directionally accurate and internally consistent.

## Accuracy Findings

### 1. Current-car loan balance is not modeled completely

The UI accepts a current loan balance, but it does not collect the current loan APR, remaining term, or monthly payment. The monthly current-car card also omits current loan payment, even though the spec says the monthly cards should show it.

Impact: once `curLoan` is above zero, the comparison becomes less reliable. The app reduces investable equity, but it does not model what happens if the user keeps the current car and continues paying the current loan. It also does not clearly model negative equity when the loan balance exceeds the sale value.

Recommendation: add current-car loan payment inputs or derived current-loan amortization:

- Current APR
- Current remaining term
- Current monthly payment, either calculated or manually entered
- Current loan balance at the end of the analysis period
- Payoff / negative-equity line item when switching

### 2. Tax benefit is materially overstated

The current tax formula is:

```js
taxBen = (totP + newOpsT) * businessUse * taxRate
```

This treats all new-car loan payments as deductible. In normal U.S. tax treatment, principal repayment is not deductible. Depending on the accounting method and business facts, deductible items may include business-use portions of interest, fuel, insurance, maintenance, registration, lease payments, depreciation, or standard mileage, but not all loan principal.

Impact: enabling business use can make the switch look much better than it really is. It also compares tax benefit for the new car only; if the current car is also business-used, the calculator should include the tax benefit of keeping it and compare the delta.

Recommendation: split tax modeling into explicit modes:

- No business tax treatment
- Standard mileage method
- Actual expense method
- Lease method
- Depreciation / Section 179 mode later

At minimum for V1, calculate deductible loan interest separately from principal and show a warning that tax output is an estimate.

### 3. Investment return should use investable net proceeds, not just positive equity

The app uses:

```js
freed = Math.max(0, currentValue - currentLoan)
```

This is fine for positive equity, but it hides negative equity. If the current loan is underwater, the user needs cash to close the old loan or roll the negative equity into the new deal.

Recommendation: explicitly calculate:

- `netSaleProceeds = currentValue - currentLoan`
- `investedAmount = Math.max(0, netSaleProceeds)`
- `cashRequiredToExit = Math.max(0, -netSaleProceeds)`

Show negative equity as a separate cost line.

### 4. The result double-counting risk is low at default, but the breakdown can confuse users

For a paid-off current car, the app's net formula is equivalent to:

```text
investment gain
+ operating savings
+ depreciation avoided
+ tax benefit
+ new car equity
- new car loan payments
```

That is consistent with the spec. However, the implementation actually compares:

```text
switch wealth = investment future value + new equity - loan payments - new operating costs + tax benefit
keep wealth = old car end value - current operating costs
```

This is a better conceptual structure than the simplified spec formula, but the displayed breakdown should match it more transparently.

Recommendation: present the result as two columns:

- Keep current car: ending car value, operating costs, loan costs, tax effect
- Switch cars: investment ending value, new car equity, operating costs, loan costs, tax effect

Then show the delta. This will reduce confusion and make audits easier.

### 5. Current-car depreciation is simplistic and mislabeled in the spec

The spec says "15%/year linear", but the code uses compound depreciation:

```js
oldEnd = currentValue * 0.85 ** years
```

Compound depreciation is usually more defensible than linear, but the spec should be corrected. Vehicle depreciation also varies enormously by age, mileage, brand, and segment.

Recommendation: update the spec to say "15% annual compound depreciation" or change the code if linear depreciation was intended. Longer term, add presets or editable depreciation assumptions for the current car as well as the new car.

### 6. New residual value can become unrealistic

The residual is a free slider with presets based on purchase price and analysis years. That is useful, but it is not constrained by loan term, mileage, taxes, fees, or market values. Users can set a residual that makes the deal look artificially favorable.

Recommendation: keep the manual slider, but display residual as both dollars and percent of purchase price. Add a quick sanity label like `63% of purchase price after 3 years`.

### 7. Transaction costs are omitted

The note correctly says transaction fees are not included. For real decisions, this is a major missing cost:

- Sales tax
- Dealer processing fee
- Title / registration
- Inspection / emissions
- Trade-in tax credit, if applicable
- Private-party sale friction
- Shipping or travel
- Immediate tires, brakes, PPI, or warranty work

Recommendation: add a compact "Transaction costs" section with separate new-car and sale-side costs. Default it to zero or a conservative preset.

### 8. New-car financing assumes zero down payment and no taxes/fees financed

The loan payment currently uses purchase price as principal. Real financed principal may be:

```text
purchase price
+ taxes and fees
+ negative equity
- down payment
- trade credit applied to purchase
```

Recommendation: add down payment and financed taxes/fees inputs. This is especially important when comparing lease, finance, and cash purchase modes.

### 9. "True monthly cost of switch" needs clearer definition

The current formula is:

```js
trueMonthly = (loanPayments + newOperatingCosts - newEquity - taxBenefit - investmentGain) / months
```

This is useful, but it is not the same as the monthly delta versus keeping the current car because it excludes current operating costs and current depreciation. The big net result is "versus keeping current car"; the monthly result is closer to "net cost of the new path after offsets."

Recommendation: rename it to one of:

- `Net monthly cost after offsets`
- `Effective monthly cost of switch path`

Also add a second line:

- `Monthly advantage vs keeping current car`

### 10. Hard vs soft labeling is good but incomplete

The UI labels estimated inputs well, but some outputs combine calculated and estimated values. For example, net position includes investment return, depreciation, residual value, maintenance, insurance, and tax assumptions.

Recommendation: tag the final net result as estimate-heavy, even though the arithmetic is calculated. A label like `model estimate` would be more honest than implying the final number is purely calculated.

## Priority Fixes

1. Add current-car loan payment / APR / remaining term, or remove current loan balance until it can be modeled properly.
2. Replace the tax formula so loan principal is not treated as deductible.
3. Add explicit negative-equity handling.
4. Add transaction costs and down payment / financed-fees inputs.
5. Change the output structure to show "keep" and "switch" scenarios separately before the net delta.
6. Clarify "true monthly cost" wording.
7. Update the spec to match the code's compound depreciation.

## Suggested V1 Calculation Model

A clearer V1 model would compute both paths independently:

```text
Keep path:
  ending current car equity
- current operating costs
- current loan payments during period
+ current tax benefit, if enabled

Switch path:
  future value of invested net proceeds
+ ending new car equity
- new operating costs
- new loan payments during period
- transaction costs
- negative equity payoff
+ new tax benefit, if enabled

Net advantage:
  switch path - keep path
```

This keeps the mental model clean: each path becomes a projected ending position, and the app compares the two.

## Bottom Line

The calculator is accurate enough for a paid-off-car prototype and useful for exploring the core "opportunity cost of car equity" insight. It is not yet accurate enough for decisions involving current auto debt, business deductions, negative equity, taxes, fees, or down payments.

The highest-value improvement is to move from a single net formula to two explicit scenarios: keep versus switch. That will make the app easier to trust, easier to explain, and easier to extend into lease mode and saved comparisons.
