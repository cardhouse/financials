# Financial Rules

This document defines the business rules for the finance engine.

All calculations are household-wide. `Entered by`/actor identity is audit metadata and may filter history, but never divides the budget or changes household totals.

## 1. Core Concepts

### Total Cash

Total cash is the sum of cash-like accounts such as checking, savings, and physical cash.

### Reserved Cash

Reserved cash is money intentionally set aside for known obligations or goals.

Examples:

- Upcoming fixed expenses.
- Bills due soon.
- Sinking funds.
- Savings goals.

### Available to Spend

Available to spend is the amount of cash that can safely be used after reservations and obligations are considered.

For the active monthly plan:

```text
Available to Spend
  = Current Projected Liquid Cash
  + Expected Income Remaining This Month
  - Short-Term Liability Reservation
  - Remaining Fixed Reservation
  - Current Goal/Sinking-Fund Reservations
  - Remaining Planned Reserved Contributions
  - Minimum Cash Buffer
```

Available to spend may be negative.

Available-to-spend does not subtract the remaining flexible budget. It answers how much cash is safely available for discretionary choices after required and reserved money is protected. The flexible budget answers how the household intends to limit that spending.

The minimum cash buffer is a household-configured amount intentionally kept uncommitted. It defaults to zero during initial setup until the household chooses an amount, is always shown as a separate line in the breakdown, and may make available-to-spend negative.

### Projected Month-End Free Cash

Projected month-end free cash tests whether a plan fits after expected flexible spending:

```text
Projected Month-End Free Cash
  = Available to Spend
  - max(Remaining Planned Flexible Spending, 0)
```

Scenario sliders change this value. It may be negative and is the primary feasibility value in the scenario planner.

### Current Projected Liquid Cash

For each liquid account, start with its latest observed balance and apply known account postings after that observation. Sum the resulting positions. Checking, savings, and cash accounts are liquid by default. Liability, retirement, brokerage, HSA/FSA, and other investment or restricted accounts do not contribute by default.

Users may explicitly change whether an eligible asset account is treated as liquid. Liability accounts can never be liquid.

If a liquid account has no observation, exclude it and show the calculation as incomplete. Do not assume a missing balance is zero.

### Expected Income Remaining

Expected income remaining is the active plan's income expected through the end of the month less income already received against those planned items, floored at zero per item. Unplanned income that has already been received is reflected through projected cash; it is not also added as remaining income. A scenario uses its own planned-income items.

### Short-Term Liability Reservation

Credit-card and other explicitly cash-settled short-term liability balances reduce available-to-spend. Long-term mortgage, auto-loan, and student-loan balances do not; their scheduled payments are fixed obligations instead.

This makes account effects consistent:

- A checking purchase reduces liquid cash.
- A credit-card purchase increases the short-term liability reservation.
- A credit-card payment reduces liquid cash and the reserved card liability by equal amounts, so it is not spending again.
- A purchase with no account still updates its category immediately, but account-based available-to-spend is labeled incomplete until an account is assigned or a new observation reconciles reality.

### Remaining Fixed Reservation

Budget allocations and bills are two views of the same expected fixed spending and must not be added together blindly.

For each fixed category:

```text
Planned Remaining = max(Fixed Allocation - Actual Fixed Spending, 0)
Known Unpaid = sum(Unpaid Bill Occurrences)
Fixed Reservation = max(Planned Remaining, Known Unpaid)
```

An unpaid bill without a category allocation contributes its full amount. This lets a concrete bill override an insufficient category estimate without double-counting the normal case.

### Goal Reservations

Current goal and sinking-fund reservations are the non-negative sum of their entry histories. Remaining planned reserved contributions are the active plan's reserved allocations less qualifying goal and investment/retirement contributions recorded this month, floored at zero per category.

If a sinking-fund expense reduces both cash and the fund's reserved balance by the same amount, available-to-spend is unchanged by that portion: reserved money was used for its intended purpose.

