# ADR-0009: GitHub Actions for automated checks

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

The Git workflow (ADR-0006) requires that pull requests pass tests and static
analysis before merge, and that `main` always stays in a known, testable state.
Automated testing is required for every critical change, and offline-first sync
(ADR-0005) needs integration coverage.

## Decision

Use **GitHub Actions** to run automated checks on **pull requests**:

- **Go:** `go test` (unit and integration where feasible) and static analysis
  (`go vet` / linters).
- **Flutter/Dart:** Flutter tests, widget/integration tests for critical flows, and
  static analysis (`flutter analyze`).
- **Dependency and code-security checks** (ADR-0008).
- Checks must **pass before merge**.

## Consequences

- A machine-verified quality gate backs the review process.
- Integration tests need a runnable PostgreSQL (and possibly Docker) in CI.
- Flutter and Go toolchains must be **pinned** in workflows to stay reproducible
  (ADR-0006).
- CI runtime and Actions minutes are a (small) recurring cost to watch as the suite
  grows.

## Alternatives considered

- **Manual pre-merge testing only:** rejected. Not reliable enough for financial
  software.
- **Other CI providers (GitLab CI, CircleCI, etc.):** viable, but GitHub is already
  our repository host and keeps the workflow in one place.
