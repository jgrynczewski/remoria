# ADR-001: Platform options comparison and decision (storage, delivery, compute)

- Status: accepted
- Date: 2025-08-09

## Context
We need a platform for storing and sharing multimedia (images, videos, arbitrary files) with: default-private assets, per-user/group sharing, global access, low ops overhead, and a path to scale beyond initial small usage (<=100 videos up to 500 MB each, but not limiting future growth). Retention 
policy: keep assets for at least 5 years; deletion supported (soft-delete and purge).

## Options Considered

### 1) On-premises / self-hosted (colo or own HW)
- Storage: Ceph/MinIO + object storage; Nginx for delivery; FFmpeg-based pipeline; Postgres on VMs.
- Pros: Full control, vendor-independent, potentially lower long-term cost at large scale.
- Cons: High upfront and ongoing ops; complex to ensure durability (replication), security, CDN, and global performance; slow to iterate.
- Fit: Overkill for our initial scale and team; highest operational risk.

### 2) AWS (Amazon Web Services)
- Storage/CDN: S3 (+ IA/Glacier), CloudFront; processing: Lambda/SQS/EventBridge/MediaConvert; metadata: RDS Postgres.
- Pros: Mature media tooling, widest ecosystem, excellent durability, strong CDN, pay-per-use; well-known patterns.
- Cons: Pricing complexity, potential vendor lock-in; some services (MediaConvert) can be costly per job.
- Fit: Very strong fit for media workloads; easy to start small and scale.

### 3) GCP (Google Cloud Platform)
- Storage/CDN: Cloud Storage, Cloud CDN; processing: Cloud Functions/Run + Transcoder API; metadata: Cloud SQL (Postgres).
- Pros: Transcoder API is robust; straight pricing; Cloud Run great DX for containers.
- Cons: CDN often fronted via external HTTP(S) LB; some regions/features less ubiquitous than AWS; IAM model can be complex.
- Fit: Strong alternative; parity for most needs; especially attractive if we prefer container-first (Cloud Run).

### 4) Azure
- Storage/CDN: Blob Storage, Azure CDN/Front Door; processing: Azure Functions + Media Services (legacy shifting), metadata: Azure Database for PostgreSQL.
- Pros: Enterprise integration, AD/B2C identity; Front Door is powerful.
- Cons: Azure Media Services is evolving/deprecating some features; tooling fragmentation; learning curve.
- Fit: Viable, especially for MS-centric orgs; slightly weaker media pipeline story today.

### 5) Alibaba Cloud
- Storage/CDN: OSS + Alibaba CDN; processing: Function Compute + MPS; metadata: ApsaraDB for PostgreSQL.
- Pros: Strong in APAC/China; competitive pricing.
- Cons: Weaker EU data residency story and ecosystem for our needs; docs/support may be harder.
- Fit: Lower priority unless we target China/APAC primarily.

### 6) Cloudflare-centric (R2 + Cloudflare Images/Stream, Workers, KV/D1)
- Storage/CDN: R2 (S3-compatible, zero egress to CF), CDN: Cloudflare; video: Cloudflare Stream; compute: Workers/Queues; metadata: D1/Postgres external.
- Pros: Excellent global CDN; simple egress model; Stream simplifies video delivery.
- Cons: R2 lacks some S3 features (eventing/lifecycle richness); Workers runtime constraints; Stream is opinionated.
- Fit: Great for CDN-heavy use; may require mixing with another cloud for DB/processing.

## Cost Snapshot (our initial scale)
Assumptions: 100 videos @ 500 MB (50 GB total), 10 GB images/files, 10k monthly views, 500 uploads/month, EU region. Egress 200 GB/month.
- Storage (object):
  - AWS S3 Standard ~ $0.023/GB-mo → ~$1.38/mo for 60 GB; IA/Glacier later reduces cost.
  - GCS Standard (EU) ~ $0.02–0.026/GB-mo → similar order.
  - Azure Blob Hot (EU) ~ $0.018–0.022/GB-mo → similar.
  - Cloudflare R2 ~ $0.015/GB-mo → ~$0.9/mo (egress to CF is $0).
- CDN egress (approx.):
  - CloudFront ~ $0.085/GB (first 10 TB, EU) → ~$17/mo.
  - Cloud CDN (EU) ~ $0.08–0.12/GB → ~$16–24/mo.
  - Azure CDN/Front Door ~ $0.08–0.12/GB → ~$16–24/mo.
  - Cloudflare CDN (Pro/Business) often bundled; effective egress $0 to CF edge, plan-dependent.
- Requests/ops: low (a few dollars across providers at our volume).
- Compute/API: serverless invocations likely <$5–10/mo initially.
- Transcoding: 
  - AWS MediaConvert on-demand per-minute (SD/HD); for short clips total likely $10–50/mo depending on volume/bitrates.
  - GCP Transcoder API similar order of magnitude.
  - Cloudflare Stream per-minute storage/encoding/streaming blends into plan-based pricing.

Conclusion: For our scale all major clouds are inexpensive; CDN egress dominates. Cloudflare has advantage on egress to their CDN (R2+CF), AWS/GCP/Azure are close.

## Decision
Select Track A (AWS-first, serverless) as the initial platform due to simplicity of a single cloud, mature media tooling, and a clear growth path. Track B (Cloudflare+GCP hybrid) remains a viable alternative if CDN egress economics/latency become dominant in the future.

Chosen stack (Track A): S3 + CloudFront; API Gateway + Lambda; RDS Postgres; MediaConvert; app-native email/password auth initially (see ADR-002).

Rationale: default-private assets with per-user/group sharing, low ops overhead, straightforward lifecycle/cost controls, and minimal monthly cost at our current usage.

## Low-usage cost snapshot (current expected usage)
Assumptions: ~20 GB total in S3, kilkanaście odsłon/mies., kilka uploadów, marginal egress (1–5 GB/mies.). EU region.
- S3 storage: ~0.4–0.6 USD/mies.
- CloudFront egress 1–5 GB: ~0.1–0.5 USD/mies.
- Requests (S3/CF): <0.10 USD
- API Gateway + Lambda: <0.50 USD
- RDS PostgreSQL (Single-AZ db.t4g.micro): ~13–18 USD/mies. (główny koszt stały)
- MediaConvert: 0 USD, jeśli brak transkodowań; pojedyncze joby zwykle <1–2 USD/szt.
- Inne (CloudWatch/KMS/SQS/EventBridge/Route53): ~1 USD

Suma: ~15–25 USD/mies. (bez transkodowań).

## Why not on‑prem now
Operational overhead and risk outweigh benefits at our current scale and team size.

## Open Questions to finalize the choice
1. Strong preference for single-cloud vs hybrid (CF+GCP)?
2. Data residency constraints (EU-only) and available regions.
3. Expected geographic distribution of viewers (to tune CDN POPs/providers).
4. Desire to minimize egress cost (pushes toward Cloudflare) vs minimize service sprawl (pushes toward AWS-only).

## Next Steps
- Proceed with ADR-002 (accepted) to implement detailed AWS design and cost controls.
