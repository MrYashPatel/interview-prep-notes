
---

# Core—Fast Cheatsheets (instances, DB, queues, caches)

## Instance family cheatsheet (AWS; analogous in GCP/Azure in brackets)

* **General APIs (HTTP/REST/GraphQL):** `c7g.large–xlarge` (Graviton3 ARM; cheap & fast). (GCP: `c3-standard` / Azure: `Dpsv5`)
* **Memory-heavy (feed fan-out, serializers):** `r7g.large–xlarge`. (GCP: `m3` / Azure: `Esv5`)
* **Throughput/network heavy (WebRTC SFU, load-balancers, websockets, proxies):** `c7gn` (400 Gbps ENA). (GCP: `c3-highmem` w/ TDP NIC / Azure: `Fsv2` with accelerated networking)
* **Background jobs/ETL/queues:** `m7g.large–xlarge` (balanced), Spot for batch.
* **Media encoding/ML vision (GPU):** `g5.xlarge/g6.xlarge`. (GCP: L4 / Azure: A10/A16)
* **ML inference (CPU cost-efficient):** `inf2.xlarge` (Inferentia2). (GCP: TPU v5e / Azure: ND)
* **High IOPS local NVMe (indexers, Kafka):** `i4i.large–xlarge`. (GCP: `n2d-local-ssd` / Azure: `Lsv3`)
* **Postgres/MySQL on EC2 (rare; managed is better):** `r7g.xlarge` w/ io2 EBS.

> Rule: default to **Graviton (g)** unless a lib forces x86; use **c7g** for APIs, **r7g** for memory, **c7gn** for network, **g5/g6** for GPU.

## Storage/EBS

* **General DB/data nodes:** `gp3` (baseline 3,000 IOPS, 125 MB/s); bump IOPS/throughput as needed.
* **Critical DB:** `io2` or `io2 Block Express` (predictable low latency).
* **Kafka/OpenSearch hot tier:** instance store (i4i) + `gp3/io2` for logs/snapshots.
* **S3 classes:** Standard (hot), IA (warm), Glacier Instant/Deep (archive). Enable **S3 Object Lock** for compliance/WORM where needed.

## Databases (what/why)

* **Aurora PostgreSQL/MySQL:** relational, ACID, multi-AZ; pick **Postgres** if you want extensions/analytics; **MySQL** if simpler/cheaper ecosystem.
* **DynamoDB:** ultra-low latency key-value; global tables; perfect for carts, sessions, feature flags, idempotency.
* **OpenSearch (Elastic):** full-text, aggregations; for search/observability.
* **Redis (ElastiCache):** in-memory cache/rate-limiter/queues/pub-sub; use cluster mode if > 1 shard.
* **Neptune / graph:** social graphs, recommendations (optional).
* **Kafka (MSK):** event backbone, exactly-once streams; or **SQS/SNS** for simple queues/pubs.
* **ClickHouse/BigQuery/Redshift:** analytics at scale; not in request path.

## Queues/Streams

* **SQS:** simple worker queues, DLQ, visibility timeout.
* **SNS:** fan-out pub-sub.
* **MSK Kafka:** high-throughput streams (order events, feed fan-out).
* **Kinesis:** managed streaming; simpler ops than Kafka; good for telemetry.

## CDN & Edge

* **CloudFront:** TLS, signed URLs/cookies, **Origin Shield**, Function\@Edge/Lambda\@Edge for auth/cookies/AB tests.
* **Global Accelerator:** anycast TCP/UDP accel for latency-sensitive (games/meet).

---

# Cross-Cloud mapping (quick)

AWS → GCP → Azure
EKS → GKE → AKS
Aurora → Cloud SQL AlloyDB → Azure DB for PG/MySQL/Flexible
DynamoDB → Cloud Bigtable/Firestore → CosmosDB
MSK → Pub/Sub + Dataflow / Kafka on Confluent → Event Hubs/Kafka
ElastiCache Redis → Memorystore → Azure Cache for Redis
OpenSearch → Elastic Cloud/Opensearch on GKE → Elastic on AKS
CloudFront → Cloud CDN → Azure CDN/Front Door

