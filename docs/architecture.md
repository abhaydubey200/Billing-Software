# RetailCore — Architecture

> Status: Design accepted, **no application code implemented yet** · Last updated: 2026-10-15
> Decisions are recorded individually in [`decisions/`](./decisions/).
> Staged delivery is planned in [`implementation-roadmap.md`](./implementation-roadmap.md).

## 1. Purpose and scope

RetailCore is an **offline-first, multi-tenant retail billing and inventory
platform** for Indian businesses. It must record sales, payments, inventory
movements and customer balances reliably, keep working when the internet is
unavailable, and synchronise that work safely once connectivity returns.

The first production target is **Android**; **Windows desktop** and browser
functionality follow after platform-specific validation.

This document describes the **intended architecture**. It is a statement of design
intent, **not a claim that any component is already built**.

## 2. Design intent vs. implemented code

This distinction is deliberate and must be maintained everywhere.

- **Implemented code in this repository today:** documentation only. There is no
  Flutter application, no Go backend, no database schema or migration, and no
  generated scaffolding.
- **Designed but not implemented:** all clients, the Go backend, the databases.
  The CI workflow (`.github/workflows/ci.yml`) detects missing projects and
  skips their jobs cleanly.
- **Frozen decisions:** the technology and product decisions in §7–§18 are
  decided; the records in [`decisions/`](./decisions/) are the authoritative
  source for *why*.
- **Not decided / open:** items listed in §21.

Every future claim about capability must state which of these it is. Storing an
attribute never implies a workflow is implemented; a documented rule never
implies the code enforces it yet.

## 3. Guiding principles

1. **Correctness before performance.** The hard problems are data contention,
   idempotency, concurrency control, isolation and recovery — not raw language
   speed. Architecture is optimised around financial and inventory correctness.
2. **Offline-first, not offline-capable.** The shop commits locally first; the
   network is treated as unreliable and occasionally absent.
3. **Boring, proven technology.** A modular monolith, one relational database and
   automated backups over distributed complexity until measurement demands more.
4. **Explicit over implicit.** Explicit SQL and transaction control, explicit
   idempotency, explicit tenant isolation, explicit conflict handling.
5. **Human-reviewed automation.** AI assistance accelerates work; no generated
   change is trusted without review and tests.
6. **Honest scope.** Never claim compliance, verification or industry support that
   the code does not provide.

## 4. System context

```text
                  RETAILCORE
                      |
      +---------------+----------------+
      |               |                |
   Flutter         Flutter          Browser
   Android         Windows          client
  (first target)  (second)         (later)
      |               |                |
      +---------------+----------------+
                      |
                  HTTPS API
             REST /api/v1 + OpenAPI
                      |
               Go Backend Service
             (modular monolith, stateless)
                      |
      +---------------+----------------+
      |               |                |
   Billing        Inventory         Ledger
      |               |                |
      +---------------+----------------+
                      |
                 PostgreSQL
               (RLS defence in depth)
                      |
              Backup and Recovery

  Separate: Provider Console (RetailCore personnel)
            — its own interface, permissions and audit trail
```

The Billing / Inventory / Ledger split represents **logical responsibilities
within one deployable service**, not separate microservices (ADR-0004).

## 5. Clients

| Client | Technology | Sequence | Notes |
|---|---|---|---|
| Shop application | Flutter + Dart | **Android first** | Primary pilot platform |
| Shop application | Flutter desktop | Windows second | Larger screen for shops that need it |
| Browser client | Flutter Web | Later, after validation | Only once browser persistence, printing and offline behaviour are validated |
| Provider console | Separate administrative interface | Alongside the backend | Used by RetailCore personnel only (§11) |

Business logic and reusable components are shared, but layouts are **not** forced
to be identical:

- **Mobile cashier:** large touch targets, fast product search, minimal
  navigation, rapid payment entry, readable receipts.
- **Desktop cashier:** keyboard shortcuts, barcode-scanner input, wider product
  table, persistent cart, more visible transaction detail.

Platform support does **not** guarantee plugin/peripheral parity. **Printer and
scanner compatibility must be validated on real supported devices**, and local
database behaviour verified separately per platform. **Manual product lookup must
always remain available even when barcode scanning works** (ADR-0001).

