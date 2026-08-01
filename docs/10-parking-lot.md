# Deferred Scope

There are no unresolved decisions blocking V1 implementation. Items in this document are explicitly deferred and must not be implemented, promoted into V1, or treated as a reason to pause unless a new product decision changes scope.

## Deferred Product Capabilities

- CSV import with mapping, preview, duplicate detection, and batch rollback.
- Direct bank/card connectors and credential storage.
- Transaction splitting across multiple budget categories.
- Pending-versus-cleared institution transaction status.
- Multi-currency accounts and exchange-rate history.
- Exact mortgage/auto/student-loan principal-and-interest splits when statement data is unavailable.
- Investment lot, tax, performance, and cost-basis accounting.
- Non-reserving informational goals or aspirational targets.
- Per-member budgets, balances, or ownership shares.
- Offline browser/PWA writes and conflict resolution.
- Native mobile applications.
- Notifications, channels, and quiet hours.
- Remote access away from the home LAN.
- Shared demo or sanitized-household mode.
- Local-AI narratives, chat, or recommendations.
- Automated money movement or automated month-end allocation.

## Deferred Infrastructure

- PostgreSQL as a second deployment profile.
- Redis, Horizon, websockets, or real-time broadcasting.
- Public API and token authentication.
- Encrypted portable exports until key recovery and restore are designed and tested.
- Prior-year archival out of the operational database.
- Public cloud hosting, public sign-up, and multi-tenant SaaS behavior.

## Permanently Out of Scope Unless the Product Is Rechartered

- A full double-entry general ledger.
- Automatic financial decisions without member approval.
- Paid third-party services required for a core workflow.
- AI-owned calculations or status.
- Microservices, event sourcing, or a generic accounting platform.

## Rule for Agents

When implementation encounters a deferred capability, preserve the documented extension seam, add no speculative abstraction beyond what V1 needs, and continue with the committed V1 behavior. Do not ask the user to choose among deferred alternatives during V1 delivery.
