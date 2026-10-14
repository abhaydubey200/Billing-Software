# ADR-0001: Flutter for mobile and desktop clients

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

RetailCore must run on Android, Windows desktop and the browser. Maintaining
separate native codebases for each platform would multiply effort and allow the
clients to drift apart. We want to share business logic and reusable components
while still adapting layouts to the device.

## Decision

Use **Flutter + Dart** for the Android and Windows desktop clients, and
**Flutter Web** for browser-based administration and reporting.

Share logic and components across platforms, but **do not force every interface to
look identical**:

- Mobile cashier: large touch targets, fast product search, minimal navigation,
  rapid payment entry, readable receipts.
- Desktop cashier: keyboard shortcuts, barcode-scanner input, wider product
  table, persistent cart, more visible transaction detail.

Roll out in sequence: (1) Android, (2) Windows desktop, (3) browser admin and
reporting, (4) broader web functionality once browser persistence, printing and
offline behaviour are validated.

## Consequences

- One language and one UI toolkit for all clients; business logic is shared.
- We must maintain Dart alongside Go (see ADR-0002).
- Flutter officially supports these platforms, but **plugin and peripheral parity
  is not guaranteed**. Printer compatibility and local database behaviour must be
  **verified separately per platform**.
- Browser offline/persistence behaviour is treated as unproven until validated,
  which is why full web parity is a later stage.

## Alternatives considered

- **Native Android + Windows (Kotlin/Swift/WinUI or similar):** highest
  per-platform fidelity, but two-to-three separate UI codebases and far more
  maintenance.
- **Web-only (e.g. a JS framework):** simplest distribution, but weakest offline
  story, poor hardware/printer access, and unsuitable as the primary offline
  cashier client.
- **React Native:** viable, but weaker Windows desktop story than Flutter for our
  requirements.
