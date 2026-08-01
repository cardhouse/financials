# Calculation Acceptance Tests

## 1. Purpose

These examples are executable specifications. Implement them as table-driven Pest unit tests and supporting feature tests. Values are USD minor units in code and shown as dollars here for readability.

Unless a rule says otherwise, division rounds to the nearest minor unit using half-up rounding. Dates and month boundaries use the household timezone.

## 2. Baseline Fixture

Current month has 30 days. Today is day 10.

### Account positions

| Account | Classification | Available-to-spend role | Projected balance |
| --- | --- | --- | ---: |
| Checking | asset | liquid | $4,500 |
| Savings | asset | liquid | $3,000 |
| Cash | asset | liquid | $100 |
| Credit card | liability | reserve from cash | $900 owed |
| Retirement | asset | non-liquid | $10,000 |
| Mortgage | liability | long-term, not cash-reserved | $200,000 owed |

Projected liquid cash is `$7,600`. Short-term liability reservation is `$900`.

### Monthly plan

| Component | Amount |
| --- | ---: |
| Expected income remaining | $3,000 |
| Fixed allocation remaining | $2,800 |
| Known unpaid fixed bills | $2,600 |
| Current positive goal reserves | $4,000 |
| Remaining reserved contributions | $500 |
| Minimum cash buffer | $1,000 |
| Remaining flexible spending | $1,200 |

Fixed reservation is `max($2,800, $2,600) = $2,800`.

```text
Available to Spend
  = 7,600 + 3,000 - 900 - 2,800 - 4,000 - 500 - 1,000
  = 1,400

Projected Month-End Free Cash
  = 1,400 - 1,200
  = 200
```

Expected result: available-to-spend `$1,400`; projected month-end free cash `$200`; headline `on_track` when no other reason exists.

## 3. Planned Flexible Purchases

### Checking purchase

Record a `$100` groceries purchase against checking.

- Liquid cash becomes `$7,500`.
- Credit-card reservation remains `$900`.
- Remaining flexible plan becomes `$1,100`.
- Available-to-spend becomes `$1,300`.
- Projected month-end free cash remains `$200`.

### Credit-card purchase

Starting from baseline, record the same purchase against the credit card.

- Liquid cash remains `$7,600`.
- Credit-card reservation becomes `$1,000`.
- Remaining flexible plan becomes `$1,100`.
- Available-to-spend becomes `$1,300`.
- Projected month-end free cash remains `$200`.

Invariant: a planned flexible purchase consumes discretionary capacity and its category plan equally, so the projected final free cash does not change.

### Purchase without account

Record the same purchase without an account.

- Groceries actual increases and remaining flexible plan becomes `$1,100`.
- Account-derived available-to-spend is marked incomplete.
- The UI must not present a newly precise available-to-spend number.
- Assigning the account later produces the same result as the corresponding example above.

## 4. Fixed Bill and Credit-Card Payment

### Fixed bill paid by credit card

Starting from baseline, pay a `$200` planned fixed bill by credit card. The bill was included in known unpaid.

- Card reservation becomes `$1,100`.
- Fixed allocation remaining becomes `$2,600`.
- Known unpaid becomes `$2,400`.
- Fixed reservation becomes `max($2,600, $2,400) = $2,600`.
- Available-to-spend remains `$1,400`.

Invariant: a planned fixed payment replaces a fixed reservation with a card obligation; it does not reduce available-to-spend twice.

### Credit-card payment

Starting from baseline, pay `$500` from checking to the credit card.

- Liquid cash becomes `$7,100`.
- Card reservation becomes `$400`.
- Account postings are `-$500` checking and `+$500` liability position; sum is zero.
- Available-to-spend remains `$1,400`.
- Budget actual is unchanged.

## 5. Income

### Planned income received

Receive `$1,000` of baseline planned income into checking.

- Liquid cash becomes `$8,600`.
- Expected income remaining becomes `$2,000`.
- Available-to-spend remains `$1,400`.

### Unplanned income received

Receive an unplanned `$500` into checking.

- Liquid cash becomes `$8,100`.
- Expected income remaining stays `$3,000`.
- Available-to-spend becomes `$1,900`.
- No category or goal receives the money automatically.

### Scenario-only income

Add `$500` to a draft scenario.

- Scenario available-to-spend and projected month-end free cash each increase `$500`.
- Active dashboard values remain baseline until adoption.

## 6. Goal and Reserved Contribution Behavior

### Virtual goal contribution

Starting from baseline, contribute `$200` virtually to a goal whose remaining reserved allocation is part of the `$500` baseline total.

- Goal reserves become `$4,200`.
- Remaining reserved contributions become `$300`.
- Liquid cash is unchanged.
- Available-to-spend remains `$1,400`.

### Sinking-fund purchase within reserve

Starting from baseline, atomically record a `$300` checking purchase and `$300` withdrawal from a goal.

- Liquid cash becomes `$7,300`.
- Goal reserves become `$3,700`.
- Available-to-spend remains `$1,400`.
- The purchase is visible as goal use, not as a contribution.

### Confirmed goal overdraft

A goal has `$100` reserved. Spend `$150` from it using checking and confirm the overdraft.

- Cash falls `$150`.
- Goal balance becomes `-$50` and displays `overdrawn`.
- Reservation falls only `$100`, from `$100` to zero.
- Available-to-spend falls `$50`.
- Without the explicit overdraft confirmation, the action fails with no writes.

### Investment contribution

Starting from baseline, transfer `$200` from checking to retirement against a reserved Investing category.

- Liquid cash falls to `$7,400`.
- Retirement asset increases to `$10,200`.
- Remaining reserved contributions fall to `$300`.
- Available-to-spend remains `$1,400`.
- Net worth is unchanged.
- Fixed/flexible spending is unchanged.

