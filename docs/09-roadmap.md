# Delivery Roadmap

## Planning Approach

Build a thin, complete monthly-finance loop before adding breadth. Each milestone should leave the application in a demonstrable and tested state. Do not start bank sync, AI, or advanced debt optimization before the manual monthly workflow is trustworthy.

## Milestone 0: Architecture and Calculation Contract

Outcome: the team can implement without redefining core financial meaning.

**Status: complete.** The interview decisions, formulas, acceptance examples, UI contract, schema, agent guide, and deployment runbook are documented.

- Approve the domain model, schema, Laravel principles, and ADRs.
- Validate available-to-spend and projected-month-end formulas with several real anonymized household examples.
- Confirm setup behavior for the user-selected minimum cash buffer and documented stale-balance defaults.
- Sketch the primary mobile and desktop workflows in `07-ui-ux.md`.
- Create a calculation acceptance-test table before application scaffolding.

Exit criteria:

- Every dashboard total has one source and a written formula.
- Credit-card purchase/payment and sinking-fund use cases reconcile conceptually.
- No unresolved question changes the shape of the first migrations.

## Milestone 1: Application Foundation

Outcome: two household members can securely access a local application.

**Status: next.**

- Scaffold Laravel 13 with the official Livewire starter kit.
- Configure SQLite, database queue, private storage, scheduler, code style, and test runner.
- Configure the Mac mini household runtime for same-host and private-LAN access, including automatic process startup and the macOS firewall.
- Implement loopback-only first-run household bootstrap and a 30-minute one-time second-member invitation with QR and fallback URL/code.
- Disable open registration after bootstrap.
- Establish household middleware, policies, shared enums/value objects, clock, and audit mechanism.
- Add CI for Laravel Pint, Larastan/PHPStan level 6, and Pest tests.
- Add health check and documented local backup/restore procedure.

Exit criteria:

- Both members can sign in and have equal access.
- A used, expired, revoked, replayed, or third-member invitation is rejected.
- An unauthenticated or non-member request cannot read household data.
- A database can be backed up and restored in a test environment.
- The app remains healthy after a Mac restart and is reachable from one phone/computer on the trusted LAN without public exposure.

## Milestone 2: Accounts and Current Position

Outcome: the dashboard can show an honest current financial position.

- Implement accounts, types/classification, liquidity, and enable/disable behavior.
- Implement append-only balance observations.
- Implement projected asset/liability positions and freshness states.
- Show liquid cash, accounts, liabilities, and net worth with `as of` indicators.
- Seed or guide sensible account setup.

Exit criteria:

- Assets and liabilities produce correct net-worth signs.
- Missing and stale balances are visible rather than treated as current zeros.
- Recording a new observation establishes a new projection baseline.

## Milestone 3: Transactions and Quick Purchase

Outcome: a purchase can be recorded in seconds and changes the correct budget/account projections.

- Implement fixed/flexible/reserved categories and defaults.
- Implement purchase, refund, income, and transfer actions with postings.
- Build the mobile quick-purchase sheet.
- Implement void-and-replace corrections and change logging.
- Add monthly transaction list/filtering.

Exit criteria:

- Cash and credit-card purchases have correct posting direction.
- Credit-card payment changes accounts but not spending.
- A purchase without an account still changes category actuals and is visibly unreconciled to an account.
- Duplicate submissions do not create duplicate transactions.

## Milestone 4: Active Monthly Budget and Dashboard

Outcome: the household can answer “Are we on track this month?”

- Implement periods, active plan revisions, allocations, and planned income.
- Implement budget progress by section/category.
- Implement available-to-spend, projected month-end free cash, and deterministic status reason codes.
- Assemble the dashboard read model and glanceable UI.
- Implement copy-forward from an explicit prior plan.

Exit criteria:

- The dashboard explains every total through a drill-down/breakdown.
- Overspending and negative totals remain visible.
- Calculations match the acceptance examples.
- Both household members see committed updates consistently.

