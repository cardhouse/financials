# Laravel Programming Principles

## 1. Purpose

These principles translate the product and financial rules into repeatable implementation decisions. They are constraints for code review, not aspirations.

## 2. Prefer Laravel's Defaults

Use Laravel conventions until a domain requirement proves they are insufficient:

- Eloquent models and relationships for persistence.
- Policies for record authorization.
- Livewire forms or Form Requests for boundary validation.
- Backed PHP enums with Eloquent enum casts for controlled states.
- Attribute/custom casts for dates and money-facing values.
- Container injection for actions, queries, calculators, and clocks.
- Database transactions for multi-record state changes.
- Events, queues, scheduler, cache, and filesystem through their Laravel contracts.

Do not add a service, repository, interface, package, or design pattern solely to make the code look architectural. Add a boundary when it isolates a real rule, side effect, or replaceable dependency.

## 3. One Action per User Intent

Every meaningful state change has an invokable application action named as a command, such as `RecordPurchase`, `AdoptBudgetScenario`, or `MarkBillPaid`.

An action should:

1. Accept an authenticated actor plus a typed immutable data object.
2. Authorize the aggregate operation or require authorization at the caller with an explicit contract.
3. Load all records needed to enforce the invariant.
4. Execute the complete write in `DB::transaction()` when more than one row is involved.
5. Write a semantic audit entry.
6. Dispatch follow-up events after commit.
7. Return the changed aggregate or a typed result.

Livewire components, controllers, jobs, and commands call the same actions. They do not duplicate the rules.

## 4. Keep Financial Math Pure

Calculation classes never query Eloquent or reach through a facade. Inputs and outputs are typed DTOs/value objects.

```php
final readonly class AvailableToSpendInput
{
    public function __construct(
        public int $liquidCashMinor,
        public int $remainingIncomeMinor,
        public int $shortTermLiabilityMinor,
        public int $fixedReservationMinor,
        public int $goalReservationMinor,
        public int $remainingReservedContributionsMinor,
        public int $minimumCashBufferMinor,
    ) {}
}
```

A calculator returns a breakdown, not just an integer, so the UI can explain the result and tests can identify the failing component.

Do not:

- perform money arithmetic in JavaScript and persist the answer;
- use floats for money;
- hide financial formulas in Eloquent accessors, Blade templates, SQL views, or chart code;
- use AI output as a numeric input;
- clamp negative results merely to make the UI look healthy.

## 5. Represent Money and Rates Explicitly

- Persist money as integer minor units (`$12.34` is `1234`).
- Use one `Money` value object or equivalent typed amount at domain boundaries; it carries amount and currency even while V1 permits only the household currency.
- Parse display strings once at the input boundary and format only at the output boundary.
- Never accept a PHP float in a financial action.
- Store APR as integer millionths of one percentage point (`6.875%` is `6,875,000`) and never convert it through a float.
- Default derived-money division uses nearest minor unit with half-up rounding. Required contributions use ceiling division so a target is not underfunded. Round only at the formula's specified step, not after every intermediate operation.
- Reject arithmetic across currencies until exchange rates, rate dates, and realized gains are designed.

## 6. Treat Time as a Dependency

- Financial dates are household-local `CarbonImmutable` dates.
- System/audit instants are UTC immutable timestamps.
- Inject a clock into actions/calculators whose result depends on today or now.
- Use `Date::setTestNow()`/Laravel time helpers in tests; never make tests depend on the wall clock.
- Month boundaries come from the household timezone.
- A recurrence scheduled for day 29–31 falls on the month's last valid day; this rule must have explicit tests.

## 7. Make State Machines Explicit

Use backed enums and transition methods/actions for lifecycles:

- budget period: open → reviewing → closed, with closed → reviewing allowed only by the explicit audited reopen action;
- scenario: draft → adopted/superseded or discarded;
- bill occurrence: upcoming → paid/skipped/cancelled;
- income occurrence: expected → received/skipped;
- import batch: uploaded → validating → importing → completed/failed.

Do not assign status strings ad hoc from UI code. Reject illegal transitions with meaningful domain exceptions.

