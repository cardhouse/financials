# Database Design

## 1. Database Strategy

V1 uses SQLite as the authoritative local database. The schema uses ordinary relational tables, foreign keys, unique constraints, and portable data types so PostgreSQL remains a deployment option without maintaining two models.

The operational database keeps all years. Yearly export and restore are valuable backup features, but moving prior years out of the database is deferred until measured volume demonstrates a need.

Conventions:

- Primary keys are unsigned big integers.
- Public/import identifiers use nullable ULIDs where stable external identity is needed.
- Every household-owned table has `household_id`, even when ownership is reachable through another table.
- Monetary columns end in `_minor` and store integer minor currency units.
- Calendar-only values use `date`; instants use UTC timestamps.
- Status and type fields use string-backed application enums with database check constraints where practical.
- User-removable reference data uses `disabled_at` or `archived_at`; financial history uses void/replacement semantics instead of destructive deletion.
- JSON is reserved for audit snapshots, import payloads, and calculation assumptions—not core relationships or queryable money.

## 2. Identity and Household

### `users`

Laravel authentication identity.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `name` | varchar | |
| `email` | varchar unique | Normalized by application |
| `password` | varchar | Laravel hash |
| auth fields | standard | remember token, verification, 2FA when enabled |
| timestamps | timestamp | |

### `households`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `name` | varchar | |
| `base_currency` | char(3) | `USD` initially |
| `timezone` | varchar | IANA name, e.g. `America/New_York` |
| `week_starts_on` | tinyint | UI preference |
| `minimum_cash_buffer_minor` | bigint | non-negative; default zero |
| `liquid_balance_stale_after_days` | smallint | default 7 |
| `other_balance_stale_after_days` | smallint | default 30 |
| `spending_projection_start_day` | smallint | default 7 |
| `spending_projection_tolerance_minor` | bigint | default 2500 ($25.00 in USD) |
| `spending_projection_tolerance_bps` | integer | default 500 (5%) |
| `setup_completed_at` | timestamp nullable | authenticated checklist completion |
| `financial_data_version` | bigint | monotonic cache/review invalidation counter |
| timestamps | timestamp | |

### `household_memberships`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `user_id` | FK | |
| `joined_at` | timestamp | |
| `last_used_account_id` | FK nullable | Per-member quick-purchase preference; must be enabled household account |

Unique: `(household_id, user_id)`. V1 has no role column because permissions are equal.

### `household_invitations`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `invited_by_user_id` | FK | |
| `token_hash` | char(64) unique | SHA-256 hash; raw token is never persisted |
| `expires_at` | timestamp | default 30 minutes |
| `accepted_at` | timestamp nullable | |
| `accepted_by_user_id` | FK nullable | |
| `revoked_at` | timestamp nullable | |
| timestamps | timestamp | |

The QR code contains only the canonical HTTPS invitation URL with a cryptographically random 256-bit token. It contains no password, email address, or financial data. Invitation acceptance:

- is available only over the trusted LAN HTTPS origin;
- requires an unexpired, unused, non-revoked token;
- lets the second member enter their own name, email, and password;
- consumes the token atomically while creating the user and membership;
- is rejected once the household has two members;
- is rate-limited and recorded in the audit log.

Only a token hash is stored. The settings screen shows a QR code plus a copyable fallback URL/code and lets either member revoke a pending invitation.

## 3. Accounts and Debt

### `accounts`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | indexed |
| `name` | varchar | |
| `kind` | varchar | checking, savings, credit_card, retirement, brokerage, hsa_fsa, mortgage, auto_loan, student_loan, cash, other |
| `classification` | varchar | asset or liability |
| `currency` | char(3) | Must equal household currency in V1 |
| `is_liquid` | boolean | Defaults true for checking, savings, and cash; false for liabilities/restricted assets |
| `reserve_from_cash` | boolean | Normally true for credit cards; false for assets and long-term debts |
| `include_in_net_worth` | boolean | default true |
| `display_order` | integer | |
| `notes` | text nullable | |
| `disabled_at` | timestamp nullable | |
| timestamps | timestamp | |

