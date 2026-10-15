# ADR-0013: Invoice numbering and tax-compliance boundaries

- Status: Accepted
- Date: 2026-10-15
- Deciders: RetailCore engineering
- Supersedes: none
- Related: ADR-0011 (financial integrity), ADR-0012 (account boundaries)

## Context

Invoices are legal documents. Their numbers must be unique, sequential within a
series and never silently reused, and they must survive offline operation. Indian
GST also imposes rules on invoice content, tax rates, place of supply, and — for
some businesses — e-invoicing. Getting these wrong has regulatory consequences,
and building full tax compliance is a large, specialist scope that must not be
claimed before it exists.

## Decision

### Tax configuration outside code

Keep **tax configuration separate from application code**. Model:

- Applicable business configuration.
- Tax classifications and rates with **effective dates**.
- **Inclusive/exclusive** pricing behaviour.
- **Intra-state / inter-state** treatment.
- Invoice **snapshots** of tax applied at finalization.
- Relevant **correction documents**.

### Compliance boundary

- We must **validate Indian GST invoice rules and merchant eligibility against
  current official guidance and a qualified tax professional** before asserting
  compliance.
- We must **not claim** GST filing, e-invoicing, IRN generation, government
  integration or full tax compliance **unless the functionality is implemented
  and validated**.

### Offline-safe invoice numbering (first release)

For the initial **single-primary-device** model:

- Numbers are drawn from a **document series scoped to the financial year**.
- The series boundary (financial-year start/reset) is **explicit and configured**,
  not implied by the clock.
- A **cancelled** invoice's number is **not silently reused**; cancellation is
  recorded and the number's status is preserved.
- Numbering must be **offline-safe**: a device that cannot reach the server must
  still produce valid, non-duplicating numbers within its own series.

### Redesign required before multiple offline devices

Enabling concurrent multi-device offline billing requires an explicit design for
**invoice-number allocation across devices** (contention, gaps and reservation),
alongside the stock-contention, conflicting-edit, transaction-ordering and
recovery work noted in ADR-0005 and ADR-0011. Until that exists, invoice numbering
is defined **only** for the single-primary-device case.

## Consequences

- Tax behaviour is data-driven and versioned, so rate changes do not require code
  releases and historical invoices keep the rates applied at the time.
- The numbering scheme is simple and defensible for one device, and its
  limitations are documented rather than hidden.
- Multi-device numbering is a **known future design task**, not an oversight.
- Compliance language in the product and documentation must stay conservative;
  marketing must not outrun implemented functionality.

## Alternatives considered

- **Global server-assigned sequential numbers:** rejected for offline operation;
  the device cannot obtain a number without the server.
- **Random/UUID invoice numbers:** rejected. Not acceptable as human-facing,
  auditable legal identifiers.
- **Numbering with silent reuse of cancelled numbers:** rejected. Risks duplicate
  legal documents.
- **Claiming GST/e-invoicing compliance without implementation:** rejected.
