# ADR-0004: Modular monolith over microservices

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

RetailCore's long-term ambition is large (thousands, potentially hundreds of
thousands of shops). It is tempting to start with microservices. However,
microservices introduce network boundaries, distributed transactions, independent
deployment pipelines and operational burden — costs that a pre-product team should
not pay before the workload justifies them.

## Decision

Build the initial backend as **one deployable Go application with clearly
separated modules** (a **modular monolith**).

Modules:

- Identity and access control
- Business and shop management
- Product catalogue
- Billing and invoices
- Payments and customer ledger
- Inventory
- Synchronisation
- Reporting
- Subscription management

The Billing / Inventory / Ledger split in the architecture diagram represents
**logical responsibilities, not separate microservices**.

## Consequences

- We can develop and test modules independently **without** distributed networking
  and distributed-transaction complexity.
- Cross-module changes that require a single database transaction remain simple.
- Module boundaries must be **actively enforced in code** (packages, interfaces,
  dependency direction) or the monolith will erode into a ball of mud.
- Extraction into services later is possible, but only when there is a clear
  operational or organisational benefit (see ADR-0007).

## Alternatives considered

- **Microservices from the start:** rejected. Premature distribution cost with no
  proven benefit at our current scale.
- **Unstructured monolith:** rejected. Without enforced module boundaries the code
  becomes hard to reason about and hard to extract from later.