Indexes: `(household_id, disabled_at, display_order)`.

### `account_balance_snapshots`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | denormalized ownership guard |
| `account_id` | FK | |
| `balance_minor` | bigint | Positive displayed balance for either asset or liability |
| `effective_at` | timestamp | When the institution/cash balance was true |
| `recorded_by_user_id` | FK | |
| `note` | text nullable | |
| timestamps | timestamp | |

Index: `(account_id, effective_at desc, id desc)`. Multiple observations are preserved.

### `debt_profiles`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `account_id` | FK unique | Must reference a liability |
| `annual_percentage_rate_micros` | bigint | Percent scaled by 1,000,000; `6.875%` is `6,875,000` |
| `minimum_payment_minor` | bigint | non-negative |
| `planned_payment_minor` | bigint | non-negative |
| `payment_day` | tinyint nullable | 1–31; last-day fallback |
| `priority` | integer nullable | custom payoff order |
| timestamps | timestamp | |

Current debt balance is not duplicated here; it comes from the account position.

## 4. Categories and Transactions

### `budget_categories`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `name` | varchar | |
| `section` | varchar | fixed, flexible, or reserved |
| `color` | varchar nullable | validated design token/hex value |
| `icon` | varchar nullable | validated icon name |
| `display_order` | integer | |
| `funding_account_id` | FK nullable | reserved-category destination such as brokerage/retirement |
| `disabled_at` | timestamp nullable | |
| timestamps | timestamp | |

Unique active names are enforced by the application for SQLite portability. Historical transactions may continue referencing a disabled category.

An enabled reserved category has exactly one funding mode: linked by one `goals.budget_category_id` or linked directly to a non-liquid asset `funding_account_id`. It cannot be linked to both. An unconfigured reserved category remains disabled and cannot receive an allocation. Categorized investment transfers require the direct funding account.

### `financial_transactions`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `public_id` | ULID unique | Stable import/export identity |
| `household_id` | FK | indexed with date |
| `type` | varchar | purchase, income, transfer, refund, adjustment |
| `occurred_on` | date | Household-local financial date |
| `amount_minor` | bigint | Positive magnitude; greater than zero |
| `category_id` | FK nullable | Required for purchase/refund; optional reserved-allocation fulfillment for transfer |
| `description` | varchar nullable | merchant/payee/source |
| `notes` | text nullable | |
| `entered_by_user_id` | FK | |
| `idempotency_key` | ULID | client/action retry key |
| `voided_at` | timestamp nullable | |
| `voided_by_user_id` | FK nullable | |
| `void_reason` | text nullable | |
| `replacement_transaction_id` | FK nullable | corrected record |
| timestamps | timestamp | |

Indexes:

- `(household_id, occurred_on, voided_at)` for monthly actuals.
- `(household_id, category_id, occurred_on)` for category progress.
- Unique `(household_id, idempotency_key)` prevents duplicate submissions/retries.

### `account_postings`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `financial_transaction_id` | FK | cascade only while parent is being initially created |
| `account_id` | FK | |
| `net_position_change_minor` | bigint | Signed; asset increase or liability reduction is positive |
| `effective_at` | timestamp | Used relative to balance snapshots |
| timestamps | timestamp | |

Indexes: `(account_id, effective_at, financial_transaction_id)` and `(financial_transaction_id)`.

Posting time rules:

- A transaction recorded for household-local today uses the committed current instant.
- A backdated date with no known time uses the start of that household-local day so a later same-day manual observation normally absorbs it.
- Imports may supply a more precise source timestamp in a later milestone.
- Purchases/income cannot be future-dated in V1; planned future events belong to bill/income occurrences or scenarios.
- Balance observations default to current instant and may be backdated only through the expanded account-history flow with an explicit timestamp.

Rules enforced by application actions:

