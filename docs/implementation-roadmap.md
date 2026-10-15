# RetailCore — Implementation Roadmap

> Status: Planned · Last updated: 2026-10-15
> Companion to [`architecture.md`](./architecture.md) and [`decisions/`](./decisions/).

## How to read this document

This roadmap stages the work needed to make the accepted architecture real. It
describes **planned** work: **no stage below is implemented**. Each stage lists
its goal and **acceptance criteria** — the observable conditions that must be
true before the stage is considered done.

Stages are sequential in dependency, not strictly in calendar time. A stage may
touch several modules, and later stages were intentionally deferred until earlier
foundations are proven (see ADR-0007).

Acceptance criteria assume the quality gates from ADR-0006 and ADR-0009: tests
and static analysis pass in CI before merge, and `main` stays in a known,
testable state.

---

## Stage 0 — Repository foundation and CI

**Goal:** a reproducible, reviewable project skeleton with automated checks.

Scope: repository layout for `apps/retail_app`, `backend/api`, `database/`,
`contracts/`, `infra/`; pinned Go and Flutter toolchain versions; Docker Compose
for local PostgreSQL; the `.github/workflows/ci.yml` jobs begin running for real
once the projects exist.

**Acceptance criteria**

- `backend/api/go.mod` and `apps/retail_app/pubspec.yaml` exist with pinned
  versions, so the Go and Flutter CI jobs stop skipping.
- `docker compose up` starts a local PostgreSQL that the backend can connect to.
- CI runs format, vet, build and tests for Go and Flutter on pull requests, and
  failures block merge.
- A contributor can clone the repository and run the documented setup with no
  undocumented manual steps.
- `.gitignore` prevents build output, local databases and secrets from being
  committed.

---

## Stage 1 — API and database migrations

**Goal:** the versioned REST contract and the PostgreSQL schema baseline, with a
migration workflow.

Scope: OpenAPI contract under `contracts/openapi.yaml` for `/api/v1`; migration
tooling under `database/migrations`; the tenant, location, user and device tables;
conventions for tenant-scoped columns and constraints.

**Acceptance criteria**

- Migrations apply cleanly from an empty database and are reversible or paired
  with a documented forward-fix strategy.
- The OpenAPI document describes the implemented endpoints and is validated in CI.
- Every business table carries a tenant key, and **row-level security policies**
  are enabled (ADR-0008).
- CI runs migrations against a fresh PostgreSQL and fails on error.
- A **cross-tenant access test** demonstrates that a guessed ID from another
  tenant is rejected (ADR-0008).

---

## Stage 2 — Tenant identity and local persistence

**Goal:** online activation, primary-device registration, and a working local
store with secure session material.

Scope: provider-driven account provisioning; temporary initial credential and
forced replacement during activation; primary-device registration; Drift schema
and migrations; secure storage of authentication/session material; explicit sync
statuses.

**Acceptance criteria**

- A shop owner account can only be created by an authorized provider action; there
  is no self-registration path (ADR-0012).
- First login with the temporary credential **forces** a password change before
  the app is usable; the prior credential is invalidated and the event is audited.
- No password or secret is stored in plaintext locally or in the cloud.
- The app persists local records and a durable outbox across restarts.
- Sync statuses (saved locally / pending / synchronized / needs attention) are
  visible and accurate against real device state.
- Cross-tenant and cross-boundary access tests cover guessed IDs and malformed
  requests.

---

## Stage 3 — Inventory and units

**Goal:** the shared inventory engine and validated units of measure.

Scope: products, product types, sellable variants with stable IDs, tenant-scoped
SKU and barcode uniqueness, configurable attribute definitions, validated `JSONB`
attributes, categories, suppliers, stock receipts, movements, counts and
adjustments; the unit catalog with dimensions and versioned conversion factors.

**Acceptance criteria**

- A single schema serves at least the garment, grocery and hardware examples
  without a schema fork (ADR-0010).
- SKU and barcode uniqueness is enforced **per tenant**, and a duplicate is
  rejected with a clear error.
- Custom attributes are validated against their definitions before storage; an
  invalid value is rejected.
- Unit conversion only occurs between **compatible dimensions** and uses
  **decimal-safe arithmetic** with documented precision.
- Conversion factors are versioned and historical movements retain the factor and
  quantity context used at the time.
- Stock movements are the source of truth; a cached balance reconciles to the
  movement history.
- A negative-stock attempt is blocked by default, and any override is
  permission-controlled and audited.

---

## Stage 4 — Billing and ledger

**Goal:** offline-capable billing with financially correct, traceable records.

Scope: invoices and line items with historical snapshots; payments, allocations
and customer credit derived from ledger entries; returns, exchanges, refunds,
credit notes and corrections; offline-safe invoice numbering; decimal-safe totals.