## 7. Amortizing Debt Without a Split

Pay a `$500` planned mortgage bill from checking when no principal/interest split is supplied.

- Checking receives a `-$500` posting.
- Fixed actual increases `$500` and remaining fixed reservation falls `$500`, leaving available-to-spend unchanged for the planned payment.
- No mortgage-liability posting is created.
- Mortgage displayed balance stays at its latest observed value and receives `debt_balance_awaiting_observation` freshness detail.
- The next manual mortgage observation becomes the new authoritative balance.

## 8. Fixed Reservation Edge Cases

| Planned remaining | Known unpaid | Expected reservation |
| ---: | ---: | ---: |
| $1,000 | $800 | $1,000 |
| $1,000 | $1,200 | $1,200 |
| -$100 | $0 | $0 |
| -$100 | $250 | $250 |
| no allocation | $250 | $250 |

Known bills are never added on top of the plan remaining.

## 9. Spending-Pace Attention

For a 30-day month, flexible allocation `$500`, actual `$200`:

- On day 6, pace warning is disabled.
- On day 10, projection is `round(200 / 10 * 30) = $600`.
- Tolerance is `max($25, 5% of $500 = $25) = $25`.
- Projected overage is `$100`, so reason `category_projected_over_budget` applies.

For allocation `$2,000`, projection `$2,090`:

- Tolerance is `max($25, $100) = $100`.
- Overage `$90` does not trigger attention.
- Projection `$2,100` triggers because the difference is at least the tolerance.

## 10. Headline Status Precedence

| Conditions | Headline | Additional reasons retained |
| --- | --- | --- |
| No reasons | on track | none |
| Pace warning only | attention needed | projected category reason |
| Category actual exceeds allocation; free cash non-negative | over budget | category reason |
| Projected free cash negative | overallocated | every category/bill/freshness reason also retained |
| Overdue bill and stale balance | attention needed | both reasons |
| Pending adopted-plan action only | attention needed | pending action reason |

Status is never persisted and UI text cannot change precedence.

## 11. Subscription Scenario

A `$30` monthly subscription has unpaid occurrences on August 15, September 15, and onward. July is already paid. Scenario cancellation effective September 1:

- July paid actual remains.
- August unpaid occurrence remains in scenario.
- September and later unpaid occurrences are excluded.
- Next-12-month savings is the sum of exactly the excluded scheduled occurrences in that window.
- Editing the scenario does not change the bill template or occurrences.

After adoption:

- A pending cancellation action exists.
- Real unpaid bills remain reserved until confirmation.
- Confirming September 1 ends the template and cancels unpaid occurrences due September 1 or later.
- Choosing `Keep subscription` creates a corrective active revision and dismisses the action.

## 12. Recurrence Boundaries

- Monthly recurrence on day 31 produces February 28 in 2027 and February 29 in 2028.
- The next occurrence returns to March 31; fallback does not permanently change the intended day.
- Generating the same horizon twice produces no duplicate occurrences.
- Template edits do not overwrite a manually modified occurrence.

## 13. Account Observation Cutoff

Checking observation: `$1,000` effective July 10 at noon. Purchase posting `-$100` July 10 at 1 PM. New observation `$850` July 11 at noon.

- Before the second observation, projected balance is `$900`.
- After the second observation, projected balance is `$850`; the earlier purchase is absorbed by the new baseline.
- A later `-$25` posting produces `$825`.
- No transaction is deleted or altered by either observation.

## 14. Scenario Isolation and Adoption

- Creating a scenario copies an explicit base revision and records lineage.
- Editing its allocations/income/subscription overrides changes no active-plan row or dashboard result.
- A stale `lock_version` update fails without overwriting the newer scenario.
- Adoption either creates the full active revision, pointer change, supersession, audit, and pending actions or creates none.
- Adopting twice is idempotently rejected; exactly one active-plan pointer remains.

## 15. Month-End Rollover

At close, Dining has `$200` unused allocation but actual positive free cash is `$120`.

- Maximum Dining rollover is `$120`, not `$200`.
- Rolling `$100` to next month's Groceries succeeds and leaves `$20` allocatable.
- The remaining `$20` must be assigned explicitly, for example to `leave as cash`, before close.
- It creates a new next-month plan revision with Groceries increased `$100`.
- It creates no account posting or transaction.
- An unselected `$100` of Dining allocation expires.

## 16. Closed-Month Corrections

- Normal writes to a closed period fail before persistence.
- Reopening requires a nonblank reason, records actor/time/reason, and moves status to `reviewing`.
- Corrections recalculate the period.
- Re-closing preserves both close events and prior month-end decision history.

## 17. Goal Forecasts

- Target `$1,000`, current `$400`, three contribution opportunities: shortfall `$600`, required monthly `$200`.
- Target `$1,000`, current `$399`, three opportunities: shortfall `$601`, required monthly `$200.34` by ceiling division.
- Shortfall `$500`, monthly contribution `$200`: completion requires three months, not two and a half.
- Completed target returns zero required contribution and completed state.
- Positive shortfall with zero contribution returns no completion date.
- Passed target date with no opportunity returns `target_date_passed`.

## 18. Debt Payoff Estimates

- Balance `$1,000`, APR `0%`, planned payment `$300`: four monthly payments with final payment `$100`.
- Balance `$1,000`, APR `12%`: first monthly interest is `$10.00` using exact scaled-integer arithmetic.
- With `$100` planned payment, first principal reduction is `$90.00`.
- A payment equal to or below first-month interest returns `not_amortizing`.
- Simulation terminates at 1,200 months with `beyond_projection_horizon` if not otherwise paid.
