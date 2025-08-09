# ADR-002: Core platform and media storage on AWS

 - Status: accepted
- Date: 2025-08-09

## Context
We are building an application to store and share resources (primarily multimedia: images, videos, and files). We need durable, cost-effective object storage, efficient global delivery, access control (private, shared via expiring links, public), and an extensible metadata model (users, orgs/teams, collections, ACLs, tags). We also foresee background processing for thumbnails, previews, and video transcoding.

Confirmed initial requirements:
- Region: primary `eu-central-1` (Frankfurt); EU residency acceptable; global delivery via CDN.
- Users: initially 1 uploader, potentially a few; design for many in future.
- Scale: up to ~100 videos overall, each up to ~500 MB; do not preclude scaling higher.
- Media types: accept broad file types; validation primarily for safety, not whitelist.
- Access model: default private; each file can be shared with specific users or groups.
- Retention: keep assets for at least 5 years; deletion supported (soft-delete + purge).
- Moderation: not required initially; keep room to add scanning/moderation later.
- Global viewers: worldwide; CDN handles performance; single primary region acceptable initially.

Assumptions to validate are listed in Open Questions.

## Decision (proposed)
Adopt a serverless-first architecture on AWS:
- Storage and delivery: Amazon S3 for originals and derivatives; Amazon CloudFront as CDN. Use pre-signed URLs and/or CloudFront signed URLs for authorized access.
- API and processing: Amazon API Gateway (HTTP) + AWS Lambda for API; EventBridge + SQS + Lambda workers for background processing; AWS Elemental MediaConvert for video transcodes.
- Metadata: Amazon RDS for PostgreSQL behind RDS Proxy; start Single-AZ `db.t4g.micro` to minimize cost, plan upgrade to Multi-AZ `db.t4g.small` as usage grows; start with Postgres full-text search, revisit OpenSearch only if needed.
- Auth: application-native email/password initially (e.g., Django auth when backend is ready). Authorization enforced in API using JWT claims and DB ACLs. Keep Cognito as an optional future enhancement if needed.
- Observability & security: CloudWatch Logs/metrics, AWS X-Ray, KMS encryption (S3, RDS, Secrets Manager), least-privilege IAM, VPC with private subnets and VPC endpoints.
- Delivery: SPA frontend on S3 + CloudFront (separate distribution from media).
- IaC & CI/CD: Terraform for infrastructure; GitHub Actions for build/deploy with manual approval for production.

## Rationale
- Pay-per-use and elasticity for unpredictable traffic.
- S3 provides durability and lifecycle controls (IA/Glacier) for cost optimization.
- CloudFront provides global edge caching and bandwidth savings.
- RDS Postgres fits relational access control and reporting needs.
- MediaConvert offloads heavy video processing reliably.

## Architecture Outline
- Buckets:
  - `assets-raw` (private, versioned, SSE-KMS): originals; lifecycle to Glacier as appropriate.
  - `assets-derivatives` (private by default, versioned, SSE-KMS): thumbnails, previews, transcodes; lifecycle to IA.
- Access & sharing:
  - Default private. Per-object ACL enforced at application layer (users/groups in Postgres).
  - Access via S3 pre-signed URLs and/or CloudFront signed URLs with short TTLs; renewable tokens.
- CDN:
  - CloudFront in front of `assets-derivatives` with Origin Access Control (OAC).
  - Private access via signed URLs/cookies; public assets via cacheable paths.
- Upload/download:
  - Client obtains S3 pre-signed URLs from API (multipart for large files); app servers do not proxy media streams.
- Processing pipeline:
  - S3 ObjectCreated -> EventBridge -> SQS -> Lambda workers (images/docs) and MediaConvert (videos).
  - Optional malware scanning (Lambda + ClamAV) and content moderation (Rekognition) on `assets-raw` (disabled initially, enable later if needed).
- Backend:
  - API Gateway + Lambda (Python/Node) for metadata CRUD, permissions, signed URLs, job orchestration.
  - RDS Postgres + RDS Proxy inside VPC; Lambdas connect via private subnets.
- Frontend:
  - SPA hosted on S3, served via separate CloudFront distribution.
- Security & ops:
  - KMS CMKs, Secrets Manager, IAM roles per service, CloudWatch dashboards/alarms, AWS Budgets.
  - Retention & lifecycle: versioning on; lifecycle transitions to IA/Glacier for cost; soft-delete via metadata tombstones; purge policy after ≥5 years (configurable).

## Low-usage cost snapshot (current expected usage)
Assumptions: ~20 GB total in S3, kilkanaście odsłon/mies., kilka uploadów, marginal egress (1–5 GB/mies.). EU region.
- S3 storage: ~0.4–0.6 USD/mies.
- CloudFront egress 1–5 GB: ~0.1–0.5 USD/mies.
- Requests (S3/CF): <0.10 USD
- API Gateway + Lambda: <0.50 USD
- RDS PostgreSQL (Single-AZ db.t4g.micro): ~13–18 USD/mies.
- MediaConvert: 0 USD, jeśli brak transkodowań; pojedyncze joby zwykle <1–2 USD/szt.
- Inne (CloudWatch/KMS/SQS/EventBridge/Route53): ~1 USD

Total: ~15–25 USD/mies. (bez transkodowań).

## Consequences
- Pros:
  - Minimal ops, strong durability, cost-effective delivery, clear separation between raw assets, derivatives, and metadata.
  - Scales from low to high traffic without re-architecture.
- Cons / Risks:
  - Lambda cold starts and DB connection storms (mitigated with RDS Proxy and provisioned concurrency on hot paths).
  - MediaConvert adds per-job costs and requires queue/monitoring.
  - Signed URL management increases key rotation/operational complexity.

## Alternatives Considered
1. ECS Fargate for API/workers
   - Pros: Long-lived connections, easier DB pooling.
   - Cons: Always-on cost and more ops; not needed initially.
2. DynamoDB for metadata
   - Pros: Serverless, highly scalable.
   - Cons: Complex relational permissions/queries harder; pushes logic into app.
3. Direct S3 access without CloudFront
   - Pros: Simpler, cheaper at very small scale.
   - Cons: Worse latency and egress costs; no edge caching.
4. Self-managed transcoding on Lambda/ECS
   - Pros: Avoid MediaConvert costs.
   - Cons: Operationally heavy; codecs/containers maintenance burden.
5. AWS CDK instead of Terraform
   - Pros: Great DX; AWS-native.
   - Cons: Team may prefer provider-agnostic Terraform; broader ecosystem.

## Initial AWS Resources (high-level)
- S3: `assets-raw`, `assets-derivatives` (versioning, SSE-KMS, lifecycle)
- CloudFront: `media-cdn` (OAC to derivatives), `web-cdn` (SPA)
- API Gateway (HTTP) + Lambda (API)
- EventBridge, SQS, Lambda workers (derivatives, extraction, moderation)
- MediaConvert: queue + IAM roles; outputs to `assets-derivatives`
 - RDS PostgreSQL (Single-AZ to start; plan Multi-AZ) + RDS Proxy
- Cognito User Pool (+ optional Identity Pool)
- VPC: private subnets, NAT as needed, VPC endpoints (S3, CW Logs)
- KMS CMKs; Secrets Manager
- CloudWatch dashboards/alarms; AWS Budgets

## Open Questions (please confirm)
1. Derivatives: image sizes, document preview formats, video rendition profiles (e.g., HLS variants) to generate.
2. Availability at MVP: confirm Single-AZ DB is acceptable (upgrade to Multi-AZ at growth).
3. Budget guardrails per month to tune caching and lifecycle aggressiveness.
