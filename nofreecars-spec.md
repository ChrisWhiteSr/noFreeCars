# No Free Car — Project Spec & AI Context

## What This Document Is

This is a combined spec and AI context file for the "No Free Car" project. It serves two purposes:

1. **Product spec** — what we're building, why, and how it should work
2. **AI context** — paste this into any LLM session to get it up to speed instantly

---

## Origin Story

The project started as a conversation about whether to sell a 2021 Audi S5 Sportback (68k miles, ~$33k trade value), invest the freed capital in the S&P 500, and switch to a cheaper car (lease or buy). The core insight: most people never calculate the true net cost of owning a car when you factor in the opportunity cost of the capital sitting in a depreciating asset.

The owner (Chris) explored trades into a Tesla Model 3 lease, VW GTI, Honda Civic Si, Acura Integra Type S, and Toyota GR Corolla — running the numbers on each. The pattern was always the same: sell the expensive car, invest the proceeds, buy something cheaper, and the investment growth + operating savings + depreciation avoided often makes the new car nearly "free."

That's the name: **No Free Car** — because there's no such thing, but you can get close.

---

## Product Vision

A single-page web calculator that answers one question: **"What does switching cars actually cost me after investment returns and tax benefits?"**

Target user: car enthusiasts and financially-minded buyers who want to model the real cost of a vehicle swap, not just the monthly payment.

---

## V1 — Current State (Working Prototype)

Single HTML file, zero dependencies, vanilla JS. Dark theme, JetBrains Mono, slider-based inputs. The current prototype now includes a short premise/explainer near the top, saved scenarios, and a yearly net-worth chart with break-even timing.

### Architecture

- Pure HTML/CSS/JS, no build step, no framework
- Input state is in the DOM (slider values + fuel toggles); saved scenarios persist snapshots in `localStorage`
- Calculations are shared through projection helpers so the headline, yearly chart, break-even text, and saved scenario summaries use the same model
- Designed mobile-first (tested on phone at dealer)

### Current Input Structure

**Hard Numbers (calculated, blue indicators):**
- Current car sale/trade value
- Current car loan balance
- Current car finance rate (APR) + remaining term → monthly payment and remaining balance (derived)
- Annual miles driven
- Fuel type toggle (Gas / EV)
  - Gas: MPG + price per gallon → monthly fuel cost (derived)
  - EV: kWh/100mi + electricity rate → monthly fuel cost (derived)
- New car purchase price
- Down payment
- Transaction costs
- New car finance rate (APR) + term → monthly payment (derived)
- New car fuel inputs (same gas/EV toggle)
- Business use % and marginal tax rate

**Soft Numbers (estimated, amber indicators with bear/base/bull presets):**
- Insurance per month (both cars)
- Maintenance per month (both cars)
- New car residual value at end of analysis period (heavy/avg/light depreciation presets)
- S&P annual return % (bear 5% / base 8% / bull 10%)

### Current Output Structure

**Monthly Cost Cards** — side by side for current vs new car:
```
Loan payment:    $730/mo   [calc]
Fuel:            $146/mo   [calc]
Insurance:       $150/mo   [est]
Maintenance:     $100/mo   [est]
────────────────────────────
Total:         $1,126/mo
```

**Net Position Result:**
- Big number: net wealth advantage/disadvantage vs keeping current car
- Two-path breakdown: keep current car vs switch cars, including end value/equity, operating costs, loan payments, transaction costs, tax benefit, and effective monthly cost after offsets
- Yearly net-worth chart plotting keep vs switch positions over the analysis period
- Break-even label showing when the switch path overtakes the keep path, if it does within the selected period

**Saved Scenarios:**
- Top-of-page saved scenario strip
- Save the current slider/fuel-toggle state as a named scenario
- Saved scenarios are visible to anyone using the same browser/profile
- Load or delete saved scenarios from the top section
- Stored in `localStorage` under `nofreecar.scenarios.v1`

### Key Design Decisions

1. **Hard vs soft number separation** — every number is tagged `[calc]` or `[est]` so the user knows what's math and what's a guess. Blue slider thumbs for calculated inputs, amber for estimates.
2. **No mystery lump sums** — loan payment, fuel, insurance, maintenance are all separate visible lines. Early prototype had a single "monthly operating" slider which was confusing.
3. **Bear/base/bull on soft numbers only** — we don't pretend estimates are precise. Presets give quick scenario toggling.
4. **Depreciation modeled at 15%/year on current car** — rough but reasonable for luxury/performance cars. Could be refined.

---

## V2 — Planned Features

### Save & Compare Instances
- Basic local save/load/delete is implemented in V1 via `localStorage`
- Remaining enhancement: richer comparison view with side-by-side cards showing car sold, car bought, net 5-year delta
- Potential visual: ranked list or bar chart across all saved scenarios