## 6. Repository structure (intended)

```text
retailcore/
├── apps/
│   ├── retail_app/           # Flutter Android/Windows client
│   └── admin_web/            # Provider console (browser)
├── backend/
│   └── api/                  # Go API (modular monolith)
├── database/
│   ├── migrations/           # PostgreSQL schema migrations
│   └── seeds/                # Development-only test data
├── contracts/
│   └── openapi.yaml          # API contract (/api/v1)
├── infra/
│   ├── docker-compose.yml
│   └── deployment/
├── docs/
│   ├── architecture.md
│   ├── implementation-roadmap.md
│   └── decisions/
├── scripts/
├── .github/
│   └── workflows/
├── .gitignore
└── README.md
```

None of these application directories exist yet; the tree shows where they will
live once the [roadmap](./implementation-roadmap.md) stages begin.

## 7. Technology decisions (frozen)

| Layer | Decision |
|---|---|
| Client | **Flutter + Dart** |
| First production target | **Android** |
| Later targets | **Windows desktop**, then browser after validation |
| Local database | **SQLite through Drift** |
| Backend | **Go modular monolith** |
| Cloud database | **PostgreSQL** |
| Flexible product attributes | **Validated PostgreSQL `JSONB`** where appropriate |
| API | **Versioned REST under `/api/v1`**, with an **OpenAPI contract** |
| Local development | **Docker Compose** |
| CI | **GitHub Actions** |
| Synchronization | **Durable local outbox, stable operation identifiers, server-side idempotency, explicit conflict handling** |
| Architecture style | **Modular monolith** — no microservices for the initial release |

**Explicitly excluded** without a future evidence-based decision:
MongoDB, Redis, Kafka, Kubernetes, AI features, and any additional database.

These are architecture decisions, **not claims that their implementation already
exists**. Each maps to an ADR:

| Decision | ADR |
|---|---|
| Flutter + Dart clients, Android first | [0001](./decisions/adr-0001-flutter-for-clients.md) |
| Go primary backend, REST + OpenAPI | [0002](./decisions/adr-0002-go-primary-backend.md) |
| PostgreSQL cloud + SQLite/Drift local | [0003](./decisions/adr-0003-postgresql-and-sqlite.md) |
| Modular monolith over microservices | [0004](./decisions/adr-0004-modular-monolith.md) |
| Offline-first outbox + idempotency | [0005](./decisions/adr-0005-offline-first-outbox-and-idempotency.md) |
| Development workflow (VS Code, FreeBuff, Git/GitHub) | [0006](./decisions/adr-0006-development-workflow.md) |
| Defer Kubernetes/Kafka/service extraction | [0007](./decisions/adr-0007-defer-premature-infrastructure.md) |
| Security and tenant isolation baseline | [0008](./decisions/adr-0008-security-and-tenant-isolation.md) |
| GitHub Actions CI | [0009](./decisions/adr-0009-github-actions-ci.md) |
| Configurable inventory and units of measure | [0010](./decisions/adr-0010-configurable-inventory-and-units.md) |
| Financial transaction integrity and snapshots | [0011](./decisions/adr-0011-financial-transaction-integrity.md) |
| Account provisioning, offline auth, isolation | [0012](./decisions/adr-0012-account-provisioning-and-offline-auth.md) |
| Invoice numbering and tax boundaries | [0013](./decisions/adr-0013-invoice-numbering-and-tax-boundaries.md) |

## 8. Product and account boundaries

RetailCore has exactly **two customer-facing account categories** (ADR-0012).

### A. RetailCore Service Provider

Authorized internal personnel who manage customer business accounts:

- Create and provision shop-owner accounts.
- Activate and suspend accounts.
- Initiate authorized password resets and revoke sessions.
- Manage support cases and service operations.
- Use a **separate administrative interface** with **separately enforced
  permissions**.

### B. Shop Owner

- Uses the separate RetailCore retail application to operate the shop.
- **Cannot self-register** and **cannot initiate a forgotten-password reset**.
- Can change a **known** password while authenticated.
- Receives a **temporary initial credential** through an approved private channel
  and **must replace it during activation**.

### Credential and access rules

- Passwords are **never stored in plaintext** and an existing password is **never
  retrievable**; appropriate password hashes are stored.
