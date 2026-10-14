# ADR-0005: Offline-first sync via outbox and idempotency

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

Shops must keep operating with no network. When connectivity returns, locally
recorded work must reach the server **without creating duplicates or corrupting
financial records**. This is the highest-risk area of the system.

## Decision

**We do not synchronise the entire local database blindly.**

- The client maintains a **durable outbox** of operations that need to reach the
  server. Each operation carries a **stable identifier**.
- The backend **records processed operation identifiers** and enforces
  **idempotency**, so retrying a request never creates duplicate transactions.

The protocol must explicitly define behaviour for these cases **before
multi-device offline operation is enabled**:

1. The application closes during synchronisation.
2. The network disappears halfway through a request.
3. A device submits the same operation twice.
4. A product's price changes while a device is offline.
5. An invoice is reversed after another payment has been recorded.

## Consequences

- Financial correctness does not depend on the network being reliable.
- Every mutating endpoint that participates in sync must be **idempotent** and take
  a client-generated stable operation identifier.
- The backend needs storage for processed operation IDs (and enough of their result
  to return a consistent response on retry).
- Conflict resolution rules (e.g. offline price drift, reversal ordering) are
  **product decisions** that must be written down, not left to implementation
  accident.
- Sync correctness must be covered by integration tests.

## Alternatives considered

- **Full local database sync:** rejected. Cannot resolve conflicting concurrent
  edits deterministically and risks overwriting or duplicating business records.
- **Server-authoritative only (no offline writes):** rejected. Breaks the offline
  requirement.
- **Last-write-wins without operation identity:** rejected. Creates duplicates on
  retry and loses financial corrections.
