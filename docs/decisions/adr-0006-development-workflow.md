# ADR-0006: VS Code + FreeBuff + Git/GitHub workflow

- Status: Accepted
- Date: 2026-10-14
- Deciders: RetailCore engineering
- Supersedes: none

## Context

We need a consistent development environment, an AI-assisted coding workflow, and
disciplined source control from day one. AI-generated code can be fast but is not
trustworthy without review and tests.

## Decision

Standardise on the following tools and workflow.

| Tool | Role | Policy |
|---|---|---|
| VS Code | Editing, debugging, terminal | Primary development environment |
| FreeBuff | AI-assisted development | **Review generated changes; never trust generated code without tests** |
| Git | Version control | Small, meaningful commits |
| GitHub | Remote repository and collaboration | Private repository during early development |
| GitHub Actions | Automated checks | Tests and static analysis on pull requests (see ADR-0009) |
| Flutter SDK | Client development | Pin and document the project version |
| Go toolchain | Backend development | Pin and document the project version |
| Docker Compose | Local services | Reproducible development environment |
| PostgreSQL | Local and production database | Versioned schema migrations |
| Bruno (or equivalent) | API testing | Maintain repeatable request collections |
| Figma | Interface design | Optional; use when design complexity justifies it |
| Sentry (or equivalent) | Error monitoring | Evaluate privacy, cost and free-tier limits |

### Default Git workflow

1. Create an issue describing the requirement.
2. Create a feature branch.
3. Implement the smallest complete change.
4. Run tests and static analysis.
5. Review the code, **including AI-generated changes**.
6. Open a pull request.
7. Merge only after the required checks pass.

**The `main` branch must always remain in a known, testable state.**

## Consequences

- A predictable, documented environment for every contributor.
- AI acceleration is bounded by mandatory human review and tests.
- We must keep tool versions pinned and documented to keep builds reproducible.
- Distinguish **free-to-download tools** from **services that are free only at low
  scale** — hosting, monitoring, backups and messaging carry recurring cost.

## Alternatives considered

- **Unstructured AI-first workflow (no review gate):** rejected. Untested generated
  changes are a correctness risk for financial software.
- **Direct commits to `main`:** rejected. `main` must stay testable.
- **Other IDEs / merge strategies:** viable, but consistency matters more than
  preference here.
