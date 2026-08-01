# Product Principles

## 1. Local ownership first

All financial data should remain local and under household control. The system must not depend on paid SaaS products or third-party financial services for core functionality.

Future integrations may be optional, but the application should remain useful without them.

## 2. Show reality

The app should reflect the household's actual financial situation, even when that situation is uncomfortable.

It should allow over-budget categories, negative available-to-spend values, and infeasible planning scenarios. These states are useful signals, not errors to hide.

## 3. Suggest, never force

The app may offer recommendations, but it should not automatically move money, change budgets, or make financial decisions for the household.

Users stay in control.

## 4. Prioritize available-to-spend

Account balance alone is not enough. The most useful number is how much money can safely be spent after accounting for fixed expenses, upcoming bills, reserved goals, and other obligations.

Available-to-spend should be a primary dashboard metric.

## 5. Planning matters as much as tracking

The app should not only answer where money went. It should help answer what happens if the household changes something.

Scenario planning is a core feature, not an advanced add-on.

## 6. Keep maintenance low-friction

Manual entry is acceptable, but it must be fast.

Common mobile tasks should take only a few seconds, especially:

- Add a purchase.
- Mark a bill as paid.
- Check whether the month is on track.

## 7. Use visuals to improve understanding

The dashboard should provide visual summaries, progress indicators, charts, and clear status states. The user should not need to scan dense tables to know whether the month is healthy.

## 8. Separate fixed, flexible, and reserved money

The system should distinguish between:

- Locked fixed expenses.
- Adjustable flexible spending.
- Adjustable reserved savings and sinking funds.

These behave differently in planning and reporting.

## 9. Design for a household, not a marketplace

The initial product is for one household with two equal-permission users. It does not need multi-tenant SaaS complexity.

If someone else wants to use it, they can run their own instance.

## 10. Build a finished product, not an endless framework

The architecture should be clean and extensible, but the goal is eventually to be done. Avoid turning the project into a generic platform unless a real household need requires it.

## 11. Deterministic core, optional AI

Financial calculations must be deterministic, explainable, and testable.

A local LLM may generate insights later, but it should consume calculated data rather than own the financial logic.

## 12. Decisions should be documented

Major product, financial, and technical decisions should be captured in ADRs so the project does not rely on chat history or memory.