## Milestone 5: Bills, Subscriptions, and Income Schedule

Outcome: known upcoming obligations and expected income refine the monthly picture.

- Implement recurring bill and income definitions.
- Generate concrete occurrences idempotently for a rolling horizon.
- Build upcoming/overdue bills and quick mark-paid flow.
- Link payment/receipt transactions in the same database transaction.
- Add subscription classification and annualized cost display.

Exit criteria:

- Occurrence generation is safe to retry and handles month-end dates.
- A paid bill cannot be counted twice.
- Fixed reservation uses the greater of plan remaining and known unpaid by category.
- The scheduler can be run repeatedly without changing correct results.

## Milestone 6: Goals and Sinking Funds

Outcome: virtual reserves are trustworthy and their effect on available cash is clear.

- Implement goals, goal entries, target dates, and contribution plans.
- Add contribution, withdrawal/use, and correction workflows.
- Show progress, required monthly amount, and estimated completion.
- Integrate current reservations and remaining contributions into dashboard calculations.

Exit criteria:

- Goal balances can be reconstructed from entries.
- Spending from a sinking fund reduces cash and reserve without double impact.
- Forecast rounding and impossible target states are explained.

## Milestone 7: Scenario Planner

Outcome: the household can safely test changes and adopt a plan.

- Create scenarios from an explicit base revision.
- Add flexible and goal sliders, one-off planned income, and subscription toggles.
- Show active-versus-scenario differences and projected month-end result.
- Implement optimistic concurrency and atomic adoption.
- Preserve adopted/superseded plan history.

Exit criteria:

- Scenario edits never affect the dashboard before adoption.
- Two simultaneous edits cannot silently overwrite each other.
- Adoption changes all plan components or none.
- Negative scenarios remain savable and visibly infeasible.

## Milestone 8: Month-End Review

Outcome: each month can be consciously completed without automatic allocation.

- Summarize actual versus plan, income, bills, and remaining free cash.
- Record explicit leave-as-cash, goal, debt, invest, and rollover decisions.
- Keep rollover off by default; explicit rollover is capped by unused source allocation and positive actual free cash, then revises the next month's plan without moving cash.
- Support review and close state transitions; closed months are read-only and may be explicitly reopened to review with an audited reason.
- Create the next period from the chosen plan/template.

Exit criteria:

- Closing never moves money implicitly.
- Decisions are auditable and linked to actual transactions when executed.
- Closed-month reporting remains stable; corrections require reopening, recalculation, and closing again.

## Milestone 9: Debt Planning and Reports

Outcome: debt and longer-term progress can be compared without weakening the monthly core.

- Add debt metadata and deterministic payoff forecasts.
- Compare avalanche, snowball, and custom strategies using saved assumptions.
- Add net-worth, spending, income, goal, and category trend reports.
- Add yearly export and restore-tested backup workflow.

Exit criteria:

- Payoff results state their assumptions and rounding.
- Reports reconcile to the same underlying transactions/snapshots as the dashboard.
- Export and restore do not require a paid service.

## Later, Only After Evidence

- CSV imports with mapping, preview, duplicate detection, and rollback.
- Optional bank/card connectors behind import adapters.
- Local advisory AI consuming prepared deterministic summaries.
- Opt-in local notifications.
- PostgreSQL or Redis if measured concurrency/performance requires them.
- Multi-currency support after exchange-rate rules are designed.

## Recommended First Vertical Slice

After scaffolding, implement one end-to-end path before broad CRUD:

1. Create a checking account and record a balance observation.
2. Create a flexible groceries category and current period allocation.
3. Record a groceries purchase against checking.
4. Project the checking balance.
5. Recalculate category progress, available-to-spend, projected month-end free cash, and status.
6. Display the result and explanation on the dashboard.

This slice proves the central data flow, money representation, authorization, actions, queries, calculation separation, Livewire interaction, and testing strategy.