- Transfer postings sum to zero and include at least two accounts.
- A transfer may reference a category only when that category is `reserved`, is not goal-backed, and the destination matches its configured funding account. Goal-backed transfers keep transaction category null and derive progress from the linked goal entry.
- A purchase may reference a reserved category only through the atomic spend-from-goal workflow and must link to the matching withdrawal entry.
- A purchase/refund/income has zero postings when account is unknown or one expected-direction posting when known.
- All referenced accounts and categories belong to the transaction household.

The posting model avoids counting a credit-card payment as an expense while supporting projected account positions after manual snapshots.

## 5. Monthly Budgeting

### `budget_periods`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `month` | date | Always first day of local month |
| `status` | varchar | open, reviewing, closed |
| `active_budget_plan_id` | FK nullable | Set after initial plan is created |
| `prepared_at` | timestamp nullable | next-period preparation completion |
| `reviewed_at` | timestamp nullable | |
| `closed_at` | timestamp nullable | |
| `reopened_at` | timestamp nullable | Most recent reopen; full history is in `change_log` |
| `reopened_by_user_id` | FK nullable | Most recent actor |
| `review_version` | bigint | captured household financial-data version |
| timestamps | timestamp | |

Unique: `(household_id, month)`. The active pointer makes the single-active-plan rule explicit and avoids relying on a database-specific partial unique index.

### `budget_plans`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `budget_period_id` | FK | |
| `name` | varchar | |
| `kind` | varchar | active_revision or scenario |
| `status` | varchar | draft, adopted, superseded, discarded |
| `based_on_plan_id` | FK nullable | lineage |
| `base_mode` | varchar | active, previous_actuals, blank, prepared_copy |
| `created_by_user_id` | FK | |
| `adopted_at` | timestamp nullable | |
| `superseded_at` | timestamp nullable | |
| `lock_version` | integer | optimistic concurrency for edits |
| timestamps | timestamp | |

Indexes: `(budget_period_id, kind, status)` and `(based_on_plan_id)`.

### `budget_allocations`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `budget_plan_id` | FK | |
| `budget_category_id` | FK | |
| `planned_minor` | bigint | non-negative |
| `notes` | text nullable | |
| timestamps | timestamp | |

Unique: `(budget_plan_id, budget_category_id)`.

Normal fixed allocations remain locked in scenarios. Pausing an optional reserved contribution sets its scenario allocation to zero; subscriptions use per-template overrides rather than a category flag.

### `budget_plan_subscription_overrides`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `budget_plan_id` | FK | Scenario or adopted active revision |
| `bill_template_id` | FK | Must be an active subscription |
| `cancel_effective_on` | date | First excluded due date boundary |
| timestamps | timestamp | |

Unique: `(budget_plan_id, bill_template_id)`. Calculators exclude only unpaid occurrences due on/after `cancel_effective_on`; paid records are untouched. Twelve-month savings sum the actual excluded recurrence schedule rather than multiplying a nominal monthly amount.

### `budget_plan_actions`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `budget_plan_id` | FK | adopted active revision |
| `type` | varchar | cancel_subscription initially |
| `bill_template_id` | FK nullable | required for cancellation |
| `planned_effective_on` | date | from scenario assumption |
| `status` | varchar | pending, completed, dismissed |
| `completed_effective_on` | date nullable | actual confirmed date |
| `resolved_by_user_id` | FK nullable | |
| `resolved_at` | timestamp nullable | |
| timestamps | timestamp | |

Adoption creates pending actions from the active revision's subscription overrides. Pending bill occurrences continue participating in the real fixed reservation. Completing a cancellation action atomically sets the template `ended_on`, cancels future unpaid occurrences on/after the actual date, and resolves the action. Dismissing it creates a new active plan revision without that override before marking the action dismissed.

### `planned_income_items`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `budget_plan_id` | FK | |
| `income_source_id` | FK nullable | null for scenario-only/ad hoc income |
| `income_schedule_id` | FK nullable | scheduled source occurrence |
| `name` | varchar | snapshot label |
| `expected_minor` | bigint | non-negative |
| `expected_on` | date nullable | |
| timestamps | timestamp | |

