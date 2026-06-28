# Product Requirements Document

## 1. Overview

This application is a self-hosted household finance planning tool for a two-person household. It provides a shared view of debt, income, account balances, monthly bills, budgets, savings goals, and financial progress.

The product should help answer:

> Are we on track this month?

and, when the answer is no:

> What can we change?

## 2. Users

### Household Member

Both users have the same permissions.

Users can:

- View all household financial data.
- Add purchases.
- Mark bills as paid.
- Update account balances.
- Manage budget categories.
- Create and edit budget scenarios.
- View dashboard, goals, debt, and reports.

## 3. Core User Goals

The app should help users:

- See the full household financial picture in one place.
- Understand the current month at a glance.
- Track budget progress.
- See upcoming bills.
- Know available-to-spend.
- Track all account types, including cash, credit, debt, retirement, brokerage, HSA/FSA, mortgage, auto loans, student loans, and cash on hand.
- Plan future spending using scenarios.
- Understand the impact of subscriptions, goals, debts, and one-off income.

## 4. Functional Requirements

### 4.1 Dashboard

The dashboard must show a summary status and detailed financial sections.

Required dashboard elements:

- Month status: on track, attention needed, over budget, or overallocated.
- Available to spend.
- Monthly income expected vs actual/entered.
- Budget progress.
- Upcoming bills.
- Account balances.
- Goal progress.
- Debt overview.
- Net worth.

The dashboard should be visual and glanceable, not only tabular.

### 4.2 Accounts

The app must support tracking account balances manually.

Account types should include:

- Checking.
- Savings.
- Credit card.
- Retirement.
- Brokerage/investment.
- HSA/FSA.
- Mortgage.
- Auto loan.
- Student loan.
- Cash on hand.
- Other asset or liability.

Requirements:

- Users can periodically update balances.
- Net worth is calculated from assets minus liabilities.
- Exact reconciliation is not required for V1.

### 4.3 Income

The app must support:

- Expected monthly income.
- One-off income.

One-off income should increase available cash without automatic allocation.

### 4.4 Budget

The monthly budget must include three sections:

- Fixed expenses.
- Flexible spending.
- Savings and sinking funds.

Requirements:

- Fixed expenses are locked in planning.
- Flexible categories are adjustable.
- Sinking funds reserve cash but remain adjustable.
- Users can add, rename, disable, and reorganize categories.
- The system should provide a sensible default category set.

### 4.5 Purchases

The app must provide a quick purchase entry flow.

Required fields:

- Amount.
- Category.

Optional fields:

- Account/card used.
- Notes.

Requirements:

- Purchases update budgets immediately.
- One purchase maps to one category in V1.
- Mobile quick-add must be low-friction.

### 4.6 Bills

The app must track bills as first-class records.

Bill fields should include:

- Name.
- Amount or estimate.
- Due date.
- Paid/unpaid status.
- Recurring schedule.
- Account paid from, when known.

Requirements:

- Bills appear on the dashboard.
- Users can mark bills as paid quickly.
- No alerts are required for V1.

### 4.7 Scenario Planner

The app must support temporary and saved budget scenarios.

Users should be able to:

- Start from previous-month spending.
- Adjust flexible categories with sliders.
- Adjust sinking fund contributions.
- Add one-off income.
- Toggle subscriptions on/off.
- See projected cash flow.
- See whether available-to-spend becomes negative.
- Save a scenario.
- Adopt a scenario as the active budget.

Scenario changes must not affect the active budget until explicitly adopted.

### 4.8 Subscriptions

Subscriptions are fixed expenses but require special planning behavior.

Requirements:

- Users can toggle subscriptions off in scenarios.
- The app shows monthly and annual savings impact.
- The app may suggest where freed-up money could go.

### 4.9 Goals and Sinking Funds

The app must support savings goals and irregular expense funds.

Examples:

- Emergency fund.
- Vacation.
- Christmas.
- Home maintenance.
- Car repairs.
- Kids' activities.
- New vehicle.

Requirements:

- Show current progress.
- Show target amount.
- Show monthly contribution.
- Show estimated completion date.
- Calculate required monthly contribution when a target date exists.

### 4.10 Debt Planner

The app must track debts and prepare for future payoff comparisons.

Debt fields should include:

- Current balance.
- Interest rate.
- Minimum payment.
- Current payment.
- Estimated payoff date.

Future requirements:

- Compare highest-interest-first strategy.
- Compare smallest-balance-first strategy.
- Support custom priority order.

### 4.11 Month-End Review

At month end, users should choose what to do with leftover money.

Options should include:

- Leave as cash.
- Allocate to a goal.
- Pay debt.
- Invest.
- Roll into a category.

The app must not automatically allocate leftover funds.

### 4.12 Insights

Insights are optional for V1 but should be considered in the architecture.

Potential future insights:

- Spending increases by category.
- Projected overspending.
- Goal completion changes.
- Subscription cancellation impact.
- Debt payoff impact.

Any AI-generated insights must be local and advisory only.

## 5. Non-Functional Requirements

### Privacy

All financial data must remain local unless the user explicitly enables an integration later.

### Cost

The app should not require paid services for core functionality.

### Usability

Desktop should support setup, review, planning, reports, and management.

Mobile should prioritize:

- Add purchase.
- Mark bill paid.
- View upcoming bills.
- Quick financial snapshot.

### Reliability

Core financial calculations should be deterministic, explainable, and testable.

### Extensibility

The app should be easy to enhance during development but should not be over-engineered into a generic platform.

## 6. Out of Scope for V1

The following are intentionally deferred:

- Automatic bank syncing.
- CSV imports.
- Receipt OCR.
- Native mobile apps.
- Alerts and reminders.
- Multi-household SaaS support.
- Detailed reconciliation workflows.
- Transaction splitting.
- AI chat with financial data.
- Non-financial household records such as warranties or maintenance logs.