# Implementation Guide

## 1. Implementation Status

Product and architecture decisions required to begin V1 are complete. Agents must implement the documented defaults and must not reopen resolved choices. Explicitly deferred features are listed in `10-parking-lot.md` and are not blockers.

If documents appear to conflict, apply this precedence:

1. Most recent applicable ADR in `04-adr.md`.
2. Financial invariants and formulas in `05-financial-rules.md`.
3. Domain invariants in `06-domain-model.md`.
4. Physical schema contract in `11-database-design.md`.
5. Laravel coding contract in `12-laravel-programming-principles.md`.
6. PRD and UI behavior.
7. Roadmap ordering.

Do not silently pick a different rule. If a genuine contradiction remains after applying precedence, document the exact conflict before changing code.

## 2. Fixed Technology Choices

| Concern | Choice |
| --- | --- |
| Runtime | PHP 8.4 |
| Framework | Laravel 13 |
| Frontend | Official Livewire starter kit, Livewire 4, Blade, Flux UI, Tailwind CSS |
| Authentication | Laravel built-in session authentication; no WorkOS, teams, or public registration |
| Database | SQLite with foreign keys enabled |
| Queue | Laravel database queue |
| Session | Laravel database session driver |
| Cache | Laravel database cache for infrastructure; versioned read-model caching only when measured |
| Files | Laravel private local filesystem disks |
| Tests | Pest through `php artisan test` |
| Formatting | Laravel Pint with Laravel preset |
| Static analysis | Larastan/PHPStan at level 6, increased only when the existing codebase passes |
| Development server | Laravel Herd on the Mac mini |
| Household runtime | Native FrankenPHP/Caddy on HTTPS port 8443, separate from Herd's ports |
| JavaScript | Alpine only for ephemeral UI; no SPA or global client store |

## 3. Scaffolding Contract

Because the repository root is non-empty, generate the Laravel application in a temporary sibling directory, then merge the generated application files into this repository. Never initialize/copy a second `.git` directory and never overwrite the planning files. Select:

- Livewire starter kit;
- built-in Laravel authentication;
- no teams;
- Pest;
- SQLite.

Merge procedure:

1. Record `git status` and create a temporary sibling with `mktemp -d`.
2. Run the Laravel installer in that temporary location with the fixed selections.
3. Remove/ignore the generated temporary `.git` metadata if present.
4. Copy generated application files into the repository while preserving the existing `docs/`, `README.md`, `AGENTS.md`, and `.git` exactly.
5. Review the complete diff before deleting the temporary directory.
6. Do not scaffold by deleting or recreating this repository root.

Then:

1. Set the Composer platform/runtime expectation to PHP 8.4.
2. Enable strict types in new domain/application PHP files.
3. Configure `DB_CONNECTION=sqlite`, database queue/session/cache, and private disks.
4. Disable email verification, public registration, forgot-password email routes, and WorkOS.
5. Keep authenticated password change and optional TOTP 2FA/recovery codes.
6. Add loopback-only bootstrap and the second-member invitation flow.
7. Enable Eloquent lazy-loading prevention outside production and model strictness in tests.
8. Add CI commands: `composer validate`, `vendor/bin/pint --test`, static analysis, `php artisan test`.
9. Add `/up` health checks for database writability, queue heartbeat, storage writability, and recent successful backup.

### Account recovery

No email service is required. Recovery paths are:

- A signed-in member changes their own password normally.
- A member with lost TOTP uses a recovery code.
- If locked out, a person with physical Mac mini access runs `php artisan financials:reset-member-password <email>` interactively. The command requires terminal confirmation, sets a one-time password, invalidates sessions and 2FA recovery state as confirmed, logs the recovery, and forces password change at login.
- The command never prints password hashes or accepts a password in shell history.

### Authentication defaults

- Passwords require at least 12 characters, mixed case, a number, and a symbol; confirmation is required on creation/change.
- Do not use a network-based compromised-password check because core authentication must remain local.
- Session inactivity lifetime is 24 hours; persistent `remember me` is disabled in V1.
- Login is limited to 5 attempts per minute by normalized email plus IP.
- Invitation acceptance is limited to 10 attempts per minute per IP in addition to token expiry/single use.
- TOTP 2FA is available but not mandatory; enabling/disabling it requires password confirmation.
- Password, household membership, export/restore, and recovery-sensitive changes invalidate other sessions where applicable.

## 4. Default Seed Data

Bootstrap creates editable categories in this order.

### Fixed

- Housing
- Utilities
- Insurance
- Phone
- Internet
- Subscriptions
- Minimum Debt Payments
- Childcare
- Other Fixed

### Flexible

- Groceries
- Dining Out
- Gas and Transportation
- Household
- Entertainment
- Clothing
- Health and Personal Care
- Kids and Activities
- Gifts
- Miscellaneous

### Reserved

- Emergency Fund
- Home Maintenance
- Car Repairs
- Vacation
- Holidays
- New Vehicle
- Investing
- Other Savings

