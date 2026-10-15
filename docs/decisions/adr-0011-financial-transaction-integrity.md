# ADR-0011: Financial transaction integrity, stock movements, returns and invoice snapshots

- Status: Accepted
- Date: 2026-10-15
- Deciders: RetailCore engineering
- Supersedes: none
- Related: ADR-0003 (PostgreSQL + SQLite), ADR-0005 (outbox + idempotency), ADR-0010 (inventory and units), ADR-0013 (invoice numbering and tax)

## Context

RetailCore records money and stock. These must be **traceable and correct**:
a shop must be able to explain what was sold, for how much, with what tax, and
where the stock went, long after the fact. Silently overwriting or deleting a
finalized transaction destroys that ability and is unacceptable for a billing
system.

Offline operation makes this harder: the client commits locally before it can
reach the server, so both local and cloud effects must be internally consistent
and safe to replay.

## Decision

**Stock is recorded as traceable movements, not as a mutable quantity field
alone.** A cached balance may exist for speed, but the movement history is the
source of truth.

Model: finalized invoices, line items, payment records, payment allocations,
customer credit, returns, refunds, exchanges, credit notes, stock receipts and
adjustment history.

### Required rules

- **Finalized invoices preserve historical snapshots** of product, unit, price,
  discount, tax and amount. Later catalogue edits must not change a past invoice.
- **Decimal-safe arithmetic** with an explicitly documented **rounding policy**.
- **Financial corrections are additive and traceable**: reversal, cancellation,
  return or adjustment records. Finalized transactions are **not silently
  overwritten or deleted**.
- **Negative stock is blocked by default.** Any override is explicit,
  permission-controlled and audited.
- **Returns** reference the original sale, cannot exceed its remaining eligible
  quantity, and record whether returned stock is **restocked or excluded from
  sellable stock**.
- **Exchanges** are represented as **linked return and sale transactions** — not
  as an in-place mutation of the original bill.
- **Credit balances are derived** from ledger entries and allocations, not stored
  as arbitrary editable balance fields.
- Recording UPI, card or cash as a **tender method does not mean the payment has
  been independently verified**. The system records what was tendered; it does
  not assert that a bank or PSP confirmed it.
- **Atomicity:** local invoice, inventory, payment, credit and outbox effects that
  form one completed sale commit together; cloud-side related effects commit in a
  single database transaction.

### Costing strategy for the first release

Adopt a **simple inventory costing strategy** (for example, moving-average or
latest-cost) for basic margin reporting only. We must **clearly distinguish
basic margin reporting from full accounting-grade valuation**: RetailCore's first
release is not a general ledger or a stock-valuation system.

## Consequences

- Every stock and money figure can be traced to the movements and entries that
  produced it.
- Storage grows with history; movement and ledger tables must be indexed and
  potentially partitioned later (ADR-0007) — not pruned in ways that break audit.
- Corrections require new documents, so UI and reporting must present original
  and correcting documents together.
- Report queries are more complex than reading a single mutable column, but are
  correct.
- The costing choice is a product decision that must be documented and
  consistently applied; changing it later requires a stated migration approach.

## Alternatives considered

- **Mutable quantity and balance columns:** rejected. Loses history and makes
  reconciliation impossible.
- **Hard-delete or edit finalized invoices:** rejected. Destroys auditability.
- **Trust client-supplied totals as final on the server:** rejected. The server
  must validate and commit its own consistent records.
- **Full double-entry general ledger in the first release:** rejected as
  premature scope; basic margin reporting is the stated goal.