A withdrawal may exceed the fund's balance after explicit user confirmation. The fund then displays a negative `overdrawn` balance. The reservation used in available-to-spend is floored at zero, so only the portion that was actually reserved offsets the cash expense; the excess reduces available-to-spend normally.

Spending from a goal/sinking fund is one atomic action that records the purchase/account effect and the linked withdrawal entry. The purchase may use the goal's reserved category but does not count as a contribution.

### Actual Spending and Remaining Plan

Actual spending is the sum of non-voided purchases less refunds in a category during the household-local budget month.

```text
Remaining Category Plan = Planned Allocation - Actual Spending
```

Remaining category plan may be negative. Calculations that represent a future reservation use `max(Remaining Category Plan, 0)`; reporting preserves the negative value to show overspending.

## 2. Budget Sections

### Fixed Expenses

Fixed expenses are consistent, committed, or required payments.

Examples:

- Mortgage or rent.
- Car loan.
- Insurance.
- Phone.
- Internet.
- Subscriptions.
- Minimum debt payments.

Rules:

- Fixed expenses are locked in the planning simulator.
- Fixed expenses reduce available-to-spend.
- Subscriptions are fixed expenses, but the planner allows a per-subscription cancellation date to test cancellation impact.

### Flexible Spending

Flexible spending contains categories the household can adjust.

Examples:

- Groceries.
- Dining out.
- Entertainment.
- Clothing.
- Household items.
- Gas.

Rules:

- Flexible categories use sliders plus exact currency inputs in planning mode.
- Purchases reduce the category immediately.
- Categories may go over budget.
- Over-budget categories should be visible and included in on-track status.

### Savings and Sinking Funds

Savings goals and sinking funds are virtually reserved money for future use.

Examples:

- Emergency fund.
- Vacation.
- Christmas.
- Home maintenance.
- Car repairs.
- Kids' activities.
- New vehicle.

Rules:

- Sinking funds reduce available-to-spend.
- They are virtually reserved, not necessarily tied to a separate bank account.
- They are adjustable in planning scenarios.
- They are not locked like fixed expenses.

## 3. Income

### Expected Monthly Income

The app should support expected monthly income for recurring income sources.

Examples:

- Salary 1.
- Salary 2.
- Other recurring income.

### One-Off Income

The app should support one-off income events.

Examples:

- Side jobs.
- Tax refunds.
- Bonuses.
- Gifts.
- Selling items.

Rules:

- Received one-off income increases projected liquid cash when an account is recorded.
- Expected one-off income increases `Expected Income Remaining` only when it is part of the active plan. Scenario-only income affects that scenario only.
- Merely entering a possible windfall outside the active plan does not increase available-to-spend.
- The app should not automatically allocate windfalls.
- Users can allocate one-off income manually when ready.

## 4. Purchases

Rules:

- V1 uses one category per purchase.
- Required fields: amount and category.
- Optional fields: account/card used and notes.
- Purchases update budget progress immediately.
- Purchases may make a category over budget.
- Purchases may make available-to-spend negative.
- A purchase with an account/card creates an account posting after the latest balance observation; a purchase without one still affects the budget but makes the account projection incomplete by that amount.
- Purchases cannot be future-dated. Today's posting uses the current instant; a backdated date without a known time uses the start of that household-local day so later observations absorb it predictably.

## 5. Credit Cards

Rules:

- Credit card purchases affect budgets immediately.
- A credit card purchase increases the credit card liability.
- Paying the credit card is a transfer from cash to liability reduction.
- Credit card payments should not count as spending a second time.
- Account postings are signed changes in net position: a card purchase is negative; a card payment is negative to cash and positive to the card liability, for a net-zero transfer.

## 6. Bills

Bills are first-class records.

Bills should support:

- Name.
- Amount or estimate.
- Due date.
- Paid/unpaid status.
- Recurring schedule.
- Account paid from, when known.

Rules:

