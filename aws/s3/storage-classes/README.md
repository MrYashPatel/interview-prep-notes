

---

# 📦 Amazon S3 Storage Classes

### 1. **S3 Standard**

* **Use case:** Default for frequently accessed data.
* **Durability:** 99.999999999% (11 nines).
* **Availability:** 99.99%.
* **Redundancy:** Data stored across **at least 3 Availability Zones (AZs)**.
* **Cost:** Most expensive of the active storage classes.
* **Example:** Website images, real-time analytics data, active logs.

---

### 2. **S3 Standard-IA (Infrequent Access)**

* **Use case:** Data that is accessed less often, but still needs to be quickly available when requested.
* **Durability:** 11 nines.
* **Availability:** 99.9%.
* **Redundancy:** Multi-AZ storage.
* **Cost:** Lower storage cost than Standard, but **retrieval fee per GB**.
* **Example:** Backups, older documents, disaster recovery data.

---

### 3. **S3 One Zone-IA**

* **Use case:** Infrequently accessed data, but **does not need multi-AZ resiliency**.
* **Durability:** 11 nines (within one AZ).
* **Availability:** 99.5%.
* **Redundancy:** Stored in **a single AZ only** (cheaper, but risky).
* **Cost:** \~20% cheaper than Standard-IA.
* **Example:** Secondary backups, easily reproducible data, temporary datasets.

---

### 4. **S3 Glacier**

(Archival class, long-term storage, low cost, retrieval delay)

* **Use case:** Archival data that you almost never need.
* **Durability:** 11 nines.
* **Availability:** Varies (you must restore before accessing).
* **Redundancy:** Multi-AZ.
* **Cost:** Very low storage cost, retrieval charges apply.
* **Retrieval times:**

  * Expedited: 1–5 minutes
  * Standard: 3–5 hours
  * Bulk: 5–12 hours
* **Example:** Compliance data, raw logs, historical records.

---

### 5. **S3 Glacier Deep Archive**

* **Use case:** Long-term archival (cheapest option), rarely accessed.
* **Durability:** 11 nines.
* **Availability:** After restore only.
* **Redundancy:** Multi-AZ.
* **Cost:** **Cheapest storage** in S3 (lower than Glacier).
* **Retrieval times:**

  * Standard: 12 hours
  * Bulk: up to 48 hours
* **Example:** Regulatory data you must keep for 7–10 years, medical records, old audit logs.

---

### 6. **S3 Intelligent-Tiering**

* **Use case:** When you **don’t know access patterns**. AWS automatically moves objects between storage tiers.
* **Durability:** 11 nines.
* **Availability:** 99.9%.
* **Redundancy:** Multi-AZ.
* **How it works:**

  * Object starts in the **frequent access tier**.
  * If not accessed for 30 days → moved to infrequent access tier (cheaper).
  * Can also transition into Glacier + Deep Archive automatically.
  * You pay a **small monitoring fee** per object.
* **Example:** Data lakes, ML training datasets, unpredictable logs.

---

# 📊 Quick Comparison

| Class                | Durability | Availability | AZs | Cost 💲  | Retrieval Time | Use Case                |
| -------------------- | ---------- | ------------ | --- | -------- | -------------- | ----------------------- |
| Standard             | 11 nines   | 99.99%       | 3+  | High     | Instant        | Hot data                |
| Standard-IA          | 11 nines   | 99.9%        | 3+  | Medium   | Instant (+fee) | Warm backups            |
| One Zone-IA          | 11 nines   | 99.5%        | 1   | Lower    | Instant (+fee) | Non-critical backups    |
| Glacier              | 11 nines   | 99.9%        | 3+  | Very Low | 1 min – 12 hr  | Archive, DR             |
| Glacier Deep Archive | 11 nines   | 99.9%        | 3+  | Lowest   | 12 – 48 hr     | Long-term archive       |
| Intelligent-Tiering  | 11 nines   | 99.9%        | 3+  | Flexible | Varies         | Unknown access patterns |

---

👉 Best Practices:

* **Hot data** → Standard
* **Backups** → Standard-IA / One Zone-IA (if non-critical)
* **Archives** → Glacier / Deep Archive
* **Unpredictable workloads** → Intelligent-Tiering

---