- A credential reset **invalidates the prior credential** and generates an
  **auditable event**.
- Provider personnel do **not** get unrestricted access to every shop's financial
  or customer records by default. Support access has appropriate authorization,
  **limited scope** and **audit history**.

## 9. Tenant model and initial-release constraints

RetailCore is designed for **multiple independent businesses**. One tenant must
**never** be able to read or modify another tenant's data.

**Initial release constraints:**

- One owner account per shop.
- One operational location per shop.
- One registered primary device per shop.
- Android first.
- English and Hindi.
- Local billing and inventory operations continue during temporary network
  outages.

Tenants, locations, users and devices are modelled **separately** so future
growth does not require redesigning the identity and data model.

**Concurrent multi-device offline billing is disabled for the first release.** The
future multi-device design must explicitly address **stock contention, invoice
numbering, conflicting edits, transaction ordering and recovery** before it is
enabled (§15, §16, ADR-0005, ADR-0011, ADR-0013).

## 10. Backend

One deployable **Go** service organised as a **modular monolith**, exposing a
**versioned REST API under `/api/v1`** described by an **OpenAPI contract**
(ADR-0002).

### Modules (logical responsibilities)

- Identity and access control
- Business and shop management
- Product catalogue
- Billing and invoices
- Payments and customer ledger
- Inventory
- Synchronisation
- Reporting
- Subscription and entitlements
- Provider console operations

Modules have clear boundaries so they can be developed and tested independently —
without the networking, deployment and distributed-transaction cost of
microservices (ADR-0004).

## 11. Provider console

The **provider console** is a separate interface for RetailCore personnel. It
supports:

- Account management (create, provision, activate, suspend).
- Credential provisioning and password reset initiation, with session revocation.
- Support cases.
- Entitlements.
- Service health.
- Audit history.

It is **not** a super-user mode inside the shop application, and it does not grant
unrestricted access to shop financial or customer records (ADR-0012). Support
actions are scoped and audited.

## 12. Data architecture

Two database layers exist because the product must survive without internet
(ADR-0003).

### Local database (per device)

- **SQLite via Drift** — type-safe persistence, SQL queries, transactions and
  migrations for Flutter across mobile, desktop and web.
- Stores the shop's **local operational records** and a durable **outbox**.
- The exact backing store and its **durability characteristics must be verified
  per target platform** — do not assume identical behaviour on Android, Windows
  and web.

### Cloud database

- **PostgreSQL** — the primary source of synchronised business records; supports
  reporting, account management and cross-device access.
- Relational tables and constraints model invoices, invoice lines, payments and
  inventory movements.
- **Validated `JSONB`** stores flexible product attributes where a fixed column is
  inappropriate (§13).
- Related changes commit in **a single database transaction** — creating an invoice
  must never leave an invoice without its corresponding inventory movements
  because one operation failed.
- Access uses **explicit SQL and transaction control** (e.g. `pgx`), with
  **versioned schema migrations**.
- **PostgreSQL row-level security** is enabled as **defence in depth**, alongside
  service-layer tenant checks (ADR-0008).

## 13. Configurable multi-industry inventory

One shared inventory engine serves different retail domains **without a separate
application or schema per industry** (ADR-0010).

The relational model supports:

- Products and product types.
- **Sellable variants** with stable identifiers.
- **Tenant-scoped** SKU and barcode constraints.
- Configurable attribute definitions.
- **Validated custom attributes stored in `JSONB`** where appropriate.
- Categories, prices, tax configuration, suppliers and stock receipts.
- Inventory movements, stock counts, adjustments and balances.
- Explicit tracking policies for units, lots, expiry and serial numbers **when
  those modes are actually supported**.

Examples: garments with size and colour; grocery sold by weight or pack; hardware
sold by piece, box or metre.

> **Scope guardrail:** supporting the *attributes* of a sector does not implement
> its workflows. Advanced sector-specific workflows are not claimed unless built
> and validated (ADR-0010, §20).

### Units of measure

- Every unit has an explicit **measurement dimension** (count, mass, volume,
  length); only **compatible** units may be converted.
- Quantities use **decimal-safe arithmetic** and a **documented precision**.
- Configurable units include pieces, packs, boxes, grams, kilograms, millilitres,
  litres, metres and centimetres.