- Bills should appear clearly on the dashboard.
- No alerts or notifications are required for V1.
- Users should be able to mark bills as paid quickly, especially on mobile.
- Marking a bill paid creates or links a single spending transaction so the paid status and budget actual cannot drift apart.
- An unpaid bill refines its fixed-category reservation; it is not added a second time on top of the full fixed allocation.
- The mark-paid form defaults to the occurrence estimate but accepts the actual paid amount.
- When actual differs from estimate, the occurrence and linked transaction use actual. The recurring template never changes automatically.
- An unchecked `Use this amount as the new future default` control may update the template and future unpaid, unmodified occurrences explicitly.

### Recurrence

- Recurrence uses an anchor date plus an integer interval in days, weeks, months, or years; arbitrary cron/RRULE input is not accepted in V1.
- Quarterly is every 3 months, semiannual every 6 months, biweekly every 2 weeks, and annual every 1 year.
- A requested day 29–31 falls on the last valid day in a shorter month, then returns to the intended day in later months.
- Semimonthly income is represented by two monthly schedules under one income source.
- Generators materialize a rolling 90-day horizon, preserve copied occurrence details, and are idempotent by schedule/template plus date.
- Template changes affect only future unpaid/expected occurrences that have not been manually modified.

## 7. Goals

Goals should support:

- Current balance.
- Target amount.
- Monthly contribution.
- Optional target date.
- Estimated completion date.

Rules:

- If a target date exists, the app calculates the monthly amount needed.
- If a monthly contribution exists, the app calculates the estimated completion date.
- Goals can be adjusted in planning scenarios.
- A withdrawal that would make the balance negative requires explicit confirmation but is allowed.
- Negative goal balances are displayed as overdrawn and are never treated as negative reserved cash.
- Every V1 goal and sinking fund is a genuine virtual cash reserve. Informational/non-reserving targets are not modeled in V1.
- A goal is not tied to one bank account; aggregate household liquid cash backs aggregate virtual reserves.

### Investment and retirement contributions

- A planned brokerage or retirement contribution uses a `reserved` budget category linked to the destination investment account.
- Executing it records a transfer from a liquid asset to the non-liquid investment/retirement asset. Its postings sum to zero, so it is not spending and does not reduce net worth.
- The transfer references that reserved category solely to fulfill its monthly allocation. It appears in reserved-section progress, not fixed/flexible spending reports.
- Before execution, the remaining reserved allocation reduces available-to-spend. After execution, liquid cash and remaining allocation both fall by the same amount, avoiding a second reduction.
- A transfer linked to a goal entry is counted once through the goal contribution, never once as a goal entry and again as a categorized transfer.

### Goal forecasts

- `Shortfall = max(Target - Current Balance, 0)`.
- Required monthly contribution uses ceiling division across remaining monthly contribution opportunities so the target is not underfunded by rounding.
- The current month counts as an opportunity only when its reserved contribution is not yet fully recorded; the target month counts.
- Estimated completion uses ceiling division of shortfall by configured monthly contribution and returns the final day of the resulting household-local month.
- A completed goal reports its actual completion state immediately.
- A zero contribution with positive shortfall has no completion date.
- A target date with no remaining opportunities reports `target_date_passed`; it never divides by zero or invents a contribution.

## 8. Debt

Debt should support:

- Current balance.
- Interest rate.
- Minimum payment.
- Current monthly payment.
- Estimated payoff date.

Future debt planning should compare:

- Highest interest first.
- Smallest balance first.
- Custom priority order.

### Amortizing debt payments in V1

- Mortgage, auto-loan, and student-loan payments are fixed-category cash outflows.
- If an exact principal/interest split is unavailable, the cash-side posting is recorded but no guessed liability posting is created.
- The liability balance is updated by the next manual observation; the UI labels its freshness honestly.
- Credit-card payments remain exact transfers because the full payment reduces that short-term liability.
- Principal/interest transaction splitting is deferred and must not be approximated from APR alone.

### Debt payoff estimate

