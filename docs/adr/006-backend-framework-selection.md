# ADR-006: Backend framework selection

- Status: accepted
- Date: 2025-08-09

## Context
We need an API-first backend for managing multimedia assets with default-private access and granular sharing. Requirements: rapid delivery, solid auth/permissions, object-level ACLs, good typing/tooling, and easy future scaling.

## Options Considered

### Django + Django REST Framework (DRF)
- Pros:
  - Batteries-included: auth, admin, ORM, migrations; DRF adds serializers, permissions (incl. object-level), pagination, throttling, schema/docs, browsable API.
  - Fast delivery for CRUD-heavy, ACL-oriented apps; rich ecosystem; mature stubs (django-stubs, drf-stubs).
  - Easy JWT integration; good testability; stable long-term support.
- Cons:
  - Heavier than microframeworks; monolithic by default (can modularize by apps).

### FastAPI + SQLAlchemy
- Pros: Modern async-first; excellent Pydantic validation; great performance; explicit typing.
- Cons: More plumbing for auth/admin/permissions; object-level ACLs require custom design; migrations/admin not as turnkey.

### Node.js (NestJS/Express)
- Pros: Strong ecosystem; TS DX; good for real-time; many libraries.
- Cons: Non-Python stack (mismatch with rest of project/tooling); more work for ACLs/admin; different ops/runtime.

### Flask (or Flask-API)
- Pros: Minimal, flexible.
- Cons: Requires assembling many parts (auth, permissions, serialization, pagination, docs); slower delivery for our use case.

## Decision
Adopt Django 5 + DRF for the backend API.

- Auth: Django auth (email/password) with self-signup via email link; JWT for API
- DB: PostgreSQL (SQLite for local dev)
- Object storage: S3 via presigned URLs (`boto3`)
- Tests: pytest + pytest-django

## Rationale
- Best fit for ACL-driven, CRUD-centric app; lowest time-to-value; robust permissions model with DRF.
- Strong typing via stubs; integrates well with our chosen tooling (uv, ruff, mypy).
- Scales organizationally by Django apps; later async/offload via AWS where needed.

## Consequences
- Monolithic service initially; microservices only if/when warranted.
- Keep domain boundaries clean to ease future extraction if needed.

## Revisit Criteria
- If real-time requirements or extreme concurrency become primary, revisit async-first framework or specialized services.
- If admin-heavy workflows dominate, leverage Django admin/custom admin sooner.
