# UI/UX Design

## Interaction Principles

- Lead with household status, available-to-spend, and the reason behind them.
- Keep common mobile actions reachable with one hand and usable without horizontal tables.
- Show negative and incomplete states honestly; do not replace them with reassuring colors or clipped values.
- Present a concise default and progressive disclosure for optional detail.
- Confirm destructive or exceptional actions, not ordinary valid entry.
- After every write, show the committed result rather than an optimistic value that may differ from server calculations.
- Use accessible labels, focus states, keyboard operation, reduced-motion support, and color-independent status cues.

## Global Navigation

Desktop uses a persistent sidebar: Dashboard, Budget, Planner, Transactions, Bills, Goals, Accounts, Debt, Reports, and Settings.

Mobile uses a five-item bottom bar: Dashboard, Budget, Quick Add, Bills, and More. More contains Planner, Transactions, Goals, Accounts, Debt, Reports, and Settings. Quick Add is an action, not a route tab.

A persistent quick-add purchase action is available from the dashboard and primary navigation.

The selected budget month appears in the page header. Changing it updates the URL query parameter and every month-scoped section. Returning to the app defaults to the current household-local month, not the last historical month viewed.

## First-Run and Member Setup

Loopback-only bootstrap collects:

1. Household name, timezone, and USD base currency confirmation.
2. First member name, email, and password.

Bootstrap becomes permanently unavailable after those records commit. The first member is then signed in and completes an authenticated setup checklist:

1. Minimum cash buffer and freshness defaults, prefilled with documented defaults.
2. Initial account creation and observed balances.
3. Default category set review.
4. Current-month income and fixed bills.
5. Initial active budget review.

The checklist is resumable, never weakens authorization, and remains accessible from Settings after initial completion.

The second-member settings flow generates a 30-minute invitation with:

- a QR code containing the canonical HTTPS invite URL;
- a copy button and short fallback code;
- expiration countdown and revoke control;
- a reminder that the device must trust the household CA first.

The recipient chooses name, email, and password. Acceptance ends at the dashboard and consumes the invite.

## Dashboard

Desktop order:

1. Month header with headline status and concise explanation.
2. Available-to-spend card with `View calculation` breakdown.
3. Projected month-end free cash and income expected/received.
4. Action-required panel for overdue bills, stale/missing balances, pending plan actions, and overdrawn goals.
5. Budget progress for fixed, flexible, and reserved sections.
6. Upcoming bills.
7. Account/net-worth summary.
8. Goal and debt summaries.

Mobile keeps the same semantic order in one column. The headline, available-to-spend, projected free cash, actions, and quick add all appear before charts or account lists.

Every calculated card shows:

- household-local calculation time;
- freshness/incomplete indicator when applicable;
- signed, unclipped values;
- a breakdown or source link;
- stable reason-code-derived wording.

Status presentation:

| Status | Visual treatment | Required text cue |
| --- | --- | --- |
| On track | positive/green token | check icon and “On track” |
| Attention needed | amber token | warning icon and “Attention needed” |
| Over budget | orange token | overage icon and “Over budget” |
| Overallocated | critical/red token | critical icon and “Overallocated” |

Color is never the sole cue. All applicable reasons appear below the headline in severity order.

## Budget

The budget page has Fixed, Flexible, and Reserved sections. Each category row shows planned, actual/contributed, remaining, and progress. Negative remaining values stay signed and visible.

- Fixed rows show linked known bills and whether known unpaid exceeds plan remaining.
- Flexible rows support ordinary active-plan editing only through a new plan revision; the current adopted revision is never silently mutated.
- Reserved rows show the goal or funding account destination and contribution progress.
- Disabled categories are hidden by default with an explicit history filter.
- Reordering is a management action and never changes historical reports.

Editing the active plan creates a named revision with a before/after summary and confirmation. Minor label/order changes to reference data do not create plan revisions unless monetary allocations change.

## Scenario Planner

Planner layout contains:

- scenario name, base revision, and unsaved/stale indicator;
- active-versus-scenario summary;
- flexible category controls;
- reserved contribution controls;
- one-off planned income items;
- individual subscription cancellation controls;
- available-to-spend and projected-month-end result with full breakdown;
- save, duplicate, discard, and adopt actions.

Sliders always have an adjacent exact currency input and keyboard controls. Server calculation is debounced during adjustment and runs immediately on blur/commit. Negative projections remain savable.

Adoption uses a review screen showing every allocation, income, and subscription difference plus current-versus-after-actions cash. The final button says `Adopt plan`; no state changes before it succeeds.

## Transactions

The transaction list defaults to the selected month and newest first. Filters include type, category, account, entered by, and voided visibility. Filtering by member never changes dashboard totals.

Rows show date, merchant/description, category or transfer label, account path, amount, entered-by initials, and correction state. Selecting a row opens details with postings, linked bill/income/goal records, audit history, void, and replace actions.

Voiding requires a reason except the 10-second quick-entry undo, which supplies the predefined reason. Replacing creates the corrected transaction and links both records in one action.

## Bills

The default view groups Overdue, Due soon, Later, and Paid this month. Each occurrence remains separate from its recurring template.

- `Mark paid` is available inline and opens the compact flow already specified.
- Skipping/cancelling requires confirmation and a reason.
- Editing an occurrence clearly distinguishes `this occurrence` from `future default`.
- Template management shows recurrence, next due date, expected account, subscription status, and history.
- Pending subscription cancellation actions remain in the action-required panel until resolved.

