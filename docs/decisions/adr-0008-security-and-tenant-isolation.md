# ADR-0008: Security and shop-level tenant isolation baseline

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

RetailCore stores business records and potentially personal information about shop
customers. Multi-tenant data must not leak between shops. Security therefore has
to be part of the initial architecture, not a later retrofit.

## Decision

Adopt the following **baseline requirements**:

- Server-side authorisation on **every** protected operation.
- Explicit **shop-level data isolation**.
- Secure storage of authentication credentials.
- Encryption in transit.
- Appropriate protection for local data and backups.
- Audit records for important financial corrections.
- Database constraints for critical relationships.
- Idempotent payment and synchronisation operations.
- **Tested** backup and restoration.
- **No production secrets committed to GitHub.**
- Automated dependency and code-security checks.

For multi-tenant access:

- Enforce **shop ownership in the service layer**.
- Use **PostgreSQL row-level security** as an additional defence layer.
- **Test that one shop cannot access another shop's records** — including through
  **guessed identifiers or malformed requests**.

Authentication:

- Use an **established authentication library or identity provider**; avoid custom
  cryptography and homemade authentication protocols.

## Consequences

- Every query and endpoint must be tenanted by shop; this is a cross-cutting
  concern to be enforced centrally rather than repeated ad hoc.
- Authorisation decisions live on the server — clients are never trusted.
- RLS adds a second line of defence but does not replace service-layer checks.
- Requires test suites that specifically attempt cross-tenant access.
- Secret management and dependency scanning become part of CI (ADR-0009).

## Alternatives considered

- **Client-side-only authorisation:** rejected. Clients are untrusted.
- **Application-layer checks alone, no RLS:** rejected as the sole defence given the
  multi-tenant requirement; RLS is kept as defence-in-depth.
- **Custom auth/crypto:** rejected. High risk, no benefit over established solutions.