- **Conversion factors are explicit and versioned**, and **historical
  transactions retain the conversion and quantity context used when finalized**.
- **Free-text unit labels must not replace a validated unit catalog.**

## 14. Transaction and financial integrity

**Stock is recorded as traceable movements**, not merely a mutable quantity field
(ADR-0011).

Modelled entities include: finalized invoices, line items, payment records, payment
allocations, customer credit, returns, refunds, exchanges, credit notes, stock
receipts and adjustment history.

### Required rules

- **Finalized invoices preserve historical snapshots** of product, unit, price,
  discount, tax and amount.
- Financial calculations use **decimal-safe arithmetic** and an **explicitly
  documented rounding policy**.
- **Corrections are traceable**: reversal, cancellation, return or adjustment
  records. Finalized transactions are **not silently overwritten or deleted**.
- **Negative stock is blocked by default**; any override is explicit,
  permission-controlled and audited.
- A **return** references the original sale, cannot exceed its remaining eligible
  quantity, and records whether returned stock is **restocked or excluded from
  sellable stock**.
- An **exchange** is represented by **linked return and sale transactions**.
- **Credit balances are derived** from ledger entries and allocations, not
  arbitrary editable balance fields.
- Recording **UPI, card or cash as a tender method does not mean the payment has
  been independently verified**.
- Local invoice, inventory, payment, credit and outbox effects are **atomic** where
  they form one completed sale; **cloud-side related effects commit
  transactionally**.

### Costing

The first release uses a **simple inventory-costing strategy** (for example,
moving-average or latest-cost) for **basic margin reporting**. This is explicitly
**not** full accounting-grade valuation; the distinction must remain visible in
reporting language (ADR-0011).

## 15. Tax, billing documents and invoice numbers

Tax configuration is kept **separate from application code** (ADR-0013). The model
covers: applicable business configuration, tax classifications and rates with
effective dates, inclusive/exclusive pricing behaviour, intra-state/inter-state
treatment, invoice snapshots, and relevant correction documents.

### Compliance boundary

Indian GST invoice rules and merchant eligibility **must be validated against
current official guidance and a qualified tax professional**. RetailCore must
**not claim** GST filing, e-invoicing, IRN generation, government integration or
full tax compliance **unless the relevant functionality is implemented and
validated**.

### Offline-safe invoice numbering (single-primary-device)

For the initial single-primary-device model:

- Numbers come from a **document series scoped to the financial year**, with an
  explicit, configured series boundary.
- A **cancelled** invoice's number is **not silently reused**.
- Finalized invoice numbers must **never be silently reused**.
- Numbering must be **offline-safe**, producing valid non-duplicating numbers
  within the device's own series.

### Redesign required before multiple offline devices

Enabling multiple offline devices requires an explicit design for
**invoice-number allocation across devices** (allocation, gaps, reservation) plus
the stock-contention, conflicting-edit, ordering and recovery work in §9 and §16.
Until then, numbering is defined only for the single-primary-device case.

## 16. Offline authentication, transactions and synchronization

Offline-first is a core product requirement, not just a cached interface
(ADR-0005, ADR-0012).

The design addresses:

- **Online** owner activation and primary-device registration.
- **Secure local storage** of authentication/session material.
- **No plaintext local passwords.**
- **Local transaction commit before the app reports a sale as saved.**
- **Durable outbox operations with stable identifiers.**
- **Safe retries** after timeouts, application restarts and lost responses.
- **Server-side idempotency** and **stored retry results**.
- **Explicit handling of conflicts** involving master-data changes and transaction
  ordering.
- **Visible statuses:** saved locally, pending synchronization, synchronized,
  needs attention.
- How **password resets, session revocation and account suspension** behave while
  the device is offline.
- A documented **prolonged-disconnection and device-replacement policy**.

**Silent last-write-wins is forbidden** for invoices, payments, customer ledger
entries and stock movements. Financial corrections must remain traceable.

**We never promise instant revocation on a device that cannot communicate with
the server** (ADR-0012).

The protocol must define behaviour **before multi-device offline operation is
enabled** for: app close during sync; network loss mid-request; duplicate
submission; price change while offline; and reversal after a later payment.

