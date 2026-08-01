# Architecture Decision Record

This document records important product, financial, and technical decisions made during project discovery.

## ADR-001: Build a local-first household finance application

**Decision:** Build a self-hosted, local-first finance application instead of relying on a paid budgeting SaaS.

**Context:** The household wants a shared view of finances without paying out of pocket or sending financial data to third-party services.

**Alternatives Considered:**

- Paid budgeting apps.
- Free spreadsheets.
- Existing open-source finance tools.
- Custom application.

**Rationale:** A custom local-first app gives the household control over data, workflow, privacy, and future automation.

**Consequences:**

- More initial development effort.
- No dependency on paid services.
- Easier to tailor the app to household-specific planning workflows.

## ADR-002: Start manual-first, automate later

**Decision:** V1 will use manual entry for balances, bills, income, category totals, and purchases. Automation is a long-term goal.

**Rationale:** Manual entry lets the product become useful quickly while avoiding bank-sync complexity early.

**Consequences:**

- The interface must make manual entry fast.
- The data model should support future CSV import and bank syncing.
- Automation must not be required for core value.

## ADR-003: Use one household with two equal users

**Decision:** The initial app supports one household and two users with equal permissions.

**Rationale:** The target use case is a shared household view for spouses. Multi-tenant support is unnecessary for V1.

**Consequences:**

- Simpler authorization model.
- No role hierarchy initially.
- Other households can run their own instance if the project is shared.

## ADR-004: Make available-to-spend a primary metric

**Decision:** The dashboard should prioritize available-to-spend over raw account balances.

**Rationale:** Account balances do not show how much money is safely spendable after bills, obligations, and reserved funds.

**Consequences:**

- The app must model reserved funds.
- Fixed expenses and upcoming bills affect available-to-spend.
- Negative available-to-spend must be supported.

## ADR-005: Separate fixed expenses, flexible spending, and sinking funds

**Decision:** The budget model will have three major sections:

1. Fixed expenses.
2. Flexible spending.
3. Savings goals and sinking funds.

**Rationale:** These categories behave differently. Fixed expenses are committed, flexible categories can be adjusted, and sinking funds are reserved but adjustable.

**Consequences:**

- Planning UI can lock fixed expenses.
- Flexible categories can use sliders.
- Sinking funds can reduce available-to-spend while remaining adjustable in scenarios.

## ADR-006: Credit card purchases affect budgets immediately

**Decision:** Credit card purchases reduce budget availability immediately, even though cash leaves later when the card is paid.

**Rationale:** Credit cards are the household's primary payment method. Waiting until payment would misrepresent spending progress.

**Consequences:**

- Purchases increase credit card liability immediately.
- Credit card payments are transfers, not new expenses.
- Budget categories stay aligned with purchase behavior.

## ADR-007: Scenarios are temporary until adopted, but can be saved

**Decision:** Budget planning changes happen in scenarios. Scenarios do not affect the active budget unless explicitly adopted. Scenarios may be saved for reuse or comparison.

**Rationale:** The household wants to experiment with sliders, subscription toggles, side income, and goals without accidentally changing the official plan.

**Consequences:**

- The system needs an active budget and saved potential budgets.
- Scenario comparison is a core workflow.
- Negative scenario cash flow is allowed and useful.

## ADR-008: Recommendations use a balanced default

**Decision:** Recommendations should use a balanced approach by default.

**Rationale:** The household wants trade-offs, not overly conservative or aggressive optimization.

**Consequences:**

- Recommendations should consider quality of life, debt payoff, savings goals, and cash flow together.
- The app may show multiple options instead of one answer.
- Future financial modes may be added later if needed.

## ADR-009: AI insights are optional and local

**Decision:** Any LLM-based insight feature should run locally and must not own core calculations.

**Rationale:** The household wants privacy and deterministic financial logic. AI may help summarize or explain trends later.

**Consequences:**

- Core calculations must be testable without AI.
- LLM features consume prepared financial summaries.
- Chat-with-data is deferred until the need is clearer.

## ADR-010: Keep financial history in the operational database

**Decision:** Keep all financial history in the operational database for V1. Provide backup/export before considering archival that removes queryable details.

**Rationale:** Household transaction volume is small for SQLite, year-over-year queries are more valuable when details remain online, and archive/restore creates meaningful correctness and recovery complexity without a measured problem.

**Consequences:**

- Year-over-year detail remains directly queryable.
- Backups and exports are not substitutes for deleting operational data.
- Retention is revisited only after measuring database size and query performance.