---

# CI/CD + Security (baseline you’ll reuse everywhere)

* **CI:** GitHub Actions/GitLab → Lint/Unit → Build (Docker Buildx) → SBOM & scan (Trivy/Grype, Snyk) → Test (integration/contract) → Sign image (cosign) → Push (ECR/GHCR)
* **CD:** Argo CD (GitOps) + Argo Rollouts/Flagger (canary/blue-green) + manual gate for regulated envs.
* **IaC:** Terraform modules + remote state (S3+DynamoDB lock).
* **AuthN/Z:** Cognito/Auth0/OIDC; IRSA for pods; least-privilege IAM; org SCP guardrails.
* **Perimeter:** WAF managed rules + rate limits; Shield Advanced where needed.
* **Obs:** Prometheus + Grafana; Loki/ELK; OpenTelemetry traces → Tempo/Jaeger; SLO burn alerts.
* **Secrets:** AWS Secrets Manager/Vault; never in env; rotation jobs.
* **Compliance:** CloudTrail all-regions, AWS Config + Security Hub; S3 Object Lock for logs.

---

# Website Types → Sub-types → Deep Infra (with exact picks)

## 1) E-Commerce

**Sub-types:** B2C store, B2B portal, C2C marketplace.

### Traffic model

Browse (read-heavy) + checkout (write-critical), spikes on sales. Strong consistency for orders/payments, eventual for catalog/search.

### Reference architecture (AWS)

* **Frontend:** Next.js SPA/SSR → S3 + CloudFront (+ Lambda\@Edge for SEO/cookies).
* **API layer (microservices):** EKS.

  * **Node groups:** `c7g.large` (general APIs), `r7g.large` (pricing/inventory), `i4i.large` (search indexers), min 3 nodes/3 AZs.
* **Datastores:**

  * **Aurora PostgreSQL** (orders, users, payments) – writer + 2 readers; `db.r7g.large` start.
  * **DynamoDB** (cart/session, inventory reservations, idempotency) – on-demand or provisioned + auto-scaling.
  * **OpenSearch** (catalog search) – 3x `r7g.large.search` hot nodes, 2x `t3.small` masters to start.
  * **Redis/ElastiCache** – 3-node cluster mode (sessions/cache) `cache.r7g.large`.
* **Media:** S3 (images); on-upload Lambda for thumbnails; signed URLs.
* **Messaging:** SQS (order events) + SNS fan-out to email/ERP; optional MSK Kafka for real-time analytics.
* **Edge:** CloudFront with long TTL on static, short on `/api/*` bypass.
* **Security:** WAF rules (SQLi/XSS/common bots), Shield Adv on sales seasons.

### CI/CD & Ops

* **Canary only on checkout/payment**, feature flags elsewhere.
* **Load test:** k6—RPS browse 90%, checkout 10%; test p95 < 200 ms for browse, < 400 ms for checkout.
* **DR:** Aurora Global Database (reader in secondary region), S3 CRR; Route53 failover.

### Pitfalls

Hot Dynamo partitions (use random suffix), thundering herd on flash sales (cache stampede protection), idempotency tokens for payment/order create.

---

## 2) Streaming

### a) Video (VOD + Live)

* **Encoding:** FFmpeg/MediaConvert on `g5.xlarge` (H.264/HEVC) for VOD; **MediaLive** for Live.
* **Storage:** S3 with HLS/DASH; lifecycle to IA/Glacier.
* **API/Meta:** EKS (`c7g.large`) for catalog/entitlement.
* **DB:** DynamoDB (assets, entitlements), Aurora PG (users/billing).
* **CDN:** CloudFront + Origin Shield; signed cookies/URLs; DRM (Widevine/PlayReady/FairPlay).
* **Realtime telemetry:** Kinesis → Firehose → S3 → Athena/QuickSight.

**Obs/SLO:** startup time, rebuffer ratio, average bitrate; alarms on rebuffer > 1% / region.