## 8. Use Eloquent Deliberately

Eloquent models may own:

- relationships;
- enum and value casts;
- local query scopes;
- simple predicates such as `isPaid()`;
- local state transitions that cannot violate another aggregate.

Eloquent models should not own:

- dashboard orchestration;
- cross-aggregate writes;
- network/filesystem calls;
- queued side effects;
- large report queries hidden behind an accessor;
- a long list of unrelated static workflow methods.

Avoid accidental lazy loading in development and tests. Queries should eager-load intentionally or select purpose-built projections. Do not solve N+1 issues by globally loading every relationship.

## 9. Separate Commands from Queries Without Ceremony

- Actions change state and return a small result.
- Query objects read state and return immutable view models.
- Calculators transform already-loaded data.

This is a code-organization rule, not a separate CQRS infrastructure. Reads and writes use the same database and Eloquent models.

Complex reports may use the query builder for explicit aggregation. Keep the query in a named query object and test it against realistic database fixtures.

## 10. Transactions Protect Invariants

Use a database transaction when a use case must be all-or-nothing, including:

- a financial transaction plus its account postings;
- marking a bill paid plus creating/linking its transaction;
- receiving expected income plus creating/linking its transaction;
- adopting a scenario plus changing the active-plan pointer;
- a goal contribution plus its supporting transfer when one is recorded;
- closing a month plus its review decisions.

Keep transactions short. Do not perform HTTP calls, file parsing, AI generation, or slow exports inside them. Queue follow-up work after commit.

## 11. Concurrency Must Be Intentional

There are two users even in V1. Assume they can edit the same month simultaneously.

- Use unique constraints for stable facts such as one occurrence per template/due date and one allocation per plan/category.
- Use `lock_version` for interactive scenario editing. A stale form receives a conflict and reload/compare option.
- During adoption/payment/close, re-read the aggregate inside the transaction and verify its current status.
- Use row locks where supported, but do not rely on SQLite row-lock semantics for correctness.
- Commands triggered by retryable requests, jobs, imports, or schedules accept an idempotency key or derive one from the natural unique key.
- Never implement “check then insert” without a supporting unique constraint.

## 12. Validation and Authorization Are Different

Validation answers “is this input shaped correctly?” Authorization answers “may this member perform this operation on this household record?” Domain invariants answer “is this transition financially valid?” Keep all three.

Validation rules cover:

- required fields, length, date shape, integer bounds, and allowed enum values;
- parsing and normalization of money strings;
- basic referenced-record existence scoped to the household.

Policies and aggregate actions cover:

- household membership and same-household ownership;
- whether a closed period may be modified;
- whether the current state permits the operation;
- multi-record invariants such as transfer balance and active-plan uniqueness.

Never trust a hidden `household_id`, `user_id`, amount direction, or status from the browser. Derive ownership and controlled values on the server.

## 13. Prefer Visible Orchestration to Hidden Side Effects

Core correctness does not live in model observers, queued listeners, or broad `saved` hooks. Those mechanisms make financial writes hard to reason about and retry.

Use explicit actions for required changes. Events are appropriate for facts and non-critical reactions such as:

- invalidating a dashboard cache;
- updating a recent-activity feed;
- triggering an optional export/insight;
- later sending an opt-in notification.

Name events in past tense (`BillPaid`, `ScenarioAdopted`) and dispatch them only after the write commits.

## 14. Preserve History; Correct Transparently

- Do not hard-delete posted transactions, goal entries, paid bill occurrences, adopted plans, or balance snapshots through normal UI paths.
- Void and replace incorrect transactions, recording actor, time, and reason.
- `quick_entry_undo` is accepted only for the original actor, purchase type, and within 10 seconds of creation; afterward normal reasoned void rules apply.
- Disable/archive reference data while retaining historical relationships.
- Adopt a new immutable plan revision instead of editing the adopted plan in place.
- Append semantic audit records from actions rather than logging arbitrary model payloads.
- Reject normal writes to a closed period; `ReopenBudgetPeriod` requires a reason and returns it to review before corrections.
- Never store credentials, tokens, password material, raw bank files, or unrelated unchanged attributes in audit JSON.