This table lets a scenario add one-off income without creating real income or changing the active plan.

## 6. Income

### `income_sources`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `name` | varchar | salary, side work, etc. |
| `disabled_at` | timestamp nullable | |
| timestamps | timestamp | |

### `income_schedules`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `income_source_id` | FK | |
| `default_amount_minor` | bigint | expected amount per occurrence |
| `recurrence_unit` | varchar | day, week, month, year |
| `recurrence_interval` | integer | positive; biweekly is week/2 |
| `anchor_on` | date | recurrence anchor |
| `intended_day_of_month` | tinyint nullable | 1–31 for month/year recurrence |
| `ends_on` | date nullable | inclusive final schedule date |
| `disabled_at` | timestamp nullable | |
| timestamps | timestamp | |

Semimonthly income uses two monthly schedules under one source, for example day 15 and day 31 (last-day fallback). Irregular income has no schedule and is entered as an occurrence or plan item directly.

### `income_occurrences`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `income_source_id` | FK nullable | one-off income may be direct |
| `income_schedule_id` | FK nullable | null for one-off income |
| `name` | varchar | snapshot label |
| `expected_minor` | bigint | |
| `expected_on` | date | |
| `status` | varchar | expected, received, skipped |
| `financial_transaction_id` | FK unique nullable | actual receipt |
| timestamps | timestamp | |

Recurring occurrence generation follows the same idempotent pattern as bills.

Unique: `(income_schedule_id, expected_on)` when a schedule is present. Scheduled items copy source name and amount so later template edits do not rewrite history.

## 7. Bills and Subscriptions

### `bill_templates`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `name` | varchar | |
| `budget_category_id` | FK | Must be fixed section |
| `default_amount_minor` | bigint | estimate |
| `amount_is_estimate` | boolean | |
| `recurrence_unit` | varchar | day, week, month, year |
| `recurrence_interval` | integer | positive; quarterly is month/3 |
| `anchor_on` | date | recurrence anchor |
| `intended_day_of_month` | tinyint nullable | 1–31 for month/year recurrence |
| `account_id` | FK nullable | expected payment account |
| `is_subscription` | boolean | enables scenario cancellation analysis |
| `website_url` | varchar nullable | optional management link |
| `ended_on` | date nullable | Actual cancellation/end date, once confirmed |
| `disabled_at` | timestamp nullable | |
| timestamps | timestamp | |

V1 deliberately uses explicit recurrence fields rather than arbitrary cron or RRULE text. Monthly is month/1, quarterly month/3, semiannual month/6, annual year/1, weekly week/1, and biweekly week/2. For intended days 29–31, invalid months use their final day without changing the intended day for later months.

### `bill_occurrences`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `bill_template_id` | FK nullable | nullable for one-off bill |
| `budget_category_id` | FK | snapshotted category |
| `name` | varchar | snapshotted label |
| `expected_amount_minor` | bigint | preserved estimate |
| `paid_amount_minor` | bigint nullable | authoritative actual when paid |
| `due_on` | date | |
| `status` | varchar | upcoming, paid, skipped, cancelled |
| `financial_transaction_id` | FK unique nullable | payment/expense transaction |
| `paid_on` | date nullable | |
| `manually_modified_at` | timestamp nullable | protects it from template regeneration |
| timestamps | timestamp | |

Unique: `(bill_template_id, due_on)` when template is present. Index: `(household_id, status, due_on)`.

`MarkBillPaid` preserves `expected_amount_minor`, writes the actual to `paid_amount_minor` and the linked transaction, and exposes the variance. A separate `update_future_default` boolean, false by default and never accepted from an implicit/hidden value, may update the template plus future upcoming occurrences where `manually_modified_at` is null. The payment and optional template update commit atomically and are separately described in the audit entry.

## 8. Goals and Reserved Cash