**Gotchas:** CORS + Range headers at edge; pre-warm CF on premieres; multi-CDN optional.

### b) Audio (music/podcasts)

* **Compute:** API microservices `c7g.large`.
* **DB:** DynamoDB (playlists, likes) + DAX for hot keys; Aurora (accounts).
* **Transcode:** `c7g` CPU ok; GPU not mandatory.
* **CDN:** longer TTL, client cache control.

---

## 3) Gaming

### a) Real-time multiplayer (browser/native)

* **Game servers:** EKS pool `c7gn.large` (high PPS) or **GameLift** fleets.
* **Matchmaking:** service + DynamoDB; Redis for lobby state.
* **Persistence:** Aurora (transactions), S3 (replays).
* **Edge:** Global Accelerator to nearest region POP; WAF + Shield.
* **Obs:** tick rate, RTT, packet loss; PDBs to avoid evictions during matches.

**Gotchas:** UDP via NLB; pod anti-affinity to spread matches; bot-based load tests.

### b) Distribution store (downloads/patcher)

* **CDN:** CloudFront large files enabled; S3 multipart upload/resume.
* **License service:** EKS `c7g.small`—signed manifests + device binding.

---

## 4) Communication/Collab

### a) Video Conferencing (WebRTC – Meet-like)

* **Signaling:** API Gateway (WebSocket) → EKS `c7g.small`.
* **Media SFU:** Janus/mediasoup/Jitsi on **`c7gn.xlarge`** (network-optimized); region per continent.
* **TURN/STUN:** coturn on `c7g.small` w/ autoscale; 443/TCP fallback.
* **State:** Redis (presence), DynamoDB (rooms), Aurora (orgs/users).
* **Edge:** Global Accelerator; multi-region active-active.

**SLOs:** join time < 2s; p95 RTT < 150 ms; packet loss < 2%.
**Gotchas:** ICE/Firewall traversal; prioritize UDP; autoscale by active streams not CPU only.

### b) Chat/Slack-like

* **Gateway:** WebSocket/API via ALB.
* **Backbone:** MSK Kafka (fan-out) on `m7g.large` brokers or use serverless MSK; SQS for side-queues.
* **DB:** DynamoDB (messages by channel-partition), OpenSearch (search), S3 (attachments).
* **Obs:** delivery latency < 200 ms; backlog age; DLQ drains.

---

## 5) Banking/Fintech

### a) Retail NetBanking

* **Network:** private-only EKS; ingress via Private ALB behind API Gateway + WAF; no public DB.
* **Compute:** `m7g.large` app nodes; HSM ops separate.
* **Data:** **Aurora PostgreSQL** `db.r7g.large` (writer + readers), **pgbouncer** if needed; **KMS + CloudHSM** for keys/signing.
* **Integrations:** PrivateLink to PSP/KYC; VPC endpoints for S3/Dynamo/STS.
* **Audit:** CloudTrail org, Security Lake/SIEM; logs to S3 with Object Lock (WORM).

**CI/CD:** change approvals, segregation of duties; CodeQL/semgrep + DAST; signed images + OPA policy.
**DR:** warm-standby cross-region; RPO ≤ 5 min, RTO ≤ 60 min (decide with biz).

**Gotchas:** PCI scope minimization; tokenization; device fingerprint fraud rules.

### b) Payments Gateway/Wallet

* **API:** ALB + EKS `c7g.large`; p95 < 150 ms.
* **Idempotency:** Redis/Dynamo TTL keys.
* **Ledger:** Aurora PG strict constraints + serializable txn for critical ops.
* **Webhooks:** SQS + DLQ; retry/backoff; reconciliation batch (Athena/Glue).

### c) Brokerage/Trading (optional)

* **Market data:** Kinesis/MSK streams; snapshot cache `r7g.large`.
* **Orders:** low-latency path `c7gn.large`; exactly-once semantics + sequence IDs.

---

## 6) Social Networking

**Sub-types:** general (FB), professional (LinkedIn), photo/video (IG).

