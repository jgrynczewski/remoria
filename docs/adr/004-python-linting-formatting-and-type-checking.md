# ADR-004: Python linting, formatting, and type checking

- Status: accepted
- Date: 2025-08-09

## Context
We need fast, consistent code quality tooling that is easy to run locally and in CI, with minimal configuration overlap and good autofix support. The project is a Django/DRF backend with tests and later infra code.

## Options Considered
1) Ruff (lint + format) + mypy
- Pros: Extremely fast; single tool for lint + format (reduces config drift); compatible with many Flake8 rules; autofix; modern ecosystem. mypy remains the de facto for static typing.
- Cons: `ruff format` is newer than Black (but mature for common cases). Some deep rules from pylint are intentionally out of scope.

2) Black + isort + Flake8 (+ plugins) + mypy
- Pros: Very mature; widely adopted; predictable formatting and lint rules.
- Cons: Multiple tools/configs; slower; plugin matrix maintenance; overlapping responsibilities (lint vs style) cause churn.

3) Pylint + Black + isort + mypy
- Pros: Deep static analysis; can enforce design smells.
- Cons: Noisy by default; slower; significant config to balance signal/noise.

## Decision
Adopt Ruff for linting and formatting, and mypy for type checking. Optionally add Bandit for security checks later.

- Lint/format: `ruff check` (with `--fix` locally) and `ruff format`
- Types: `mypy` with a pragmatic baseline and stricter settings over time
- Security (optional later): `bandit -r .`

## Rationale
- Speed and simplicity increase adoption and consistency.
- One source of truth (pyproject.toml) for style/lint rules.
- mypy remains best-in-class for Python type checking.

## Configuration Outline (pyproject.toml)
- ruff:
  - target-version = py311/py312 (TBD)
  - line-length = 100–120 (TBD)
  - select common rulesets (E,F,I,UP,TRY,PL,COM,ISC,PYI as needed)
  - per-file-ignores for migrations/tests if needed
- ruff format: enable; respect line-length
- mypy:
  - python_version 3.12 (TBD)
  - strict optional toggles (warn-redundant-casts, no-implicit-optional, disallow-any-generics)
  - ignore generated files (migrations)

## CI Pipeline
- `ruff format --check` and `ruff check`
- `mypy`
- block merge on failures

## Alternatives and Revisit Criteria
- If we require deeper code-smell analysis, revisit adding selective pylint rules.
- If `ruff format` conflicts with team preferences, consider switching to Black while keeping Ruff for lint.