### `goals`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `budget_category_id` | FK unique nullable | reserved category used for monthly contribution |
| `name` | varchar | |
| `type` | varchar | savings_goal or sinking_fund |
| `target_minor` | bigint nullable | |
| `target_on` | date nullable | |
| `default_monthly_contribution_minor` | bigint | non-negative |
| `display_order` | integer | |
| `archived_at` | timestamp nullable | |
| timestamps | timestamp | |

### `goal_entries`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `goal_id` | FK | |
| `type` | varchar | contribution, withdrawal, correction |
| `amount_minor` | bigint | Signed; contribution positive, withdrawal negative |
| `occurred_on` | date | |
| `financial_transaction_id` | FK nullable | optional supporting transaction |
| `entered_by_user_id` | FK | |
| `reason` | text nullable | required for correction |
| `voided_at` | timestamp nullable | |
| timestamps | timestamp | |

Index: `(goal_id, occurred_on, voided_at)`. Current reserved balance is `SUM(amount_minor)` over non-voided entries.

The persisted sum may be negative. `WithdrawFromGoal` requires a separate confirmation flag when the projected balance is below zero and records that confirmation in the semantic audit entry. Available-to-spend uses `max(current goal balance, 0)` for each goal; it never treats an overdrawn fund as negative reserved cash.

## 9. Month-End Decisions

### `month_end_allocations`

Records explicit user choices during review rather than automatically moving leftover funds.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK | |
| `budget_period_id` | FK | |
| `type` | varchar | leave_as_cash, goal, debt, invest, category_rollover |
| `amount_minor` | bigint | positive |
| `status` | varchar | draft or applied |
| `goal_id` | FK nullable | |
| `goal_entry_id` | FK nullable | approved goal contribution |
| `debt_account_id` | FK nullable | |
| `source_budget_category_id` | FK nullable | rollover source |
| `target_budget_category_id` | FK nullable | rollover target in next month |
| `financial_transaction_id` | FK nullable | if action was executed/recorded |
| `decided_by_user_id` | FK | |
| timestamps | timestamp | |

The target columns are constrained in the application according to `type`. Goal/debt/invest decisions link the entry/transaction the member explicitly records as completed; the application never initiates an external transfer. Leave-as-cash and rollover have no financial transaction.

For `category_rollover`, source and target are required, the amount is capped by the source's positive unused allocation, and aggregate month-end allocations are capped by positive actual free cash. Applying rollover atomically creates a new revision of the next period's active plan with the target allocation increased; it creates no financial transaction.

Actual month-end free cash excludes unused plan envelopes and remaining planned contributions; it protects projected liquid cash against short-term liability balances, known unpaid fixed bills, current positive goal reserves, and the household minimum cash buffer. This close-specific calculation must not reuse projected month-end free cash unchanged.

Draft allocations have no financial effect and do not increment the household financial-data version. When free cash is positive, draft rows total the reviewed amount exactly, including explicit leave-as-cash. `CloseBudgetPeriod` verifies the captured household financial-data version, atomically creates/links the approved entries/transactions/revision, marks allocations applied, closes the period, and increments the version. Concurrent financial changes force review refresh. Zero/negative free cash is acknowledged without positive allocation rows.

## 10. Import and Audit Support

### Future `import_batches`

The CSV-import milestone adds batch/file metadata, external references, row failures, and transaction origin columns in its own migrations. V1 manual transaction tables do not carry nullable import scaffolding. Imports must call the same actions and add database-backed idempotency constraints when implemented.

### `change_log`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `household_id` | FK nullable | null for pre-household auth events |
| `actor_user_id` | FK nullable | null for scheduler/system |
| `action` | varchar | semantic action name |
| `subject_type` | varchar | controlled morph map alias |
| `subject_id` | bigint | |
| `before` | json nullable | redacted changed values |
| `after` | json nullable | redacted changed values |
| `request_id` | ULID nullable | correlation |
| `created_at` | timestamp | append-only |

Audit rows explain meaningful financial changes. Passwords, tokens, recovery codes, and uploaded file content are never captured.