* **Services:** user, identity, graph, feed, media, notifications, search.
* **Compute:** APIs `c7g.large`; feed workers `r7g.large`; media pipeline `c7g`.
* **Graph:** DynamoDB adjacency lists **or** Neptune (graph).
* **Feed:** fan-out-on-write via Kafka + Redis lists **or** fan-out-on-read via Dynamo queries.
* **DB:** Aurora PG (accounts/payments), DynamoDB (feed/items), Redis (sessions/feed hot).
* **Search:** OpenSearch (user/post).
* **Push:** SNS + APNS/FCM.

**SLOs:** timeline p95 < 250 ms; notification delivery < 2s.
**Gotchas:** privacy/erasure workflows; spam/abuse ML; media thumbnailer on upload.

---

## 7) Informational/News

**Sub-types:** encyclopedia, newsrooms, blogs.

* **Headless CMS:** Strapi/Ghost/Contentful; publish → S3 → CloudFront (SSG).
* **Compute:** SSR pages on `c7g.small` if dynamic; else static.
* **Search:** OpenSearch; sitemap jobs.
* **Cache:** edge-heavy (stale-while-revalidate); purge by surrogate keys.

**Spikes:** elections/sports—pre-warm; enable Origin Shield; disable heavy JS during spikes.

---

## 8) Educational

### a) MOOC

* **Video stack:** like streaming VOD; DRM optional.
* **LMS APIs:** EKS `c7g.large`.
* **DB:** Aurora PG (courses, grades), DynamoDB (progress), Redis (sessions).
* **Proctoring:** WebRTC `c7gn.large` pool; sensitive storage separate.

### b) Institutional LMS

* **Multi-tenant:** per-org config; rate limits per tenant.
* **Integrations:** SAML/OIDC; SCORM/xAPI; webhooks via SQS.

---

## 9) Portfolio/Personal

* **Static:** S3 + CloudFront; CI on push → invalidation.
* **Zero-ops:** cost peanuts; uptime ping only.

---

## 10) Government Portals

**Sub-types:** public services (tickets/tax), info portals.

* **Network tiers:** Internet → WAF/Front Door → DMZ ALB → App (EKS) → Data (Aurora/RDS) in isolated subnets; east-west SG policies.
* **Access:** SSM Session Manager (no SSH); break-glass only.
* **Data:** S3 with Object Lock, Glacier retention; e-sign service HSM.
* **Queues:** SQS for workflows; batch windows for heavy jobs.
* **Compliance:** WCAG, audit trails, change advisory board.

---

# Tools—What they are & why you’d use them (crisp)

**Terraform** – Infra as code; reproducible VPC/EKS/RDS/WAF/CDN.
**Helm/Kustomize** – Kubernetes packaging & env overlays.
**Argo CD** – GitOps CD; desired state sync; audit of changes.
**Argo Rollouts/Flagger** – Canary/blue-green with metric gates.
**GitHub Actions/GitLab CI** – CI pipelines; build/test/scan/sign.
**Trivy/Grype/Snyk** – Container + deps vulnerability scanning.
**Checkov/tfsec/TFLint** – IaC misconfig scanning.
**CodeQL/semgrep** – SAST on PRs.
**OpenTelemetry** – Standard tracing/metrics/logs instrumentation.
**Prometheus** – Metrics scrape; HPA signals.
**Grafana** – Dashboards/alerts; SLO burn calc.
**Loki/ELK** – Logs aggregation + search.
**Jaeger/Tempo** – Distributed traces.
**k6/Locust** – Load tests; script user journeys.
**OWASP ZAP** – DAST; baseline scans pre-prod.
**Vault/Secrets Manager** – Central secrets, rotation.
**WAF/Shield** – Layer-7 firewall, DDoS defense.
**CloudTrail/Config/Security Hub/GuardDuty** – Audit, drift, posture, threat intel.
**Global Accelerator** – Anycast routing for low-latency TCP/UDP.
**MediaConvert/MediaLive** – Transcoding & live pipelines.
**GameLift** – Managed game servers & scaling.
**MSK Kafka/SQS/SNS/Kinesis** – Streams/queues; event backbones.
**OpenSearch** – Search + analytics for text/logs.