## Goals and Sinking Funds

Goal cards show reserved balance, target, target date, monthly contribution, required contribution, forecast date, and progress. Every card states that the amount is a virtual household cash reserve.

Actions are Contribute, Withdraw/use, Correct, Edit plan, and Archive.

- Contribution may be virtual or linked to an account transfer.
- Withdrawal exceeding balance shows reserved amount, excess amount, resulting negative balance, and explicit confirmation.
- Overdrawn goals use critical text and signed balance; progress is not clamped to a misleading zero-only display.
- Correction requires a reason and is visually distinguished from normal contribution/withdrawal history.

## Accounts and Debt

Accounts group Liquid assets, Other assets, Short-term liabilities, and Long-term liabilities. Each row shows observed balance, projected balance, observation time, freshness, and net-worth inclusion.

`Record balance` is the primary action. It never asks the user to reconcile prior transactions; the resulting observation becomes the new projection baseline.

Debt detail shows balance freshness, APR, minimum/planned payment, payoff projection assumptions, and linked bill. When principal splitting is unavailable, show `Balance updates with your next observation` after a payment rather than a guessed reduction.

## Month-End Review

Review is a guided sequence:

1. Resolve or acknowledge missing/stale balances and unpaid/overdue bills.
2. Review income and category plan versus actual.
3. Show actual positive free cash available for decisions.
4. Allocate it among leave as cash, goals, debt, investment, and explicit category rollover.
5. Preview next-month plan effects.
6. Confirm close.

The UI requires positive free cash to be fully assigned, with `Leave as cash` holding any retained portion. It prevents over-allocation and rollovers above their source unused allocation. Goal allocation creates a virtual reserve only after confirmation. Debt/invest choices require the member to confirm the external action occurred and record the source/destination; the app never initiates it. A changed balance/transaction invalidates the review version and requires refresh before close.

A closed month shows a read-only banner and `Reopen month` action. Reopen requires a reason, returns to review, and explains that the month must be closed again.

## Forms, Feedback, and Accessibility

- Currency fields accept ordinary decimal input, display the household currency, reject more than two fractional digits for USD, and never pass floats to domain actions.
- Dates display in household-local format while retaining an accessible full date label.
- Primary actions use explicit verbs; exceptional confirmations state the financial consequence.
- Buttons disable during submission; idempotency protects repeated requests.
- Loading placeholders preserve layout. A failed calculation shows the last inputs and retry action, never a stale total presented as current.
- Empty states explain the first useful action and do not manufacture zero-valued financial certainty.
- Charts duplicate essential values in text/table form.
- Touch targets are at least 44 by 44 CSS pixels.
- WCAG 2.2 AA contrast and keyboard behavior are required.
- Respect `prefers-reduced-motion`; animations are never required to understand state.
- After modal/sheet close, focus returns to the invoking control; validation moves focus to the first invalid field.

## Mobile Quick Purchase

The default sheet contains:

1. Amount, focused immediately with a decimal currency keypad where supported.
2. Category, using recent/favorite ordering plus search.
3. Save.

Defaults and optional behavior:

- Purchase date defaults to household-local today.
- The current member's last-used enabled account/card is preselected and may be removed.
- Merchant/payee, notes, and changing the date are behind an optional details expansion.
- The form never asks for household or member ownership; the server derives them.
- Save disables while submitting and carries an idempotency token to prevent double taps.
- On success, the sheet closes, affected dashboard/budget regions refresh from server results, and focus returns to the invoking control.
- A success toast offers `Undo` for 10 seconds. Undo invokes an audited void with reason `quick_entry_undo`; it never hard-deletes the transaction.
- After the toast expires, the normal transaction correction/void workflow remains available.
- If account/card is omitted, the success state explains that the budget is updated but account-based available-to-spend is incomplete until assigned/reconciled.

Validation appears next to the relevant field and preserves entered values. Amount and category are the only required user-entered fields.

## Mark Bill Paid

- Amount defaults to the occurrence estimate and may be replaced with the actual paid amount.
- Paid date defaults to today; the template's expected payment account is preselected when enabled.
- When actual differs from estimate, show an unchecked `Use this amount as the new future default` control.
- Leaving it unchecked changes only the occurrence and its linked transaction.
- Checking it explicitly updates the recurring template and eligible future unpaid/unmodified occurrences.
- Success closes the control and refreshes bills, fixed reservation, category actuals, accounts, and dashboard status from the committed result.

## Scenario Subscription Toggle

- Each subscription is toggled individually, even when several share a category.
- Turning one off asks for/effectively displays a cancellation date, defaulting to household-local today for the current-month scenario or the period start for a future scenario.
- The comparison shows excluded unpaid occurrences, next-12-month savings, and monthly equivalent.
- Paid charges remain visible as actuals and are never visually removed.
- The UI states that scenario changes do not cancel the external service or alter the real bill until adoption resolves that action.

### Adoption follow-up

- Adoption preview separates `projected after actions` from `currently reserved until actions are completed`.
- Each assumed cancellation becomes a dashboard checklist item with `Confirm cancelled` and `Keep subscription` actions.
- Confirmation asks for the actual effective date before changing the real bill schedule.
- Keeping the subscription explains that the active plan will receive a corrective revision.
- Pending actions contribute an `attention needed` reason but do not block adoption.
