
---

## **1. Critical Production Outage – Multi-layer Cause**

**Scenario**:

* A core banking API is down.
* Alert says **latency > 2s** and **error rate > 40%**.
* Grafana shows pod restarts in one microservice.
* Logs show `connection reset by peer` from Aurora DB.
* Aurora CPU is fine, but **DB connections are maxed out**.
* HPA scaling is failing.

**Challenges**:

* Root cause isn’t obvious: turns out **a GitHub Actions pipeline deployed a faulty migration** → new query triggered **N+1 selects** → overwhelmed DB connection pool → caused cascading failures → K8s restarted pods due to liveness probe failures → HPA couldn’t scale because IRSA policy was accidentally revoked.

**Skills tested**:

* Incident response under pressure.
* Reading logs + metrics in parallel.
* DB connection pooling knowledge.
* CI/CD rollback strategy.
* IRSA + IAM debugging.

---

## **2. Cross-account OIDC Trust Break**

**Scenario**:

* GitHub Actions OIDC trust between AWS Account A (CI) and Account B (prod) **randomly fails** during deployments.
* No changes in IAM roles according to CloudTrail.

**Root cause**:

* The GitHub Actions workflow was updated to use a **different environment name**, which broke the trust condition (`sub` claim mismatch in IAM trust policy).

**Challenge**:

* Requires deep understanding of AWS STS, OIDC claims, GitHub Actions contexts, and trust policies.

---

## **3. Ghost Traffic Causing AWS Bill Spike**

**Scenario**:

* Suddenly CloudFront + S3 bill goes up 4x overnight.
* No legitimate traffic spike in application analytics.

**Root cause**:

* Public S3 bucket being scraped by bots from multiple regions due to a leaked link in an old PDF document.

**Challenge**:

* Requires quick S3 Access Logs + Athena queries + WAF rate limiting + CloudFront signed URLs implementation **while live traffic continues**.

---

## **4. Argo CD Sync Order Deadlock**

**Scenario**:

* Multi-cluster Argo CD App-of-Apps setup.
* After upgrading Helm charts, half the apps are stuck in `Pending` forever.
* Manual `kubectl apply` works fine.

**Root cause**:

* A circular dependency in Helm values (`ConfigMap` from App A needed by App B, but App B’s CRD was applied before App A’s ConfigMap existed).

**Challenge**:

* Required redesign of sync waves + resource hooks + Helm post-renderers.

---

## **5. Terraform State Corruption Under Lock**

**Scenario**:

* Terraform Cloud backend.
* State locked due to an aborted run, and now every apply fails.
* Manual unlock not possible because of a **backend bug in an older Terraform version**.

**Root cause**:

* Needed to export state, manually edit JSON, and import again — without breaking existing infra.

**Challenge**:

* Deep Terraform backend knowledge + versioning quirks.

---

## **6. Kubernetes DNS Meltdown**

**Scenario**:

* Services randomly failing with `no such host`.
* Restarting CoreDNS fixes for 5 mins then fails again.

**Root cause**:

* An internal batch job was flooding DNS with thousands of short-lived lookups per second, exhausting CoreDNS cache and causing eviction loops.

**Challenge**:

* Requires **networking + K8s internals** + CoreDNS tuning (`cache`, `max_concurrent`).

---

## **Why These Are Difficult**

1. **Multiple systems involved** — Networking, IAM, CI/CD, observability, databases, K8s.
2. **No single log line tells the story** — you have to connect dots across tools.
3. **Time pressure** — usually during outages, with stakeholders waiting.
4. **Security & compliance constraints** — can’t just “test in prod” casually in fintech/banking.

---