Bootstrap creates zero-balance sinking-fund goals linked to Emergency Fund, Home Maintenance, Car Repairs, Vacation, Holidays, New Vehicle, and Other Savings. Investing remains disabled until a member selects a non-liquid brokerage/retirement funding account. Users may rename, reorder, or disable seeded records; domain logic depends on section and relationships, never a category name.

### Account defaults

| Kind | Classification | Liquid | Reserve from cash |
| --- | --- | --- | --- |
| checking, savings, cash | asset | yes | no |
| credit card | liability | no | yes |
| retirement, brokerage, HSA/FSA | asset | no | no |
| mortgage, auto loan, student loan | liability | no | no |
| other | member chooses asset/liability and eligible flags explicitly | explicit | explicit if liability |

All account kinds include net worth by default. A member may exclude an account from net worth explicitly. Liability accounts cannot be liquid; asset accounts cannot reserve from cash.

Household preference defaults are USD, `America/New_York`, Sunday week start, 7-day liquid/card freshness, 30-day other freshness, projection from day 7, $25/5% projection tolerance, and zero minimum cash buffer until setup chooses another amount.

## 5. Migration Order

Keep migrations small and ordered by dependency:

1. Laravel auth/session/cache/queue tables.
2. `households`, `users` adjustments, `household_memberships`, `household_invitations`.
3. `accounts`, `account_balance_snapshots`, `debt_profiles`.
4. `budget_categories` without cross-table funding FK if needed to avoid cycles.
5. `financial_transactions`, `account_postings`; import-specific columns are deferred.
6. `budget_periods` without active-plan FK, then `budget_plans`, allocations, income items, subscription overrides/actions, then add active-plan FK.
7. `income_sources`, `income_schedules`, `income_occurrences`.
8. `bill_templates`, `bill_occurrences`.
9. `goals`, `goal_entries`, then add optional category/funding relationships.
10. `month_end_allocations`.
11. `change_log`, `service_heartbeats`, `backup_runs`.

All foreign keys use explicit delete behavior. Financial history normally restricts deletion. Cascade is allowed only for an aggregate that has never been posted/adopted and is being discarded atomically, such as a draft scenario and its child rows.

## 6. Route and Screen Map

| Method/path | Screen/action | Authorization |
| --- | --- | --- |
| `GET /setup` | loopback-only bootstrap | only before household exists |
| `GET/POST /invite/{token}` | second-member acceptance | valid invitation + LAN HTTPS |
| `GET /dashboard?month=YYYY-MM` | household dashboard | member |
| `GET /budget?month=YYYY-MM` | active plan/progress | member |
| `GET /planner?month=YYYY-MM` | scenarios | member |
| `GET /transactions?month=YYYY-MM` | transaction history | member |
| `GET /bills?month=YYYY-MM` | occurrences/templates | member |
| `GET /goals` | goals and sinking funds | member |
| `GET /accounts` | accounts and observations | member |
| `GET /debt` | debt overview/projections | member |
| `GET /reports` | historical summaries | member |
| `GET /review/{period}` | month-end review | member |
| `GET /settings/*` | household/member/preferences | member |

State changes occur through Livewire actions backed by application Action classes. Route names and Livewire component names may follow Laravel starter-kit conventions, but public behavior and authorization are fixed by this map.

## 7. Action Inventory

Implement actions as they become needed; names are authoritative intent:

### Household

- `BootstrapHousehold`
- `CreateHouseholdInvitation`
- `AcceptHouseholdInvitation`
- `RevokeHouseholdInvitation`
- `UpdateHouseholdPreferences`

### Accounts and transactions

- `CreateAccount`
- `UpdateAccount`
- `DisableAccount`
- `RecordAccountBalance`
- `RecordPurchase`
- `RecordIncome`
- `RecordTransfer`
- `RecordReservedTransfer`
- `RecordRefund`
- `VoidTransaction`
- `ReplaceTransaction`

### Budgeting

- `OpenBudgetPeriod`
- `PrepareNextBudgetPeriod`
- `CreateActiveBudgetRevision`
- `CreateBudgetScenario`
- `UpdateScenarioAllocation`
- `AddScenarioIncome`
- `SetScenarioSubscriptionCancellation`
- `AdoptBudgetScenario`
- `CompleteBudgetPlanAction`
- `DismissBudgetPlanAction`

### Bills and income

- `GenerateBillOccurrences`
- `GenerateIncomeOccurrences`
- `MarkBillPaid`
- `SkipBillOccurrence`
- `ReceiveIncomeOccurrence`

### Goals and month end

- `ContributeToGoal`
- `SpendFromGoal`
- `WithdrawFromGoal`
- `CorrectGoalBalance`
- `BeginMonthEndReview`
- `RecordMonthEndAllocation`
- `CloseBudgetPeriod`
- `ReopenBudgetPeriod`

Every action follows `12-laravel-programming-principles.md`: typed input, authorization, short transaction, invariant validation, semantic audit, after-commit event, typed result.

