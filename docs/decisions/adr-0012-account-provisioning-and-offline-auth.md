# ADR-0012: Account provisioning, credential reset, offline authentication and provider/shop isolation

- Status: Accepted
- Date: 2026-10-15
- Deciders: RetailCore engineering
- Supersedes: none
- Related: ADR-0005 (outbox + idempotency), ADR-0008 (security and tenant isolation)

## Context

RetailCore has two very different kinds of user: **RetailCore Service Provider**
personnel who manage customer business accounts, and **shop owners** who operate
a shop. A shop owner must not be able to create their own account or reset a
forgotten password by themselves, because accounts are provisioned and recovered
through a controlled, auditable channel.

At the same time, the shop must keep working **offline**. That creates tension:
a device that cannot reach the server cannot be revoked instantly, yet we must
not weaken authentication or pretend revocation is immediate.

## Decision

**Two customer-facing account categories, with separate interfaces and
separately enforced permissions.**

### A. RetailCore Service Provider

- Authorized internal personnel who manage customer business accounts.
- Creates and provisions shop-owner accounts.
- Activates and suspends accounts.
- Initiates authorized password resets and revokes sessions.
- Manages support cases and service operations.
- Uses a **separate administrative interface** with separately enforced
  permissions.

### B. Shop Owner

- Uses the separate RetailCore retail application to operate the shop.
- **Cannot self-register** and **cannot initiate a forgotten-password reset**.
- Can change a **known** password while authenticated.
- Receives a **temporary initial credential** through an approved private channel
  and **must replace it during activation**.

### Credential rules

- **Never store passwords in plaintext and never make an existing password
  retrievable.** Store appropriate password hashes.
- A credential reset **invalidates the prior credential** and generates an
  **auditable event**.
- Provider personnel do **not** receive unrestricted access to every shop's
  financial or customer records by default. Support access has appropriate
  authorization, **limited scope** and **audit history**.

### Offline authentication and session material

- Activation and primary-device registration happen **online**.
- Authentication/session material is stored **securely on the device**; there is
  **no plaintext local password**.
- Password resets, session revocation and account suspension are enforced when
  the device next reaches the server. A **prolonged-disconnection policy**
  (documented) defines how long a device may continue to operate and what happens
  when it reconnects after suspension or revocation.
- **Never promise instant revocation on a device that cannot communicate with the
  server.**

## Consequences

- Sign-up and recovery flows differ from consumer SaaS: they are provider-driven
  and auditable.
- The provider console is a distinct application surface with its own access
  control and audit trail, not a super-user mode inside the shop app.
- Offline operation must be expressed as a **time-bounded privilege** with a
  documented policy, not as unlimited local access.
- Revocation, suspension and reset events must be representable in the sync
  protocol so a returning device reconciles its state.
- Cross-tenant and cross-boundary tests are required: provider scopes must be
  proven to be limited, and shops must be proven unable to reach one another's
  records (ADR-0008).

## Alternatives considered

- **Self-service sign-up and forgot-password email flow:** rejected. Conflicts
  with controlled provisioning and account ownership.
- **Shared super-user account inside the shop app:** rejected. No separation of
  duties, no auditable scoping.
- **Unlimited offline operation:** rejected. Leaves no enforcement path for
  suspension or revocation.
- **Provider access to all shop data by default:** rejected. Violates
  least-privilege and shop confidentiality.
