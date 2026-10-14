# RetailCore — Architecture

> Status: Draft · Last updated: 2026-10-14
> Derived from the RetailCore Final Technology and Development Strategy.
> Decisions are recorded individually in [`decisions/`](./decisions/).

## 1. Purpose and scope

RetailCore is an **offline-first retail billing and inventory platform**. It must
record sales, payments, inventory movements and customer balances reliably, keep
working when the internet is unavailable, and synchronise that work safely once
connectivity returns.

The system targets **Android, Windows desktop and the browser**, backed by a
single backend that is designed to scale without prematurely adopting
microservices.

This document describes the architecture. It is descriptive of intent, not a
claim that any component is already built.

## 2. Guiding principles

1. **Correctness before performance.** The hard problems are data contention,
   idempotency, concurrency control, isolation and recovery — not raw language
   speed. Architecture is optimised around financial and inventory correctness.
2. **Offline-first.** The shop keeps operating with no network. The network is
   treated as an unreliable, occasionally-absent resource.
3. **Boring, proven technology.** Prefer a modular monolith, one relational
   database and automated backups over distributed complexity until measurement
   demands more.
4. **Explicit over implicit.** Explicit SQL and transaction control, explicit
   idempotency, explicit tenant isolation, explicit resolution of sync edge cases.
5. **Human-reviewed automation.** AI assistance accelerates work; no generated
   change is trusted without review and tests.
6. **Scoped adoption of scale tooling.** Kubernetes, Kafka and service
   extraction are deferred until the product's measured workload requires them.

## 3. System context

```text
                    RETAILCORE
                        |
          +-------------+-------------+
          |             |             |
      Flutter         Flutter       Browser
      Android         Windows        Client
          |             |             |
          +-------------+-------------+
                        |
                    HTTPS API
                   (REST + OpenAPI)
                        |
                 Go Backend Service
               (modular monolith, stateless)
                        |
       +----------------+----------------+
       |                |                |
    Billing          Inventory         Ledger
       |                |                |
       +----------------+----------------+
                        |
                  PostgreSQL
                        |
              Backup and Recovery
```

The three-way split (Billing / Inventory / Ledger) represents **logical
responsibilities within one deployable service**, not separate microservices.

## 4. Clients

| Client | Technology | First target | Notes |
|---|---|---|---|
| Mobile application | Flutter + Dart | Android (first) | Primary pilot platform |
| Desktop application | Flutter desktop | Windows (second) | For shops needing a larger screen |
| Browser application | Flutter Web | Admin & reporting | Broader web functionality later |

Business logic and reusable components are shared across clients, but layouts are
**not** forced to be identical:

- **Mobile cashier:** large touch targets, fast product search, minimal
  navigation, rapid payment entry, readable receipts.
- **Desktop cashier:** keyboard shortcuts, barcode-scanner input, wider product
  table, persistent cart, more visible transaction detail.

Platform support does **not** guarantee plugin/peripheral parity. **Printer
compatibility and local database behaviour must be verified separately on each
target platform.**

### Rollout sequence

1. Android application (initial retail pilot).
2. Windows desktop application (large-screen shops).
3. Browser-based administration and reporting.
4. Broader web functionality once browser persistence, printing and offline
   behaviour are validated.

## 5. Backend

One deployable **Go** service organised as a **modular monolith**, exposing a
**modular, versioned REST API** described by **OpenAPI**.

### Modules (logical responsibilities)

- Identity and access control
- Business and shop management
- Product catalogue
- Billing and invoices
- Payments and customer ledger
- Inventory
- Synchronisation
- Reporting
- Subscription management

Modules have clear boundaries so they can be developed and tested independently —
without the networking, deployment and distributed-transaction cost of
microservices.

## 6. Data architecture

Two database layers exist because the product must survive without internet.

### Local database (per device)

- **SQLite via Drift** — type-safe persistence, SQL queries, transactions and
  migrations for Flutter across mobile, desktop and web.
- Stores the shop's **local operational records** and a durable **outbox** of
  operations awaiting the server.
- The exact backing store and its durability characteristics must be **verified
  per target platform**.

### Cloud database

- **PostgreSQL** — the primary source of synchronised business records; supports
  reporting, account management and cross-device access.
- Relational tables and constraints model invoices, invoice lines, payments and
  inventory movements.
- Related changes commit in **a single database transaction** — creating an
  invoice must never leave an invoice without its corresponding inventory
  movements because one operation failed.
- Access uses explicit SQL and transaction control (e.g. a maintained driver such
  as `pgx`), with **versioned schema migrations**.

### Sync model — the most important design decision

**We do not synchronise the entire local database blindly.**

- The client maintains a durable **outbox** of operations that must reach the
  server. Each operation carries a **stable identifier**.
- The backend records processed operation identifiers and enforces **idempotency**,
  so retrying a request never creates duplicate transactions.

The protocol must explicitly define behaviour for:

- The application closing during synchronisation.
- The network disappearing halfway through a request.
- A device submitting the same operation twice.
- A product's price changing while a device is offline.
- An invoice being reversed after another payment has been recorded.