## 17. Security and SaaS isolation

Baseline controls (ADR-0008, ADR-0012):

- **Server-side authorization for every protected operation.**
- **Tenant-scoped queries and relationships.**
- **PostgreSQL row-level security** as defence in depth.
- **Cross-tenant access tests** covering guessed IDs and malformed requests.
- **Separate service-provider and shop-owner access boundaries.**
- **Least-privilege provider access** and **audited support actions**.
- **TLS**, managed secrets, appropriate **local-data protection** and protected
  **backups**.
- **Rate limiting** and secure session handling.
- **Audit records** for important financial corrections, credential events and
  privileged account operations.
- **Data export, account closure, retention, backup and restoration** procedures.
- **No production secrets in source control or diagnostic output.**

## 18. Main shop application workflows

Planned screen groups (design intent; none implemented yet):

1. **Home dashboard** — sales totals, bill count, payment summaries, customer
   credit, low stock and **sync status**.
2. **Billing** — product search, SKU/barcode lookup, variant selection, cart,
   discounts, customer selection, payments, bill finalization, **receipt
   recovery**, PDF/share and printer configuration.
3. **Inventory** — products, variants, units, stock receipts, stock counts,
   adjustments and stock history.
4. **Customers / Udhaar** — customer profiles, outstanding balances, ledger and
   partial payments.
5. **More** — sales history, returns/exchanges, reports, suppliers,
   import/export, backups, printer settings, language, security and
   synchronization support.

Hardware and printer compatibility must be **validated on real supported
devices**, and **manual product lookup must always remain available** even when
barcode scanning is available.

The separate **provider console** workflows are described in §11.

## 19. Deployment and environments

- **Local development:** Docker Compose for reproducible local services
  (PostgreSQL and related dependencies).
- **Production hosting:** managed cloud compute and managed PostgreSQL; the
  specific provider is chosen once deployment requirements and cost are
  established (§21).
- **CI/CD:** GitHub Actions runs tests and static analysis on pull requests
  (ADR-0009). Until the Go and Flutter projects exist, the workflow detects their
  absence and skips those jobs cleanly.

## 20. Scope boundaries

The initial release must **not falsely claim**:

- Concurrent multi-device offline billing.
- Complete support for every industry.
- Full accounting or bank reconciliation.
- Automatic UPI/card payment verification.
- Government e-invoice/IRN integration without implementation.
- Universal printer/scanner compatibility.
- Unlimited offline operation without a defined account/session policy.

The architecture stays **extensible** while unproven or high-cost capabilities are
**deferred** (ADR-0007).

## 21. Unresolved operational and regulatory decisions

Recorded openly; **not invented here, and not to be presented as settled**:

1. **Tax compliance validation** — Indian GST invoice rules, rates by category and
   merchant eligibility must be confirmed against **current official guidance and
   a qualified tax professional**. E-invoicing/IRN applicability thresholds are
   unresolved.
2. **Rounding and costing policy** — the exact rounding rule and the chosen simple
   costing method must be confirmed by the business/finance owner.
3. **Prolonged-disconnection and device-replacement policy** — the maximum offline
   operating window, re-authentication cadence and replacement/transfer procedure
   are not yet fixed.
4. **Managed cloud provider and managed PostgreSQL offering** — deferred until
   production requirements and cost are established.
5. **Authentication library or identity provider** — not yet selected (ADR-0008).
6. **Error-tracking and product-analytics services** — candidates exist; privacy,
   cost and free-tier limits must be evaluated (ADR-0006).
7. **Data retention, export and account-closure durations** — required for the
   security baseline; specific durations are a business/legal decision.
8. **Multi-device design** — invoice numbering, stock contention and conflict
   resolution rules for concurrent devices remain to be designed (ADR-0013).
9. **Printing/hardware matrix** — the supported device and printer list must be
   established through real-device validation.

## 22. Non-goals (for now)

- Microservices, Kubernetes, Kafka or a distributed architecture.
- MongoDB, Redis, or any additional database without an evidence-based decision.
- A second backend language for ordinary billing operations.
- Full desktop/browser feature parity at launch.
- Full accounting-grade valuation or bank reconciliation.
- AI features in the initial release.
- Synchronising the entire local database.