### Community Comparison Feed (V3+)
- Anonymous submissions: car sold → car bought → net delta
- Leaderboard of real trades people have made
- Data model:
  ```
  {
    sold: { year, make, model, sale_price },
    bought: { year, make, model, purchase_price },
    net_delta_5yr: number,
    assumptions: { sp_return, biz_use_pct },
    timestamp: ISO date
  }
  ```
- Requires a simple backend (SQLite + tiny API, or Firebase, or Supabase)

### Lease Mode
- Toggle between Finance and Lease for the new car
- Lease inputs: monthly payment, down payment, term, mileage allowance
- Lease has no equity at end — output reflects that
- Money factor / APR equivalent display

### Vehicle Presets
- Quick-load common vehicles with pre-filled MPG, insurance range, maintenance range
- Not a database — just a handful of curated presets (Tesla Model 3, GR Corolla, GTI, Civic Si, etc.)
- User can override any preset value

### Section 179 / Bonus Depreciation
- Toggle for first-year accelerated deduction
- Vehicle weight class matters (under/over 6,000 lbs GVWR)
- Cap display for "luxury auto" limits ($20,200 first year for light vehicles)
- Adjusts tax benefit calculation in year 1

---

## Technical Notes for AI Assistants

### When working on this project:

1. **Keep it vanilla.** No React, no build tools, no npm. Single HTML file is the goal for V1-V2. If we need a backend for V3 community features, keep it minimal (Express + SQLite or equivalent).

2. **Mobile-first.** This was literally built and tested at a car dealership on a phone. All inputs must be thumb-friendly. Sliders over text inputs. Big touch targets.

3. **Dark theme only for now.** The aesthetic is intentional — JetBrains Mono, dark background (#0a0f1a), blue for calculated, amber for estimated, green for positive results, red for negative.

4. **The math matters.** Loan amortization uses standard formula. Investment uses compound growth. Depreciation on current car is 15%/year compound. The app uses shared projection helpers, called by `calc()`, to keep headline output, chart output, break-even text, and saved scenario summaries consistent.

5. **Don't over-engineer.** The user (Chris) is a vibe coder with real dev environments. This prototype is for sorting out the interface and logic. Production deployment will happen separately.

6. **Accuracy labeling is non-negotiable.** Every output must be tagged as calculated or estimated. This is the core UX principle — never hide the uncertainty.

### Key formulas:

```
Monthly payment = P * r * (1+r)^n / ((1+r)^n - 1)
  where P = principal, r = monthly rate, n = months

Loan balance at month t = P * (1+r)^t - PMT * ((1+r)^t - 1) / r

Investment FV = PV * (1 + annual_return)^years

Depreciation avoided = current_car_value - (current_car_value * 0.85^years)

Tax benefit = (deductible_expenses * biz_use_pct) * marginal_tax_rate

Net position = (investment_gain + operating_savings + depreciation_avoided + tax_benefit + new_car_equity) - (total_loan_payments)
```

### File structure (current):

```
nofreecars.html    ← everything, single file
```

### File structure (planned V2):

```
index.html         ← app shell
css/
  styles.css       ← extracted styles
js/
  calc.js          ← calculation engine
  ui.js            ← DOM manipulation, slider bindings
  storage.js       ← localStorage save/load
  presets.js       ← vehicle preset data
```

### File structure (planned V3 with backend):

```
client/
  index.html
  css/styles.css
  js/calc.js, ui.js, storage.js, presets.js
server/
  index.js         ← Express or similar
  db.js            ← SQLite connection
  routes/
    submissions.js ← community feed CRUD
data/
  nofreecars.db    ← SQLite file
```

---

## Context: Chris's Situation (for reference)

- Based in Baltimore, MD
- Owns a 2021 Audi S5 Sportback, 68k miles, ~$33k trade value
- Has an LLC, can deduct 80% business use
- Currently evaluating: Toyota GR Corolla Premium Plus (used 2024, ~$38-40k)
- Previous cars: E30 318, E39 M5, two E36 M3s, WRX wagon, 06 STI, 996 GT3, Alfa Giulia Ti Sport, ND2 Miata
- Priorities: small, planted, AWD/RWD, manual preferred, practical enough for daily + cargo
- Investment thesis: sell S5, invest $33k in S&P, buy cheaper car, net advantage over 3-5 years

---

## Open Questions

1. Should depreciation model be more sophisticated? (Curve vs linear, make/model specific)
2. Is there value in pulling live gas prices or S&P returns via API?
3. Should we support multi-car scenarios? (e.g., sell one car, buy two cheaper ones)
4. How to handle trade-in vs private sale price difference in the UI?
5. Community feed: anonymous only, or allow usernames?
6. Should we calculate break-even lease payment? (The monthly lease price where switching stops being advantageous)