## 15. Cache Derived Data, Never Financial Truth

The database remains authoritative. Start with uncached indexed queries.

If caching becomes necessary:

- cache immutable dashboard/read-model objects, not Eloquent models;
- include household, period, and a financial-data version in the key;
- increment/invalidate only after a successful relevant commit;
- make a cache miss indistinguishable in correctness from a cache hit;
- never use cache locks as the only enforcement of a financial invariant.

## 16. Queue Only Work That May Finish Later

Appropriate queued work:

- parse a large CSV after upload;
- generate an export or report;
- recompute optional historical projections;
- create an advisory local-AI narrative.

Inappropriate queued work:

- creating the postings for a purchase;
- updating the active plan after adoption;
- marking the bill paid after its transaction is created;
- anything the immediate response claims has already succeeded.

Jobs carry record IDs and idempotency information. Set explicit timeouts, bounded retries, and failure reporting. Re-load current state when the job runs.

## 17. Test Behaviors and Invariants

### Unit tests

Pure calculations and state rules:

- available-to-spend and its breakdown;
- fixed reservation `max(planned remaining, known unpaid)`;
- negative values and overspending;
- asset/liability projection signs;
- credit-card payment nets to zero;
- goal forecasts and rounding;
- confirmed goal overdrafts and reservation flooring;
- recurrence at month/leap-year boundaries;
- monthly status reason codes.

Spending-pace tests use the household-local inclusive day number as elapsed days and the month's actual day count. Pace warnings begin on the configured start day and compare the projected overage with the greater of the configured fixed-minor-unit tolerance or basis-point tolerance. Integer arithmetic and the documented rounding rule are required.

### Feature/integration tests

Laravel behavior using the real schema:

- household scoping and policies;
- transaction/action atomicity;
- scenario isolation, stale version conflict, and adoption;
- bill occurrence idempotency and payment linking;
- manual balance snapshot projection cutoff;
- month-close decisions;
- scheduler and queue dispatch after commit;
- import duplicate detection;
- export authorization and private storage.

### Test design rules

- Use factories with named states (`creditCard`, `liquid`, `paid`, `scenario`).
- Prefer fixed explicit amounts and dates over random values in financial assertions.
- Use table-driven datasets for calculation edges.
- Include at least one end-to-end happy path for the monthly household workflow.
- Run tests against SQLite in normal development; periodically run portable-schema tests against PostgreSQL only if PostgreSQL remains a supported target.
- Do not mock Eloquent internals. Mock only true external boundaries such as a filesystem adapter or future bank connector.

## 18. Use Reason Codes for Explainability

Status and recommendation calculators return stable reason codes with numeric evidence, for example:

- `available_to_spend_negative`;
- `known_bills_exceed_fixed_plan`;
- `category_over_budget`;
- `category_projected_over_budget`;
- `account_balance_stale`;
- `plan_projected_month_end_negative`;
- `debt_balance_awaiting_observation`;
- `adopted_plan_action_pending`.

The UI maps reason codes to human language. A future local LLM may rewrite these facts conversationally but cannot invent or override the status.

The headline uses this fixed precedence: `overallocated`, `over_budget`, `attention_needed`, `on_track`. Calculators still return every applicable reason so a higher-severity headline never hides useful detail.

## 19. Keep Configuration Out of Domain Logic

Environment configuration controls infrastructure: database path, queue connection, filesystem disks, session security, backup destination, and optional integrations.

Household financial preferences belong in persisted settings with validation and audit history: timezone, base currency, stale-balance threshold, desired cash buffer, and status thresholds.

Do not read `env()` outside configuration files or hide household rules in deployment configuration.

## 20. Definition of Done for a Financial Feature

A feature is complete when:

- its terms and invariant are documented;
- authorized happy and failure paths are implemented;
- multi-row writes are atomic and retry-safe where necessary;
- calculation output is explainable;
- history/correction behavior is defined;
- unit and feature tests cover boundary cases;
- the mobile interaction remains usable;
- logs and audit records exclude sensitive payloads;
- no optional service is required for the core path.