**Acceptance criteria**

- A completed sale commits local invoice, inventory, payment, credit and outbox
  effects **atomically** (ADR-0011).
- Finalized invoices preserve product, unit, price, discount, tax and amount
  **snapshots**; later catalogue edits do not change them.
- Totals use decimal-safe arithmetic and the documented **rounding policy**.
- Corrections create reversal/return/adjustment records; no finalized transaction
  is silently overwritten or deleted.
- A return cannot exceed the original sale's remaining eligible quantity and
  records restock vs. excluded stock.
- An exchange is stored as linked return and sale transactions.
- Credit balances are derived from ledger entries and allocations, never edited
  directly.
- Invoice numbers are unique within their series, offline-safe, and **not silently
  reused** after cancellation (ADR-0013).
- Receipt recovery works after an app crash mid-billing, consistent with the
  local commit.

---

## Stage 5 — Synchronization

**Goal:** safe, idempotent sync of locally committed work.

Scope: the outbox protocol; stable operation identifiers; server-side idempotency
with stored retry results; explicit conflict handling; retry safety across
timeouts, restarts and lost responses; suspension/revocation reconciliation.

**Acceptance criteria**

- Re-submitting the same operation (including after a lost response) creates **no
  duplicate** invoice, payment, ledger entry or stock movement (ADR-0005).
- Every mutating sync endpoint requires a client-generated stable operation ID and
  is idempotent.
- Each documented edge case has a test: close during sync, network loss
  mid-request, duplicate submission, price change while offline, reversal after a
  later payment.
- There is **no silent last-write-wins** for invoices, payments, ledger entries or
  stock movements; conflicts route to a "needs attention" state with a defined
  resolution.
- A device returning after suspension or password reset reconciles its state; the
  documented prolonged-disconnection policy is enforced.
- Multi-device offline billing stays **disabled** and the guard is tested.

---

## Stage 6 — Provider console

**Goal:** a separate, least-privilege interface for RetailCore personnel.

Scope: account management (create/provision/activate/suspend); credential
provisioning and reset initiation; session revocation; support cases;
entitlements; service health; audit history.

**Acceptance criteria**

- The console is a **separate interface** with its own authentication and
  authorization, not a super-user mode in the shop app (ADR-0012).
- Provider personnel cannot access shop financial or customer records by default;
  any support access is scoped, time-bounded and audited.
- Provisioning, activation, suspension, reset and revocation each write an
  **audit record**.
- Session revocation invalidates existing sessions server-side; the effect on an
  offline device matches the documented policy.
- Tests prove a shop owner cannot reach provider-only operations and vice versa.

---

## Stage 7 — Reports, import/export, backups and hardware validation

**Goal:** operational reporting, data portability, recoverable backups and
verified hardware support.

Scope: sales, payment, credit and low-stock reports; clearly described as **basic
margin reporting**, not accounting-grade valuation; import/export; backup and
restore procedures; printer/scanner configuration and validation.

**Acceptance criteria**

- Reports are labelled to distinguish basic margin reporting from full accounting
  valuation (ADR-0011).
- Data export and account-closure procedures exist and have been exercised.
- A backup can be **restored into a clean environment** and verified, not merely
  taken.
- Import validates input and reports rejected rows with reasons.
- Printer and barcode-scanner support is validated on **real supported devices**,
  and **manual product lookup remains available** independently of scanning.
- Browser functionality, if started, remains behind platform-specific validation.

---

## Cross-cutting quality gates

Applies to every stage, from ADR-0006, ADR-0008 and ADR-0009:

- Tests and static analysis pass in CI; merge is blocked on failure.
- Financial and inventory logic uses decimal-safe arithmetic with documented
  precision and rounding.
- Audit records exist for privileged and financially significant operations.
- No secrets in source control or diagnostic output.
- Documentation distinguishes designed behaviour from implemented behaviour.

## Unresolved decisions that block or shape later stages

Carried from [`architecture.md` §21](./architecture.md); these are **not settled**
and should be resolved as the relevant stage approaches:

- Tax rules, rates and eligibility validated with official guidance and a
  qualified tax professional (blocks Stage 4 completeness, Stage 7 reporting).
- Rounding and costing policy confirmed by the business owner (Stage 4).
- Prolonged-disconnection and device-replacement policy (Stage 5).
- Managed cloud provider / PostgreSQL offering (Stage 1 selection, deployment).
- Authentication library or identity provider (Stage 2).
- Error-tracking and analytics services (cross-cutting).
- Data retention, export and account-closure durations (Stage 7).
- Multi-device design — numbering, stock contention, conflict rules
  (post-release; ADR-0013).
- Supported printer/hardware matrix (Stage 7).
