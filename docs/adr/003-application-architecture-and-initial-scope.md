# ADR-003: Application architecture and initial scope

- Status: proposed
- Date: 2025-08-09

## Context
We will implement the core application first, then wire the cloud infrastructure. The app manages multimedia assets with default-private visibility, per-user/group sharing, and long-term retention. Initial usage is small, but we should not block future scaling.

## Decision (proposed)
Backend stack:
- Framework: Django 5 + Django REST Framework (DRF)
- Auth: Django auth (email/password), JWT tokens for API
- DB: PostgreSQL
- Object storage integration: S3-compatible via `boto3` (pre-signed URLs)
- Config: `.env` + 12-factor style settings
- Background tasks (later): AWS-native (Lambdas) or Celery if needed locally; not required for MVP

Frontend (later): SPA (React or similar), static hosting; initial focus is backend API.

## Rationale
- Django+DRF provides rapid CRUD and robust auth/permissions out of the box.
- Postgres suits relational ACLs (users, groups, collections).
- S3 pre-signed URLs keep large media off the app servers.
- JWT for stateless API access; compatible with future Cognito/SSO if needed.

## Initial Domain Model (MVP)
- User: Django default
- Group: Django Groups (optional in MVP) or custom `Team`
- Asset:
  - id, owner (FK User), bucket, key, size, content_type, checksum, created_at
  - visibility: default private
  - metadata: JSON (EXIF, tags)
  - retention_until, soft_deleted_at
- Share:
  - id, asset (FK), subject_type (user|group), subject_id, permission (view|download), expires_at (optional)
- Derivative (later):
  - id, asset (FK), type (thumbnail|preview|video:profile), bucket, key, size, width/height, created_at

## Initial API (MVP)
- Auth
  - POST `/auth/login` → JWT
  - POST `/auth/register` (optional) or admin-provisioned users
- Assets
  - POST `/assets/upload-initiate` → {bucket, key, presigned_url, headers, part_size}
  - POST `/assets` (finalize) → create DB record after successful upload
  - GET `/assets/{id}` → metadata (owner or shared)
  - GET `/assets/{id}/download` → short-lived pre-signed URL (or CloudFront signed)
  - DELETE `/assets/{id}` → soft-delete
- Sharing
  - POST `/assets/{id}/share` → grant to user/group (with expiry optional)
  - DELETE `/assets/{id}/share/{share_id}` → revoke

Permissions:
- Owners have full control. View/download for explicitly shared subjects.

## Non-Functional Requirements (MVP)
- Default-private assets; no public listing
- Input validation and content-type checking (safety-first)
- Structured logging (JSON-ready); correlation ids in API
- Migrations with Alembic-equivalent not needed (Django migrations suffice)
- Basic rate limiting at API gateway level later

## Out of Scope (MVP)
- Video transcoding and image/document derivative generation
- Advanced search; start with simple filters
- Organizations/teams beyond basic groups

## Implementation Plan (high level)
1. Create Django project and DRF setup; settings via `.env` (Postgres URL, S3 bucket names, region)
2. Models: `Asset`, `Share`
3. JWT auth
4. Endpoints: upload-initiate, finalize, metadata read, download link, share grant/revoke
5. Basic tests (unit + API) and local `docker-compose.yml` for Postgres

## Alternatives Considered
- FastAPI + SQLAlchemy: leaner runtime, but more manual work for auth/permissions and admin
- Node/NestJS: good DX, but team familiarity and future Django admin argue for Django

## Open Questions
- Do we allow self-registration or admin-provisioned accounts only?
- Minimum password/2FA policy for MVP?
- Which JWT library preference (e.g., `djangorestframework-simplejwt`)?