- Use the latest projected liability balance and `max(minimum payment, planned payment)`.
- Simulate one payment per month using `monthly interest = round_half_up(balance_minor * apr_micros / 1,200,000,000)`, equivalent to balance × APR-percent ÷ 1200.
- Apply interest, then payment; the final payment is capped at balance plus interest.
- If the payment does not exceed the first month's interest, report `not_amortizing` and no payoff date.
- APR zero uses the same simulation without interest.
- Stop after 1,200 months and report `beyond_projection_horizon` rather than looping indefinitely.
- The estimate assumes no new charges, fees, rate changes, or extra payments and displays those assumptions.

## 9. Scenario Planning

Rules:

- Scenario changes are temporary until explicitly adopted.
- Scenarios may be saved.
- Planning mode may show negative available-to-spend.
- Negative values are valid because they show the gap needed to make a plan feasible.
- Users may adjust flexible categories, sinking fund contributions, expected one-off income, and subscription status.
- Fixed expenses are locked except for subscription on/off analysis.

### Period preparation

- On the 25th, the scheduler idempotently creates the next calendar-month period if absent.
- Its initial active revision copies current flexible and reserved allocations.
- Each fixed category starts at `max(current active allocation, next-month scheduled bill total)`.
- Planned income items come from next-month expected income occurrences.
- The copy may be overallocated; preparation never hides an infeasible result.
- Later bill/income changes do not silently rewrite the adopted revision; normal reservation/remaining-income calculations use current concrete occurrences and the dashboard explains differences.
- Month-end rollovers or an adopted scenario create later active revisions.

New scenario bases are explicit: current active revision, previous-month actuals for flexible categories, or blank flexible/reserved allocations. Fixed obligations are derived/locked in every base.

### Subscription scenario rules

- A cancellation toggle is scoped to one bill template, not its entire fixed category.
- It records an effective cancellation date in the scenario.
- The scenario excludes unpaid occurrences due on or after that date from its fixed reservation and projected cash flow.
- Paid occurrences and occurrences before the date remain unchanged.
- The scenario engine derives the affected fixed-category adjustment from excluded occurrences; users do not directly edit the locked fixed allocation.
- Savings over the next 12 months are the sum of scheduled excluded occurrences, including quarterly or annual recurrence. `monthly equivalent` is that 12-month amount divided by 12 using the standard money rounding rule.
- Changing a scenario never disables the real bill template or changes actual occurrences.
- Adopting a scenario copies each cancellation assumption into the active revision and creates a pending action; adoption does not claim the external service was cancelled.
- While pending, the known unpaid occurrence remains in fixed reservation and the dashboard shows the adopted plan is not fully enacted.
- Confirming external cancellation records the actual effective date, ends the template, cancels future unpaid occurrences on/after that date, and completes the action atomically.
- Choosing to retain the subscription dismisses the action and creates a new active plan revision that removes the cancellation assumption and restores the derived fixed amount.

## 10. Month-End Review

At the end of a month, leftover money should not be automatically allocated.

The user chooses what to do with remaining funds.

Options are:

- Leave as cash.
- Allocate to a savings goal.
- Make an extra debt payment.
- Invest.
- Roll into one or more budget categories.

### Actual month-end free cash

At review, unused plan amounts are historical variances, not future obligations. Allocatable leftover is:

```text
Actual Month-End Free Cash
  = Current Projected Liquid Cash
  - Short-Term Liability Reservation
  - Known Unpaid Fixed Bills
  - Current Goal/Sinking-Fund Reservations
  - Minimum Cash Buffer
```

Expected income remaining is zero for the closed period. Unused fixed envelopes without a known unpaid bill, unused flexible allocations, and unfulfilled reserved contributions expire unless the member explicitly reallocates actual free cash. Positive allocatable leftover is `max(Actual Month-End Free Cash, 0)`.

If positive free cash exists, month-end decisions must account for all of it before close; an explicit `leave as cash` amount covers any retained portion. If free cash is zero or negative, the member acknowledges that state and no positive allocation is created.

Decision effects:

- `Leave as cash` records the choice and makes no financial entry.
- `Goal` creates the explicitly approved virtual goal contribution.
- `Debt` records a payment that the member confirms occurred, including its cash account; the app never initiates the external payment.
- `Invest` records a transfer that the member confirms occurred from cash to the investment account.
- `Category rollover` revises the next plan and moves no cash.

Close revalidates the pre-decision free-cash snapshot, all linked entries/transactions, and the complete allocation total in one transaction. If balances changed during review, close fails with a refreshed review rather than silently adjusting a decision.

### Category rollover

- Category rollover is off by default and is never automatic.
- During month-end review, a member may explicitly choose a source category, target category in the next month, and amount.
- The amount cannot exceed the source category's positive unused allocation.
- Total month-end allocations, including rollover, cannot exceed positive actual month-end free cash. An infeasible future plan may still be created later through normal planning, but month-end review does not label nonexistent leftover cash as rollover.
- A rollover records a month-end decision and increases the target allocation in a new revision of the next month's active plan.
- Rollover does not create an account transaction, goal entry, or movement of cash.
- Unused allocations that are not explicitly rolled over expire with the closed plan.

### Closing and corrections

- A closed month is read-only through normal transaction, budget, bill, income, and goal workflows.
- Either household member may explicitly reopen a closed month.
- Reopening requires a reason and records the actor and time in the audit history.
- Reopening returns the month to `reviewing`, not directly to ordinary `open` status.
- Corrections recalculate the month's dashboard, reports, and review totals from source records.
- The household must review and close the month again after corrections.
- Reopening does not delete the original close or month-end decision history.

## 11. On-Track Status

The dashboard should summarize whether the household is on track.

Headline states, from highest to lowest precedence:

1. Overallocated.
2. Over budget.
3. Attention needed.
4. On track.

Rules:

- **Overallocated:** projected month-end free cash is negative.
- **Over budget:** projected month-end free cash is non-negative, but actual spending exceeds the active allocation in one or more categories.
- **Attention needed:** neither higher state applies, but at least one warning reason exists: an overdue bill, a relevant missing/stale balance, an amortizing debt payment awaiting a new balance observation, known unpaid bills above the remaining fixed plan, an unresolved adopted-plan action, or flexible spending pace projected to exceed its allocation.
- **On track:** none of the conditions above applies.
- The app evaluates every reason even after determining the headline. A month may lead with `overallocated` while also showing category, bill, and freshness reasons.
- Headline copy is derived from stable reason codes and totals, not stored as mutable state.

Default attention thresholds:

- Checking, savings, cash, and cash-reserved credit-card balances become stale after 7 full days without an observation.
- Other enabled account balances become stale after 30 full days.
- Spending-pace projection begins on household-local day 7 of the month.
- For a flexible category with a positive allocation:

```text
Projected Category Spending
  = round(Actual Spending / Elapsed Days * Days In Month)

Projection Tolerance
  = max($25.00, round(Allocation * 5%))
```

- `category_projected_over_budget` applies when projected spending exceeds allocation by at least the projection tolerance.
- Before day 7, actual category progress remains visible but pace alone does not trigger attention.
- These are household settings initialized to 7 days, 30 days, day 7, $25.00, and 5%; changing them is audited.

The app should show details every time, but lead with the highest-precedence summary.

Examples:

- We are on track this month.
- Dining out is projected to exceed budget.
- Available to spend is negative.
- Upcoming bills exceed unreserved cash.

## 12. Recommendations

Recommendations should be balanced by default.

Rules:

- The app suggests actions but never forces them.
- Recommendations should explain trade-offs.
- Recommendations may include reducing flexible spending, pausing goals, canceling subscriptions, adding income, or accepting a lower month-end balance.
- The app should not automatically reallocate money without user approval.

## 13. Historical Data

Rules:

- The current year should remain active and easy to query.
- Prior years remain queryable in the operational database for V1.
- Yearly export and restorable backups are required before any future archival removes operational detail.
- Year-over-year summaries should remain available.