## ADR-011: Use a Laravel modular monolith

**Decision:** Build one Laravel application with logical modules organized by business capability.

**Rationale:** One household and two users do not justify distributed systems. Domain boundaries still improve clarity and testing inside a monolith.

**Consequences:**

- One application, database, deployment, and primary language.
- Actions and queries may cross modules explicitly inside the application process.
- No microservices, message broker, event sourcing, or generic package framework in V1.

## ADR-012: Use Laravel 13, Livewire, and the official starter kit

**Decision:** Start with Laravel 13 and the official Livewire starter kit using built-in Laravel authentication.

**Rationale:** The product is a private, server-authoritative, form- and dashboard-heavy application. Livewire provides reactive mobile and desktop interactions while keeping validation and financial state in PHP.

**Consequences:**

- PHP 8.3 or newer is required.
- Livewire, Blade, Tailwind, and Flux form the UI stack.
- Public registration is disabled after household setup.
- A separate SPA, public API, and third-party identity service are unnecessary.

## ADR-013: Use SQLite first

**Decision:** Use SQLite as the authoritative V1 database while keeping migrations and core queries relational and reasonably portable.

**Rationale:** SQLite is local, free, operationally small, backup-friendly, and sufficient for two household users.

**Consequences:**

- Foreign keys must be enabled.
- Interactive concurrency uses optimistic version checks and unique constraints, not assumptions about row-level locking.
- Queue storage may also use the database; Redis is not required.
- PostgreSQL is an available future migration path, not a second supported environment from day one.

## ADR-014: Store money as integer minor units

**Decision:** Persist monetary values as 64-bit integer minor units and prohibit floating-point arithmetic in financial logic.

**Rationale:** Integer amounts are exact, easy to test, and sufficient while the household operates in one base currency.

**Consequences:**

- Monetary database columns use an `_minor` suffix.
- Input parsing and output formatting happen at application boundaries.
- Rates use scaled integer representation with explicit rounding rules.
- Multi-currency arithmetic is rejected until its rules are designed.

## ADR-015: Separate observed balances from recorded transactions

**Decision:** Preserve manual account balance observations and use signed account postings after the latest observation to calculate projected balances.

**Rationale:** The app must support both low-friction transaction entry and periodic manual balances without claiming exact bank reconciliation.

**Consequences:**

- Dashboard balances are labeled observed, projected, stale, or incomplete.
- A new observation establishes a projection baseline without deleting earlier transactions.
- Credit-card payments are net-zero transfers between an asset and liability position.
- Account selection remains optional for purchases, but omission is visible in projection freshness/completeness.

## ADR-016: Version adopted budgets and isolate scenarios

**Decision:** Treat adopted plan revisions as immutable. Scenarios are copies with lineage, and adoption atomically changes the period's active-plan pointer.

**Rationale:** Users need experimentation without accidental changes and must be able to understand what plan was active at a point in time.

**Consequences:**

- Actual transactions belong to period/category rather than a plan revision.
- Scenario edits use optimistic concurrency.
- Adoption never mutates the previous active plan in place.
- Historical plan comparisons remain explainable.

## ADR-017: Distinguish discretionary capacity from plan feasibility

**Decision:** Show `Available to Spend` and `Projected Month-End Free Cash` as separate metrics.

**Rationale:** Required obligations and reserves determine safe discretionary capacity, while the remaining flexible plan determines whether the complete plan fits. Combining the ideas makes scenario adjustment behavior ambiguous.

**Consequences:**

- Available-to-spend does not subtract remaining flexible allocations.
- Projected month-end free cash does subtract them and is the scenario planner's feasibility result.
- Checking, savings, and cash are included by default; credit-card balances, explicit reserves, fixed obligations, remaining reserved contributions, and a configurable minimum cash buffer are protected.
- Expected active-plan income remaining in the month is included and shown separately from cash already held.
- Both values may be negative and must have explainable breakdowns.

## ADR-018: Keep core writes synchronous and explicit

**Decision:** Required financial changes are performed by explicit application actions inside database transactions. Queues and events handle only work that may safely occur after commit.

**Rationale:** A user should never see a bill marked paid while its transaction or budget effect is waiting on a job. Hidden observers also make financial behavior difficult to trace.

**Consequences:**

- Livewire components, commands, and future endpoints reuse the same actions.
- Core side effects do not live in model observers or queued listeners.
- Jobs are reserved for imports, exports, optional insights, and other retryable follow-up work.

