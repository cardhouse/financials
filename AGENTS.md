# Financials Agent Contract

## Objective

Build the documented V1 household finance application. Product decisions required for implementation are complete. Do not ask the user to revisit an established default or a deferred feature.

## Read Before Editing

Read these files completely, in order, before implementation work:

1. `docs/13-implementation-guide.md`
2. `docs/05-financial-rules.md`
3. `docs/06-domain-model.md`
4. `docs/11-database-design.md`
5. `docs/12-laravel-programming-principles.md`
6. `docs/14-calculation-acceptance-tests.md`
7. The relevant milestone in `docs/09-roadmap.md`
8. `docs/07-ui-ux.md` for UI work
9. `docs/15-deployment-runbook.md` for runtime/deployment work
10. `docs/16-reporting-and-export.md` for reports/export work

Consult `docs/04-adr.md` for rationale and decision precedence. `docs/10-parking-lot.md` is deferred scope, not an invitation to implement or ask about it.

## Non-Negotiable Architecture

- Laravel 13, PHP 8.4, official Livewire starter kit, SQLite, Pest, Pint, Larastan.
- One modular monolith, one household, exactly two equal-permission users.
- Household-wide financial totals; actor identity is audit/filter metadata only.
- Integer minor units for money and scaled integers for rates; never financial floats.
- Pure deterministic calculators; AI never participates in financial logic.
- Explicit application Actions for writes and Query objects for reads.
- Required multi-row financial changes are synchronous database transactions.
- No core financial behavior in model observers or queued listeners.
- Adopted plans are immutable revisions; scenarios are isolated until atomic adoption.
- Preserve posted history through void/replacement, entries, revisions, and semantic audit.
- SQLite/unique constraints plus optimistic `lock_version`; do not assume row locking.
- Serve only `public/`; private data never enters Git or the web root.

## Scope Rule

Implement roadmap milestones in order. Do not add bank sync, CSV import, transaction splitting, multi-currency, public API, AI, notifications, remote access, per-member budgets, double-entry accounting, or other deferred capabilities during V1 work.

Use the smallest design that satisfies current documented behavior. Preserve a clean seam only where the architecture document explicitly identifies one; do not build speculative generic abstractions.

## Financial Correctness Gate

- Implement applicable cases in `docs/14-calculation-acceptance-tests.md` before UI polish.
- Negative and incomplete states are valid outputs, not exceptions to hide.
- Every calculated result returns an explainable breakdown and stable reason codes.
- Never double-count bills versus fixed allocations, goal entries versus linked transfers, or credit-card purchases versus payments.
- Never guess a loan principal split.
- When an account is omitted, mark account-derived values incomplete rather than inventing a precise balance.

## Required Work Pattern

1. Confirm the current roadmap milestone and relevant invariants.
2. Write/update unit tests for calculations and state transitions.
3. Add migrations/models/factories/policies.
4. Implement typed Actions and Queries.
5. Add feature tests for authorization, atomicity, idempotency, and concurrency.
6. Build the Livewire UI following `docs/07-ui-ux.md`.
7. Run:

```text
composer validate
vendor/bin/pint --test
vendor/bin/phpstan analyse
php artisan test
```

8. Report exactly what is complete, what was verified, and the next roadmap step.

## Documentation Changes

Implementation must conform to documentation. Change a product rule only when the user explicitly changes it. If code reveals a genuine contradiction:

1. Cite the conflicting passages.
2. Apply the precedence in `docs/13-implementation-guide.md` when it resolves the conflict.
3. Otherwise stop only that conflicting slice, document the issue, and continue independent in-scope work.

Do not turn ordinary engineering judgment—private helper names, component extraction, indexed query shape—into a user question.

## Data and Security

- Treat all data as sensitive even on the home LAN.
- Never log form payloads, notes, raw imports, credentials, tokens, recovery codes, exports, or full financial records.
- Derive household/user ownership server-side; never trust hidden identifiers.
- Keep public registration and email password reset disabled.
- LAN runtime is HTTPS-only at `https://financials.local:8443`; see the deployment runbook.
- No destructive data command, broad deletion, or history rewrite without explicit user authorization.

## Completion Standard

A feature is not complete until its financial invariant, authorization, correction/history behavior, empty/incomplete state, mobile interaction, audit behavior, and automated tests are implemented. Passing a happy path alone is insufficient.
