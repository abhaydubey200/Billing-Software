# ADR-0002: Go as the primary backend language

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

The backend's central workload is transactional: create invoices, record payments,
maintain customer balances, record inventory movements, synchronise offline
transactions, enforce business-level permissions, generate reports and serve many
shops and devices. Language choice affects throughput, deployment, hiring,
operational complexity and the future AI roadmap.

A language does not make an application scalable by itself. Transaction design,
database constraints, synchronisation, observability, deployment and operational
discipline determine success.

## Decision

Use **Go** as the primary production backend, exposing a **modular, versioned REST
API described by OpenAPI**.

Reasons Go fits RetailCore's transactional workload:

- Compiled performance and a straightforward deployment model (single binary).
- Built-in concurrency (goroutines) suited to network services.
- First-class support for database transactions and cancellation of database
  operations in the standard library and ecosystem.
- Static typing and a small surface area that support disciplined modular design.

**Retain Python** for future forecasting, analytics and AI — not as a second
backend for ordinary billing.

**Do not create a second backend service until a concrete requirement justifies
it.**

## Consequences

- We maintain Go and Dart (see ADR-0001) rather than using Python throughout.
  Acceptable if the repository stays organised, the architecture is documented and
  duplication is avoided.
- Go's AI/ML library ecosystem is limited relative to Python; specialised
  analytics/AI work will live in Python services when the need is real.
- Performance is not automatically solved by the language — correctness depends on
  transaction design and operational discipline.

## Alternatives considered

| Criterion | Go | Python | Java/Kotlin | TypeScript/Node |
|---|---|---|---|---|
| API performance | Excellent | Good | Excellent | Good–excellent |
| Memory efficiency | Generally strong | Higher overhead | JVM-dependent | Runtime-dependent |
| AI/ML ecosystem | Limited vs Python | Excellent | Good | Good |
| Type safety | Static | Hints not enforced | Static | Static (TS) |
| Best fit | **Primary backend** | Analytics/AI | Viable | Viable |

- **Python everywhere:** strong if AI features come early, but the core product is
  retail transaction software and should be optimised for financial/inventory
  correctness first.
- **Java/Kotlin or TypeScript/Node:** viable alternatives, rejected in favour of Go's
  simplicity, deployment model and concurrency fit.
