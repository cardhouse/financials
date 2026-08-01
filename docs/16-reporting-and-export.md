# Reporting and Export Contract

## 1. Reporting Principles

- Reports use the same transactions, entries, observations, and calculators as the dashboard; they do not maintain a second ledger.
- Every report states household, date range, household timezone, calculation time, and completeness/freshness limitations.
- Actor filters are allowed only in activity/entry reports and never redefine household financial totals.
- Voided records are excluded from totals and available through an explicit correction-history view.
- Transfers are excluded from income/spending; reserved contribution transfers appear in contribution reports.
- Negative values and incomplete periods remain visible.

## 2. V1 Reports

### Monthly plan versus actual

For each selected period and category:

- active plan revision used for the displayed comparison;
- planned amount;
- fixed/flexible spending actual or reserved contribution actual;
- signed variance;
- known bill variance for fixed categories;
- status reasons.

Historical users may switch to another adopted revision for comparison, but actuals do not change.

### Spending by category

Non-voided purchases less refunds grouped by fixed/flexible category and household-local occurrence date. Reserved contributions and transfers are excluded. Sinking-fund purchases may be shown under `Goal use` with their reserved category but are not mislabeled as contributions.

### Income

Received income transactions grouped by source/month, with expected-versus-received comparison when an occurrence exists. Scenario-only and unreceived income is excluded from actual income.

### Reserved contributions and goal use

Shows goal contributions, investment/retirement categorized transfers, goal withdrawals, and resulting reserved balances separately. Linked goal transfer/contribution pairs count once.

### Net worth history

For each month-end instant and account:

1. Use the latest observation at or before month end.
2. Apply non-voided postings after that observation through month end.
3. Convert asset/liability net positions to net worth.
4. Exclude accounts with no qualifying observation and mark the month incomplete.

Do not use a later observation to backfill an earlier month. Report total assets, total liabilities, net worth, and excluded/stale account count.

### Cash-flow summary

Received income minus fixed/flexible spending for the date range. Transfers, virtual goal entries, and investment contributions are reported separately. This is household cash-flow reporting, not an accounting statement.

### Debt summary

Observed/projected balances, APR, minimum/planned payments, payoff estimate, observation freshness, and change across selected observation dates. Payments without exact principal split do not fabricate historical principal reduction.

### Year summary

For a calendar year: income, fixed/flexible spending, reserved contributions, goal use, net-worth start/end/change where observations permit, debt balance change, and category monthly trend. All twelve months remain drillable.

## 3. Charts

V1 charts are:

- monthly spending by fixed/flexible section;
- category trend over selected months;
- income versus spending by month;
- net-worth line with incomplete-month markers;
- debt balance trend from observations;
- goal reserved balance/contribution trend.

Every chart has an accessible table containing the same numbers. Missing data is a gap/marker, never a zero data point.

## 4. Financial Data Export

V1 export is an authenticated, password-confirmed ZIP generated to a private temporary disk and downloaded once over HTTPS. It contains:

```text
manifest.json
household.csv
members.csv                 # names/emails only; no auth secrets
accounts.csv
account_balance_snapshots.csv
categories.csv
transactions.csv
account_postings.csv
budget_periods.csv
budget_plans.csv
budget_allocations.csv
bills.csv
bill_occurrences.csv
income_sources.csv
income_occurrences.csv
goals.csv
goal_entries.csv
month_end_allocations.csv
change_log.csv              # redacted audit values
```

The manifest includes schema version, application version, export instant, base currency, timezone, file checksums, and row counts.

Exports exclude password hashes, remember/session tokens, 2FA secrets/recovery codes, invitation tokens, app/CA keys, environment variables, cache, queue payloads, logs, and backup configuration.

The export is not encrypted in V1. Before generation, the UI warns the member to store it securely. The temporary server copy expires after the first successful download or one hour, whichever comes first. Export generation and download are audited without recording file contents.

## 5. Export Is Not Backup/Restore

CSV/JSON export supports transparency and portability. It is not the operational restore mechanism and V1 does not import its ZIP. Disaster recovery uses the verified SQLite/private-files backup in `15-deployment-runbook.md`.

## 6. Query Tests

- Report totals reconcile to transaction/entry fixture sums.
- Transfers do not inflate spending or income.
- Refunds reduce spending in their occurrence month.
- Voids disappear while replacement appears once.
- Linked goal transfers count once.
- Month-end net worth never uses an observation after the cutoff.
- Missing account history marks incomplete instead of assuming zero.
- Export row counts/checksums match generated files and sensitive columns are absent.
