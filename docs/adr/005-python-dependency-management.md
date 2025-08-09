# ADR-005: Python dependency management

- Status: accepted
- Date: 2025-08-09

## Context
We need reliable, reproducible dependency management with lockfiles for CI and deployment, good performance, and easy local workflows. Project scope includes Django/DRF app, tests, and later AWS integrations.

## Options Considered
1) uv (pip + venv replacement, ultra-fast)
- Pros: Very fast resolver/installer; deterministic lockfile (`uv.lock`); compatible with `pyproject.toml`; drop-in for modern Python builds; great for CI speed.
- Cons: Newer than Poetry; smaller ecosystem of docs/examples (growing quickly).

2) Poetry
- Pros: Mature; widely adopted; lockfile; good dev/prod groups; scripts.
- Cons: Slower; sometimes resolver quirks; heavier metadata management.

3) pip-tools (pip-compile + requirements.txt)
- Pros: Simple, transparent; integrates well with pip; reproducible pins.
- Cons: Two-step workflow (compile/sync); no native groups/markers UX; slower in large graphs.

4) Plain pip + requirements*.txt
- Pros: Minimal complexity.
- Cons: No lockfile by default; reproducibility and updates more manual.

## Decision
Adopt `uv` for dependency management and virtualenvs. Use `pyproject.toml` for metadata and groups (main/test/dev). Commit `uv.lock` for reproducible builds.

## Rationale
- Best-in-class speed, simple UX, reproducible lockfile; easy CI integration.
- Future-friendly and compatible with standard Python packaging.

## Workflow
- Setup: `uv sync`
- Add deps: `uv add django djangorestframework ...`
- Add dev deps: `uv add --group dev ruff mypy pytest`
- Run tools: `uv run ruff check`, `uv run mypy`, `uv run pytest`
- Update: `uv lock --upgrade`

## Alternatives and Revisit Criteria
- If team prefers Poetry’s UX, can switch with minimal changes (keep `pyproject.toml`).
- If CI environments lack uv preinstall and cold boots dominate, we can cache `.uv` to recover speed.
