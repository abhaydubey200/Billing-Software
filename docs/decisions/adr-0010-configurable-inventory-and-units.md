# ADR-0010: Configurable multi-industry inventory and units of measure

- Status: Accepted
- Date: 2026-10-15
- Deciders: RetailCore engineering
- Supersedes: none
- Related: ADR-0003 (PostgreSQL + SQLite), ADR-0011 (financial integrity)

## Context

RetailCore must serve different retail domains — garments (size and colour),
grocery (weight, loose and pack), hardware (piece, box, metre) — from **one
product and one database schema**. A separate application or a per-industry
schema would multiply maintenance, fragment the sync model and make the same
shop's data inconsistent across updates.

Products also need flexible, shop-defined attributes (fabric, brand, pack size,
fragrance) that are not knowable up front. At the same time, quantities must be
measurable in meaningful units: "2 boxes" and "500 g" cannot be compared or
converted unless the unit is well defined.

## Decision

Build a **shared inventory engine** with configurable, tenant-scoped metadata
rather than industry-specific code paths.

The relational model supports:

- Products and product types.
- **Sellable variants** with stable identifiers (a variant is the unit that is
  priced, stocked and sold).
- **Tenant-scoped** SKU and barcode uniqueness constraints (a code unique inside
  one shop, not globally).
- **Configurable attribute definitions** (a shop decides which attributes exist
  for its product types).
- **Validated custom attributes stored in PostgreSQL `JSONB`** where a fixed
  column would be inappropriate. Values are validated against the attribute
  definition before storage; `JSONB` is a storage detail, not a substitute for
  validation.
- Categories, prices, tax configuration, suppliers and stock receipts.
- Inventory **movements**, stock counts, adjustments and balances.
- Explicit tracking policies for units, lots, expiry and serial numbers **when
  those modes are actually supported** — never assumed from stored attributes.

### Units of measure

- Every unit has an explicit **measurement dimension** (count, mass, volume,
  length). Only units of the **same dimension** may be converted.
- Quantities use **decimal-safe arithmetic** with a documented precision;
  floating point is not acceptable for measured quantities.
- Configurable units include pieces, packs, boxes, grams, kilograms,
  millilitres, litres, metres and centimetres.
- **Conversion factors are explicit and versioned.** Historical transactions
  retain the conversion factor and quantity context used when the transaction was
  finalized — a later change to a factor must not rewrite history.
- **Free-text unit labels must not replace a validated unit catalog.**

### Guardrail on scope

Storing an attribute does **not** implement a sector workflow. We must not claim
that a vertical (for example, pharmacy batch recall or apparel size-curve
planning) is supported merely because its attributes can be stored.

## Consequences

- One schema and one inventory module serve many domains; adding a domain is
  configuration, not a fork.
- `JSONB` gives flexibility but demands application-level validation and
  indexing strategy; queries over `JSONB` must be designed deliberately.
- Unit conversion, decimal precision and factor versioning become **core,
  tested** logic, because every quantity and price calculation depends on them.
- Attribute definitions must be versioned or stable enough that historical
  records remain interpretable.
- Some advanced per-industry workflows are explicitly deferred, and the
  documentation must keep that distinction honest.

## Alternatives considered

- **Separate application/schema per industry:** rejected. Duplicated code and
  data models, fragmented sync and reporting.
- **Fixed wide columns for all possible attributes:** rejected. Unbounded schema
  growth, sparse tables, constant migrations.
- **Fully schemaless product documents:** rejected. Weakens relational integrity
  for pricing, tax and stock that financial records depend on.
- **Free-text units:** rejected. Makes conversion and reporting meaningless.
