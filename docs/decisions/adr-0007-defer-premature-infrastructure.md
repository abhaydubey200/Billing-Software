# ADR-0007: Defer Kubernetes, Kafka and service extraction

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

The long-term ambition is large, which makes it tempting to introduce
Kubernetes, Kafka and a distributed microservice architecture up front. Those
technologies add real cost and ongoing operational responsibility.

## Decision

**We will not introduce Kubernetes, Kafka or a distributed microservice
architecture merely because the long-term ambition is large.** Adopt them only
when the product's actual workload requires them.

Scale in measured stages:

**Stage 1 — Initial pilot**
- One Go backend deployment
- One PostgreSQL database
- Local SQLite databases
- Automated backups
- Basic monitoring

**Stage 2 — Growing customer base**
- Multiple **stateless** Go application instances
- Managed PostgreSQL with appropriately sized compute and storage
- Connection pooling
- Query and index optimisation
- Background workers for non-critical asynchronous work
- Improved monitoring and recovery procedures

**Stage 3 — Larger production workload**
- Read replicas where reporting requirements justify them
- A durable job queue where background workload warrants one
- Analytics separated from transactional workloads
- Partition large tables only when measurements justify it
- Extract individual services only with a clear operational or organisational benefit

## Consequences

- Lower cost and operational burden while the product is finding fit.
- We must keep the door open for scale: **stateless** services, clean module
  boundaries (ADR-0004) and a database that can be scaled/managed later.
- Adoption of each scale tool must be **triggered by measurement**, not by
  aspiration or fashion.

## Alternatives considered

- **Kubernetes/Kafka from day one:** rejected. Premature operational cost with no
  proven workload.
- **Microservices at launch:** rejected. See ADR-0004.
- **No scaling plan at all:** rejected. We keep the architecture ready to scale
  even while we defer the tooling.