**These cases must be resolved before multi-device offline operation is enabled.**

## 7. Scalability path

Scale in stages, driven by measurement — never by ambition alone.

### Stage 1 — Initial pilot
- One Go backend deployment
- One PostgreSQL database
- Local SQLite databases
- Automated backups
- Basic monitoring

### Stage 2 — Growing customer base
- Multiple **stateless** Go application instances
- Managed PostgreSQL with appropriately sized compute and storage
- Connection pooling
- Query and index optimisation
- Background workers for non-critical asynchronous work
- Improved monitoring and recovery procedures

### Stage 3 — Larger production workload
- Read replicas where reporting justifies them
- A durable job queue where background workload warrants one
- Analytical workloads separated from transactional workloads
- Partition large tables only when measurements justify it
- Extract services only with a clear operational or organisational benefit

## 8. Security baseline

RetailCore stores business records and potentially personal information about shop
customers, so security is part of the initial architecture:

- Server-side authorisation on every protected operation
- Explicit **shop-level data isolation** (every record tenanted)
- Secure storage of authentication credentials; **no custom cryptography**
- Encryption in transit
- Protection for local data and backups
- Audit records for important financial corrections
- Database constraints for critical relationships
- Idempotent payment and synchronisation operations
- **Tested** backup and restoration
- No production secrets committed to GitHub
- Automated dependency and code-security checks

Multi-tenant access enforces shop ownership in the service layer, with
**PostgreSQL row-level security** as an additional defence. We must test that one
shop cannot access another shop's records — including via guessed identifiers or
malformed requests.

## 9. Deployment and environments

- **Local development:** Docker Compose for reproducible local services
  (PostgreSQL and related dependencies).
- **Production hosting:** managed cloud compute and managed PostgreSQL; the
  specific provider is chosen once deployment requirements and cost are
  established.
- **CI/CD:** GitHub Actions runs tests and static analysis on pull requests.

## 10. Observability

- **Error tracking:** Sentry or an equivalent service — evaluate privacy, cost
  and free-tier limits before committing.
- **Product analytics:** privacy-conscious event tracking to measure activation,
  usage and retention.
- Distinguish **free-to-download tools** from **services that are free only at low
  scale**; hosting, monitoring, backups and messaging introduce recurring cost.

## 11. Repository structure

```text
retailcore/
├── apps/
│   ├── retail_app/           # Flutter Android/Windows client
│   └── admin_web/            # Optional browser administration
├── backend/
│   └── api/                  # Go API
├── database/
│   ├── migrations/           # PostgreSQL schema migrations
│   └── seeds/                # Development-only test data
├── contracts/
│   └── openapi.yaml          # API contract
├── infra/
│   ├── docker-compose.yml
│   └── deployment/
├── docs/
│   ├── architecture.md
│   ├── data-model.md
│   └── decisions/
├── scripts/
├── .github/
│   └── workflows/
├── .gitignore
└── README.md
```

## 12. Technology decisions

| Layer | Choice | ADR |
|---|---|---|
| IDE | VS Code | [0006](./decisions/adr-0006-development-workflow.md) |
| AI workflow | FreeBuff, human-reviewed | [0006](./decisions/adr-0006-development-workflow.md) |
| Source control | Git + GitHub | [0006](./decisions/adr-0006-development-workflow.md) |
| Mobile / desktop | Flutter + Dart | [0001](./decisions/adr-0001-flutter-for-clients.md) |
| Primary backend | Go | [0002](./decisions/adr-0002-go-primary-backend.md) |
| API | REST + OpenAPI | [0002](./decisions/adr-0002-go-primary-backend.md) |
| Cloud database | PostgreSQL | [0003](./decisions/adr-0003-postgresql-and-sqlite.md) |
| Local database | SQLite + Drift | [0003](./decisions/adr-0003-postgresql-and-sqlite.md) |
| Sync model | Outbox + idempotency | [0005](./decisions/adr-0005-offline-first-outbox-and-idempotency.md) |
| Architecture | Modular monolith | [0004](./decisions/adr-0004-modular-monolith.md) |
| Scale tooling | Deferred (no K8s/Kafka yet) | [0007](./decisions/adr-0007-defer-premature-infrastructure.md) |
| Security | Tenant isolation baseline | [0008](./decisions/adr-0008-security-and-tenant-isolation.md) |
| Automated checks | GitHub Actions | [0009](./decisions/adr-0009-github-actions-ci.md) |
| AI / forecasting | Python later, if justified | [0002](./decisions/adr-0002-go-primary-backend.md) |

## 13. Non-goals (for now)

- Microservices, Kubernetes, Kafka or a distributed architecture.
- A second backend language for ordinary billing operations.
- Full desktop feature parity in the browser on day one.
- Synchronising the entire local database.

## 14. Open questions

- Which managed cloud provider and managed PostgreSQL offering meets our cost and
  privacy requirements? (Deferred until production requirements are established.)
- Which identity provider or authentication library will we adopt?
- Which error-tracking and product-analytics services fit our free-tier and
  privacy constraints?