## 8. Query and Calculator Inventory

Queries load and normalize data:

- `GetMonthlyDashboard`
- `GetAvailableToSpendBreakdown`
- `GetBudgetProgress`
- `GetScenarioComparison`
- `GetUpcomingBills`
- `GetAccountPositions`
- `GetGoalForecasts`
- `GetDebtOverview`
- `GetMonthEndReview`
- `GetNetWorthHistory`

Pure calculators perform math:

- `AccountPositionCalculator`
- `FixedReservationCalculator`
- `ReservedContributionCalculator`
- `AvailableToSpendCalculator`
- `BudgetProgressCalculator`
- `ScenarioProjectionCalculator`
- `MonthlyStatusCalculator`
- `GoalForecastCalculator`
- `DebtPayoffCalculator` in its roadmap milestone.

Do not query from calculators or calculate authoritative totals in Livewire/Blade/JavaScript.

## 9. Transaction-Type Matrix

| Type | Category | Postings | Budget effect |
| --- | --- | --- | --- |
| Cash/checking purchase | required fixed/flexible | one negative asset posting | spending actual |
| Credit-card purchase | required fixed/flexible | one negative liability-position posting | spending actual |
| Sinking-fund purchase | matching reserved goal category | purchase posting plus atomically linked negative goal entry | goal use, not contribution actual |
| Refund | required original section | one opposite-direction posting when account known | reduces spending actual |
| Income | none | one positive asset posting when account known | received income |
| Cash transfer | none | two or more postings summing zero | none |
| Credit-card payment | none | negative cash + positive liability posting, sum zero | none |
| Investment/retirement contribution | required reserved category | negative liquid + positive investment asset, sum zero | reserved contribution actual |
| Goal contribution with transfer | category derived from goal, transaction category null | transfer postings sum zero plus linked goal entry | goal contribution actual once |
| Virtual goal contribution | category derived from goal | no posting; goal entry only | goal contribution actual |
| Amortizing loan payment without split | required fixed category | negative cash posting only | fixed spending actual; liability awaits observation |

## 10. Plan Revision Rules

- Adopted active revisions are immutable.
- Editing active monetary allocations creates a new active revision and supersedes the old one atomically.
- Actuals belong to period/category and are compared with whichever revision is selected.
- Reference-data label/order changes do not rewrite plan history.
- Scenario `lock_version` is required on every update.
- Adoption revalidates the base/current version, copies scenario values into a new adopted revision, updates the period pointer, supersedes the previous revision, and creates required pending actions in one transaction.
- `PrepareNextBudgetPeriod` runs on the 25th: copy flexible/reserved allocations, set fixed allocations to the greater of copied plan or scheduled bill total by category, and create income items from expected occurrences. It is idempotent and never rewrites an existing prepared period.
- Scenario creation requires base mode `active`, `previous_actuals`, or `blank`; the chosen mode is persisted/audited.

## 11. Framework Events and Jobs

After-commit events include:

- `FinancialDataChanged`
- `TransactionRecorded`
- `BillPaid`
- `AccountBalanceRecorded`
- `ScenarioAdopted`
- `GoalBalanceChanged`
- `BudgetPeriodClosed`

Listeners may increment the household financial-data version, clear read caches, or enqueue non-critical work. Do not use an event listener for a required sibling write.

V1 queued jobs are limited to export generation, backup verification, and later milestone imports/reports. Jobs use database queue, explicit timeout, at most three attempts unless a job documents another safe policy, and stable idempotency keys.

## 12. Testing Gate

Before merging any vertical slice:

```text
composer validate
vendor/bin/pint --test
vendor/bin/phpstan analyse
php artisan test
```

Tests must include the applicable examples from `14-calculation-acceptance-tests.md`. A failing financial acceptance example is a release blocker and must not be changed merely to match current code.

## 13. Delivery Order

Implement roadmap milestones in order. Within each milestone:

1. Add/update domain documentation only if the implementation reveals a true contradiction.
2. Write calculator/state unit tests.
3. Add migrations, enums, models, factories, policies.
4. Implement actions and query objects with feature tests.
5. Build the smallest Livewire UI using the established design system.
6. Verify responsive/accessibility behavior.
7. Run the full gate and update the implementation checklist.

The first functional vertical slice remains checking account → observation → groceries category/allocation → purchase → account projection → dashboard calculations.

## 14. Definition of Implementation-Ready

An agent may begin without further product questions because:

- V1 scope and deferred scope are explicit.
- Stack and tooling are fixed.
- formulas and acceptance values are fixed;
- entities, schema columns, relationships, and invariants are specified;
- actions, queries, status precedence, UI hierarchy, and deployment target are specified;
- security, onboarding, correction, audit, backup, and concurrency behavior are specified.

Ordinary implementation details—class extraction, component composition, query optimization, naming a private helper—remain agent engineering judgment within these constraints and do not require user clarification.
