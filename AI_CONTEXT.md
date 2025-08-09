# Engineering and Architecture Guidelines

This document captures general, technology-agnostic principles for building and operating this project. It intentionally avoids product-specific or stack-specific details.

## Principles
- **Clarity first**: Optimize for readability and maintainability over cleverness.
- **Small, cohesive modules**: Each module should have a single clear responsibility.
- **Explicit over implicit**: Prefer explicit configuration, contracts, and data flows.
- **Fail fast and loud**: Surface errors early; avoid silent failures and hidden side effects.
- **Security and privacy by default**: Minimize data collection; least-privilege access.
- **Measurable quality**: Use metrics and tests to validate behavior and performance.
- **Automate the basics**: Formatting, linting, tests, and builds should be automated.

## Code Quality Standards
- **Naming**: Use descriptive full words. Functions are verbs; variables/classes are nouns. Avoid abbreviations.
- **Structure & control flow**: Prefer guard clauses and early returns to reduce nesting.
- **Errors**: Prefer explicit error handling. No catch-and-ignore. Include context in error messages.
- **Types**: Use static typing/type hints where available. Avoid untyped escape hatches like `any`.
- **Comments & docs**: Document public APIs and non-obvious decisions. Avoid trivial comments.
- **Formatting**: Enforce a single formatter and linter. Keep code style consistent.
- **Tests**: 
  - Unit tests for core logic; integration tests for boundaries; e2e for critical flows.
  - Deterministic, isolated tests; avoid network/time dependencies unless mocked.
  - Aim for risk-driven coverage rather than a fixed percentage target.
- **Dependencies**: Pin versions, prefer fewer dependencies, and review licenses. Regularly update with a changelog.
- **Security basics**: Input validation, output encoding, secret management, least privilege, and secure defaults.

## Architectural Guidelines
- **Boundary-oriented design**: Separate domain logic from infrastructure (e.g., Clean/Hexagonal Architecture).
- **Contracts at boundaries**: Define clear interfaces between modules/services; use DTOs/schemas at I/O edges.
- **Configuration**: Externalize via environment variables with sane defaults. No secrets in code or repo.
- **Observability**: Structured logging (JSON in production), metrics, and tracing for critical paths.
- **Data management**: Versioned migrations, backups, and clear ownership. Idempotent migration scripts.
- **APIs**: Versioned, consistent, and documented. Support idempotency, pagination, and rate limiting where relevant.
- **Asynchrony & jobs**: Offload long-running tasks to background workers with retries and backoff.
- **Caching**: Explicit strategy (keys, TTL, invalidation). Validate cache correctness in tests.
- **Scalability**: Stateless services where possible; state in managed stores. Horizontal scaling first.
- **Internationalization & accessibility**: Design for i18n/l10n and basic a11y from the start if user-facing.

## Process & Workflow
- **Conventional Commits (no emojis required)**: `type(scope): subject`. One logical change per commit.
  - Types: feat, fix, docs, style, refactor, test, perf, ci, chore, build, revert.
- **Code review**: Small PRs, clear descriptions, and checklists. Require at least one approval.
- **CI/CD**: On every PR/push: format, lint, test, type-check, and build. Block merges on failures.
- **ADRs**: Record significant decisions in short Architecture Decision Records.
- **Definition of Done**: Code, tests, docs, and monitoring/alerts updated; feature behind a toggle if risky.

## ADR Template (short)
```
# ADR-XXX: Title

- Status: proposed | accepted | superseded by ADR-YYY | deprecated
- Date: YYYY-MM-DD
- Context: What problem or constraint led to this decision?
- Decision: What is decided? (be concise)
- Consequences: Positive/negative trade-offs and follow-up actions
- Alternatives considered: Briefly list rejected options and why
```

## Security & Compliance Baseline
- Threat model major components; prioritize mitigations.
- Handle PII/media metadata carefully; define retention and deletion policies.
- Secrets: use a vault/KMS; rotate regularly; never commit secrets.
- Dependencies: enable vulnerability scanning (SCA) and fix critical issues promptly.

## Pre-merge Checklist
- Formatting, linting, tests, and type checks are green.
- Backward compatibility considered (APIs, schemas, data migrations).
- Observability: logs/metrics/traces updated; alerts for critical paths.
- Security: inputs validated, permissions enforced, secrets handled correctly.
- Documentation updated (user-facing and developer docs as applicable).

