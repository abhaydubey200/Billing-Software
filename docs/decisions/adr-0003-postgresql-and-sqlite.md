# ADR-0003: PostgreSQL in the cloud, SQLite locally

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

The product must keep working without internet access, yet also support reporting,
account management and cross-device access. That requires two database layers.

## Decision

**Two layers:**

1. **Local database (per device): SQLite via Drift** — type-safe persistence, SQL
   queries, transactions and migrations for Flutter across mobile, desktop and
   web. Stores the shop's local operational records and the outbox.
2. **Cloud database: PostgreSQL** — the primary source of synchronised business
   records; supports reporting, account management and cross-device access.

Additional rules:

- Model invoices, invoice lines, payments and inventory movements with
  **relational tables and constraints**.
- Related changes must commit in a **single database transaction**. Creating an
  invoice must never leave an invoice without its corresponding inventory
  movements because one operation failed.
- Use **explicit SQL and transaction control** with a maintained PostgreSQL driver
  (e.g. `pgx` or equivalent), plus **versioned schema migrations**.

## Consequences

- The same logical data model exists in two places, so schema changes must be
  coordinated between the SQLite (Drift) migrations and the PostgreSQL migrations.
- Drift's exact backing store and **durability characteristics must be verified per
  target platform** — do not assume identical behaviour on Android, Windows and web.
- PostgreSQL provides the transactional and replication capabilities the
  architecture needs, but high availability still requires deliberate configuration
  and operational testing.
- Local and production databases use the same engine family, easing SQL reuse, but
  dialect differences must still be handled.

## Alternatives considered

- **Server-only database (no local store):** fails the offline-first requirement.
- **Cloud-based embedded/NoSQL store:** weaker relational integrity for financial
  records and weaker transactional guarantees for linked changes.
- **Same engine everywhere (e.g. PostgreSQL locally):** impractical on mobile;
  too heavy for a per-device store.