## ADR-019: Closed months require explicit audited reopening

**Decision:** Normal financial writes are rejected for a closed month. Either household member may reopen it to `reviewing` by providing a reason; the month must be reviewed and closed again after corrections.

**Rationale:** Closed reports should remain stable, while household members still need a transparent way to correct mistakes discovered later.

**Consequences:**

- Reopen and re-close actions preserve actor, time, reason, and prior decision history.
- Corrections recalculate the month from authoritative records.
- No hidden edit path bypasses the period state rule.

## ADR-020: Allow confirmed goal and sinking-fund overdrafts

**Decision:** A goal withdrawal may exceed its recorded balance after explicit confirmation. The negative balance remains visible as overdrawn, while reserved-cash calculations floor that goal's reservation at zero.

**Rationale:** The application must show actual household decisions rather than rejecting an expense merely because its virtual reserve was insufficient.

**Consequences:**

- The excess over the available reserve reduces available-to-spend normally.
- The UI must distinguish an overdrawn fund from ordinary incomplete progress.
- Confirmation is recorded in the audit history.

## ADR-021: Derive one headline status using fixed precedence

**Decision:** The monthly headline status is derived in the order overallocated, over budget, attention needed, and on track. All applicable reason codes and evidence are still returned.

**Rationale:** A single glanceable answer needs deterministic precedence, while financial decisions still require the complete explanation.

**Consequences:**

- Status is calculated, not stored or edited.
- A higher-severity state does not hide lower-severity bill, category, or freshness warnings.
- UI wording cannot redefine the calculation.
- Household defaults are 7-day liquid/credit-card freshness, 30-day other-account freshness, projection starting on day 7, and an overage tolerance equal to the greater of $25 or 5% of allocation.

## ADR-022: Host V1 on one Mac mini for private-LAN access

**Decision:** Run the application, SQLite database, worker, scheduler, and private files on the household Mac mini. Members may use that Mac or connect through a stable private-LAN address/port from phones and computers.

**Rationale:** This satisfies shared local access without cloud hosting, paid infrastructure, or sending financial data outside the household.

**Consequences:**

- V1 has no public internet exposure, router port forwarding, sharing tunnel, or away-from-home access.
- Runtime processes start automatically and survive Mac restarts.
- The web server serves only Laravel's `public` directory and the macOS firewall limits exposure.
- Development uses Herd; the household runtime uses native FrankenPHP/Caddy classic mode at `financials.local:8443` to avoid Herd port conflicts.

## ADR-023: Require locally trusted HTTPS on the household LAN

**Decision:** Every LAN connection uses HTTPS through `financials.local:8443` and a certificate issued by Caddy's internal household-local CA trusted on each approved device. No authenticated HTTP fallback exists.

**Rationale:** Financial data and authentication sessions should not travel unencrypted merely because the network is private.

**Consequences:**

- Approved devices require one-time local CA installation and trust.
- Secure, HTTP-only, same-site session cookies remain enabled.
- Private CA/server keys stay outside the repository and application exports.
- The deployment runbook must cover renewal, device removal, and CA recovery.

## ADR-024: Onboard the second member with a QR invitation

**Decision:** First-run setup is loopback-only. After bootstrap, the first member generates a 30-minute, one-time HTTPS invitation that may be scanned as a QR code; the second member chooses their own credentials and receives equal permissions.

**Rationale:** A QR flow is convenient on mobile and avoids displaying or transmitting a temporary password.

**Consequences:**

- Only the token hash is stored; the QR contains no password or personal data.
- Acceptance atomically consumes the invitation and enforces the two-member limit.
- The phone/computer must trust the household CA before setup.
- Public registration and email delivery remain disabled.

## ADR-025: Keep all financial reporting household-wide

**Decision:** Budgets, balances, available-to-spend, goals, debts, reports, and dashboard status always represent the household as one financial unit.

**Rationale:** Both members share equal control and the product answers a household planning question, not an individual expense-accounting question.

**Consequences:**

- Actor identity supports audit, activity, and optional transaction-history filters only.
- There are no per-member budgets or ownership shares in V1.
- A member filter must never alter household financial calculations.

## ADR-026: Make category rollover an explicit month-end decision

**Decision:** Category rollover is disabled by default and never automatic. A member may roll an amount during month-end review into a category in the next plan.

**Rationale:** An unused category allocation is not itself cash. Rollover should represent a conscious allocation of actual household leftover funds.

**Consequences:**

