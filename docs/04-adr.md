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

## ADR-010: Prior-year data may be archived

**Decision:** The app may keep the current year active while archiving prior-year details into restorable yearly snapshots.

**Rationale:** The household wants year-over-year visibility without unnecessarily growing the active database.

**Consequences:**

- Yearly summary data should stay available.
- Full details should be restorable if needed.
- Archive/export design should be considered before implementation.