Closing and reopening a period always append semantic `budget_period.closed` or `budget_period.reopened` records. A reopen audit entry includes the required reason; earlier close/reopen history is never overwritten by the convenience timestamps on `budget_periods`.

## 11. Calculation Queries

### Projected account position

1. Select the latest balance snapshot at or before the calculation instant.
2. Convert the displayed snapshot balance to net position: positive for assets, negative for liabilities.
3. Add non-voided account postings whose `effective_at` is later than the snapshot.
4. Convert the result back to a positive displayed balance according to account classification.

If no snapshot exists, the account is incomplete and is excluded from totals with a visible warning rather than silently treated as zero.

### Monthly actual by category

Category progress depends on section:

- Fixed/flexible actual is non-voided purchases less refunds by category and month.
- Reserved goal actual is positive contribution entries for the goal attached to that category; withdrawals and corrections do not count as monthly contributions.
- A purchase linked to a goal withdrawal appears in goal activity and transaction history but does not count as a reserved contribution.
- Reserved investment/retirement actual is a non-voided categorized transfer into the category's validated funding account.
- A goal contribution supported by a transfer is counted through its goal entry only; the transfer category remains null to prevent double counting.
- Income and uncategorized transfers have no category actual.

### Fixed reservation without double counting

For each fixed category:

```text
planned remaining = max(active allocation - actual spending, 0)
known unpaid = sum(unpaid bill occurrences)
fixed reservation = max(planned remaining, known unpaid)
```

An unpaid bill without a category allocation contributes its full amount. This uses the category plan as the normal envelope and the bill schedule as better information when known obligations exceed it.

### Short-term liability reservation

Sum the positive projected balances of liability accounts with `reserve_from_cash = true`. Credit cards default to true. Mortgage, auto-loan, and student-loan principal defaults to false because their scheduled monthly payments are already fixed obligations.

### Goal reservation

Current goal reservation is the sum of `max(goal entry balance, 0)` across goals. Remaining planned reserved contribution for the month is `max(active reserved allocation - qualifying contribution actual, 0)` per reserved category.

## 12. Integrity and Concurrency

- SQLite foreign key enforcement must be enabled in every environment.
- Adoption, bill payment, goal contribution, and month close run in database transactions.
- The parent aggregate row is selected with `lockForUpdate()` where the database supports it; `lock_version` additionally detects stale scenario forms and remains effective on SQLite.
- Unique keys make bill/income occurrence generation and imports safe to retry.
- Aggregate actions validate household ownership before persistence; route model binding alone is not an authorization boundary.
- Database constraints cover shape and uniqueness. Domain actions cover multi-row and type-dependent invariants.

## 13. Backup and Retention

- Back up the SQLite database using SQLite's online backup mechanism or a transactionally consistent database command, not a blind copy during writes.
- Store backups outside the served `public` directory.
- Support encrypted export as a later milestone; do not claim exports are encrypted until implemented and recovery-tested.
- Test restore procedures, including attachments/import files, before relying on backup automation.
- Preserve operational history by default. Data pruning is limited to caches, expired sessions, old jobs, and explicitly configured raw import files.

## 14. Operational Health Tables

### `service_heartbeats`

| Column | Type | Notes |
| --- | --- | --- |
| `service` | varchar PK | scheduler or queue |
| `last_seen_at` | timestamp | UTC |
| `status` | varchar | healthy or error |
| `safe_detail` | json nullable | no paths, payloads, or secrets |

The scheduler updates its own heartbeat and dispatches a unique queue-heartbeat job each minute; processing that job updates the queue row. This distinguishes an idle queue from a dead worker.

### `backup_runs`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | bigint PK | |
| `started_at` | timestamp | |
| `completed_at` | timestamp nullable | |
| `status` | varchar | running, verified, failed |
| `backup_identifier` | varchar | safe basename/ULID, not full private path |
| `database_sha256` | char(64) nullable | |
| `integrity_result` | varchar nullable | |
| `failure_code` | varchar nullable | safe controlled code |

Health endpoints read only the latest rows and return coarse status; operational paths and error details remain in private local logs.