- Rollover is capped by both the source's unused allocation and aggregate positive actual free cash.
- It creates a new next-month plan revision, not a bank transaction.
- Unused allocations expire unless explicitly rolled over.

## ADR-027: Optimize quick purchase for amount and category

**Decision:** Mobile quick purchase requires only amount and category. Date defaults to today, the member's last-used account is preselected but removable, and merchant/notes/date changes use progressive disclosure.

**Rationale:** Manual-first operation succeeds only if the household's most frequent write takes a few seconds.

**Consequences:**

- A successful save closes the sheet and refreshes authoritative server totals.
- A 10-second undo performs an audited void rather than deletion.
- Double submissions are prevented with an idempotency token.
- Omitting the account is allowed but visibly makes account-based available-to-spend incomplete.

## ADR-028: Keep bill occurrence actuals separate from recurring defaults

**Decision:** Paying a bill records the actual amount on that occurrence and transaction. A differing amount may become the future template default only through an unchecked-by-default explicit control.

**Rationale:** One unusual bill should not silently distort every future plan, while repeated price changes should remain easy to adopt.

**Consequences:**

- Template updates affect only future unpaid, unmodified occurrences.
- Payment and an explicitly requested future-default update commit atomically.
- Audit history distinguishes the payment from the template change.

## ADR-029: Model scenario subscription cancellations per template and date

**Decision:** A scenario may exclude unpaid subscription occurrences on or after a selected cancellation date. The override belongs to one bill template and does not modify real bill data.

**Rationale:** Categories may contain multiple subscriptions, recurrence may not be monthly, and already-paid charges remain part of reality.

**Consequences:**

- Fixed-category scenario adjustments are derived from the excluded scheduled occurrences.
- Next-12-month savings use the real recurrence schedule; monthly equivalent is derived from that total.
- Editing a scenario never claims that the external subscription was actually cancelled.

## ADR-030: Adopt external assumptions as pending actions

**Decision:** Scenario adoption may proceed with subscription cancellation assumptions, but each becomes a pending plan action. The real unpaid bills stay reserved until cancellation is confirmed.

**Rationale:** Adopting a budget is not evidence that an external service was cancelled, and the app must not make cash appear available prematurely.

**Consequences:**

- Adoption preview distinguishes intended savings from currently protected obligations.
- Confirming cancellation updates the real template/occurrences and completes the action atomically.
- Retaining the subscription dismisses the action and creates a corrective active-plan revision.
- Pending plan actions contribute an attention-needed reason.

## ADR-031: Make every V1 goal a cash reserve

**Decision:** Every V1 goal and sinking fund represents virtually reserved household cash and reduces available-to-spend. Non-reserving informational targets are deferred as a separate concept.

**Rationale:** A single `Goal` meaning keeps reservation calculations explainable and prevents visually similar targets from having hidden different cash effects.

**Consequences:**

- Goals are backed by aggregate liquid cash, not tied to one bank account.
- Goal balances come from entries and may be overdrawn only through confirmed withdrawals.
- Informational targets must not be implemented by adding an `is_reserved` switch to V1 goals.

## ADR-032: Treat investment contributions as categorized transfers

**Decision:** A planned brokerage or retirement contribution uses a reserved category. Execution is a net-zero account transfer that fulfills the category allocation without counting as spending.

**Rationale:** Moving cash into an owned investment account reduces liquidity but not net worth and must not be reported as consumption.

**Consequences:**

- Only reserved categories may classify a transfer.
- The configured destination funding account is validated by the action.
- Goal-backed transfers are counted through their goal entry and are not categorized again.

## ADR-033: Never guess amortizing-debt principal

**Decision:** When a mortgage, auto-loan, or student-loan payment lacks an exact principal/interest split, V1 records the fixed cash outflow but does not post a guessed liability reduction.

**Rationale:** Deriving principal from incomplete assumptions would make net worth look precise while being wrong.

**Consequences:**

- Manual liability observations remain the authoritative debt balance.
- The UI shows debt freshness after payments.
- Exact split transactions are deferred; credit-card payments remain exact transfers.

## ADR-034: Count expected income only from the active plan

**Decision:** Available-to-spend includes remaining income items from the active plan. Possible or scenario-only income does not count until adopted; received unplanned income affects projected cash when recorded.

**Rationale:** The dashboard should not spend hypothetical income while still allowing scenarios to test it.

**Consequences:**

- Planned income is reduced as linked occurrences are received.
- Unplanned receipts are not double-counted as expected income.
- Windfalls are never allocated automatically.