---

# Networking & Edge (details you’ll need)

* **VPC per env**, 3 AZs; public (ALB/NLB) + private (EKS) + isolated (DB/Redis).
* **VPC endpoints** (S3, STS, ECR, Secrets Manager) to keep traffic private.
* **NAT GW:** 1/AZ; consolidate where safe (cost).
* **MTU:** 9001 on EC2; cluster CNI supports jumbo frames for throughput jobs.
* **Ingress:** ALB Ingress Controller on EKS; NLB for UDP (games/RTC).
* **Service mesh (optional):** Istio/App Mesh for mTLS, traffic shifting, policy.
* **DNS:** Route 53; weighted/latency/health-check policies; failover plans.

---

# Observability (per-type KPIs)

* **E-com:** checkout success %, p95 API latency, Redis hit ratio, payment decline reasons.
* **Streaming:** startup delay, rebuffer %, bitrate distribution, CDN hit ratio.
* **Gaming:** tick rate, RTT, packet loss, lobby fill time.
* **Meet:** join success %, MOS, jitter, TURN ratio.
* **Banking:** auth success %, fraud flags, ledger lag, RDS replica lag.
* **Social:** feed p95, write amplification, spam detection rate.
* **News:** CDN hit %, origin latency, publish-to-live time.
* **LMS:** video watch-through, quiz latency, proctoring alerts.

---

# Security essentials (don’t skip)

* **TLS 1.2+**, HSTS, CSP; signed cookies/URLs for assets.
* **JWT/OIDC** for API; rotate signing keys; device fingerprints where relevant.
* **IRSA** for pod AWS creds; deny node metadata to pods.
* **Admission control** (OPA/Kyverno): image signatures, non-root, resource limits.
* **Backups** tested restores; periodic DR game-days.
* **Least-priv SG/IAM**; egress restricted; no public DBs—ever.

---

# Capacity & Testing recipes

* **E-com:** simulate flash sale (10–20× baseline); pre-scale HPA minReplicas; cache stampede test.
* **Streaming:** CDN offload goal > 95%; soak test long sessions; origin shield on.
* **Gaming:** 1,000 synthetic bots/region; AZ drain chaos; UDP packet loss injection.
* **Meet:** multi-party (3/9/25 tiles) tests; SFU CPU/net thresholds; TURN-only drills.
* **Banking:** TPS on core flows, serializable tx contention tests, webhook storm.
* **Social:** write burst to fan-out; rebuild feed backfill under load.

---

# Ready-to-use “starter matrix”

| Component     | Default pick                      | When to change                                         |
| ------------- | --------------------------------- | ------------------------------------------------------ |
| API compute   | `c7g.large` on EKS                | heavy JSON → `c7g.xlarge`; network-intense → `c7gn`    |
| Cache         | Redis `cache.r7g.large` (cluster) | > 60k ops/s → add shards; persistence needed → add AOF |
| Relational DB | Aurora PG `db.r7g.large`          | many small writes → MySQL; geo-readers → Global DB     |
| KV store      | DynamoDB on-demand                | stable traffic → provisioned + auto scale              |
| Search        | OpenSearch 3× `r7g.large.search`  | heavy aggregations → add `i4i` hot tier                |
| CDN           | CloudFront + Origin Shield        | ultra low RTT games/RTC → Global Accelerator           |
| Queue         | SQS + DLQ                         | high-throughput/streams → MSK Kafka                    |
| Edge auth     | Lambda\@Edge/CF Function          | complex AB/cookies → Lambda\@Edge                      |

---

# What this gives you

* Types + sub-types covered: **E-com, Streaming (VOD/Audio), Gaming (Realtime/Store), Comm (Meet/Chat), Banking (Retail/Payments), Social, News, Educational (MOOC/LMS), Portfolio, Government**.
* For each: **workload pattern → exact infra picks (instances/DB/cache) → CI/CD → SLOs → pitfalls**.
* **Tool dictionary** with crisp “what & why”.