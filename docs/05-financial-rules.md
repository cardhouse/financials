# Financial Rules

This document defines the business rules for the finance engine.

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

Conceptually:

```text
Available to Spend = Total Cash - Reserved Cash - Remaining Required Obligations
```

Available to spend may be negative.

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
- Subscriptions are fixed expenses, but the planner may allow enable/disable toggles to test cancellation impact.

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

- Flexible categories may use sliders in planning mode.
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

- One-off income increases available cash.
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

## 5. Credit Cards

Rules:

- Credit card purchases affect budgets immediately.
- A credit card purchase increases the credit card liability.
- Paying the credit card is a transfer from cash to liability reduction.
- Credit card payments should not count as spending a second time.

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

## 7. Goals

Goals should support:

- Current balance.
- Target amount.
- Monthly contribution.
- Optional target date.
- Estimated completion date.

Rules:

- If a target date exists, the app may calculate the monthly amount needed.
- If a monthly contribution exists, the app may calculate the estimated completion date.
- Goals can be adjusted in planning scenarios.

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

## 9. Scenario Planning

Rules:

- Scenario changes are temporary until explicitly adopted.
- Scenarios may be saved.
- Planning mode may show negative available-to-spend.
- Negative values are valid because they show the gap needed to make a plan feasible.
- Users may adjust flexible categories, sinking fund contributions, expected one-off income, and subscription status.
- Fixed expenses are locked except for subscription on/off analysis.

## 10. Month-End Review

At the end of a month, leftover money should not be automatically allocated.

The user chooses what to do with remaining funds.

Options may include:

- Leave as cash.
- Allocate to a savings goal.
- Make an extra debt payment.
- Invest.
- Roll into one or more budget categories.

## 11. On-Track Status

The dashboard should summarize whether the household is on track.

Potential states:

- On track.
- Attention needed.
- Over budget.
- Overallocated.

The app should show details every time, but lead with a clear summary.

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
- Prior years may be archived into yearly snapshots.
- Year-over-year summaries should remain available.
- Archived details should be restorable if needed.