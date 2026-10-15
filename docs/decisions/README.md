# Architecture Decision Records (ADRs)

Each ADR captures one significant, hard-to-reverse decision: the context, the
decision, and its consequences. Decisions follow the RetailCore Final Technology
and Development Strategy.

**Status values:** Proposed · Accepted · Superseded by ADR-NNNN · Deprecated

To add a decision, copy the format below into `adr-NNNN-short-title.md` and add a
row to the index. Do not rewrite history — supersede instead.

| ADR | Title | Status |
|---|---|---|
| [0001](./adr-0001-flutter-for-clients.md) | Flutter for mobile and desktop clients | Accepted |
| [0002](./adr-0002-go-primary-backend.md) | Go as the primary backend language | Accepted |
| [0003](./adr-0003-postgresql-and-sqlite.md) | PostgreSQL in the cloud, SQLite locally | Accepted |
| [0004](./adr-0004-modular-monolith.md) | Modular monolith over microservices | Accepted |
| [0005](./adr-0005-offline-first-outbox-and-idempotency.md) | Offline-first sync via outbox and idempotency | Accepted |
| [0006](./adr-0006-development-workflow.md) | VS Code + FreeBuff + Git/GitHub workflow | Accepted |
| [0007](./adr-0007-defer-premature-infrastructure.md) | Defer Kubernetes, Kafka and service extraction | Accepted |
| [0008](./adr-0008-security-and-tenant-isolation.md) | Security and shop-level tenant isolation baseline | Accepted |
| [0009](./adr-0009-github-actions-ci.md) | GitHub Actions for automated checks | Accepted |
| [0010](./adr-0010-configurable-inventory-and-units.md) | Configurable multi-industry inventory and units of measure | Accepted |
| [0011](./adr-0011-financial-transaction-integrity.md) | Financial transaction integrity, stock movements, returns and invoice snapshots | Accepted |
| [0012](./adr-0012-account-provisioning-and-offline-auth.md) | Account provisioning, credential reset, offline authentication and provider/shop isolation | Accepted |
| [0013](./adr-0013-invoice-numbering-and-tax-boundaries.md) | Invoice numbering and tax-compliance boundaries | Accepted |

The table above is the authoritative index. ADRs 0001–0009 predate the wider
product decisions recorded in 0010–0013; those newer records **extend** the
existing accepted decisions rather than superseding them. No accepted ADR is
currently superseded or amended. Where a record relates to another, it lists the
relationship in a `Related:` header line.

## Template

```markdown
# ADR-NNNN: Title

- Status: Proposed
- Date: YYYY-MM-DD
- Deciders: <names/roles>
- Supersedes: <ADR-NNNN or none>

## Context
What forces are at play? What is the problem?

## Decision
What we will do.

## Consequences
Positive, negative and follow-up implications.

## Alternatives considered
What we rejected and why.
```
