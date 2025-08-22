
---

## **1️⃣ Make a bucket public for static website hosting**

* **Enable Static Website Hosting** in bucket properties.
* **Set index.html / error.html**.
* **Disable Block Public Access** (careful!).
* Add **Bucket Policy** allowing `s3:GetObject` for `Principal: *`.

✅ Best practice: Use **CloudFront with OAC** instead of making S3 public.

---

## **2️⃣ Difference between S3 and EBS**

| Feature           | S3                            | EBS                      |
| ----------------- | ----------------------------- | ------------------------ |
| Type              | Object storage                | Block storage            |
| Access            | HTTP API                      | Attached to EC2 only     |
| Durability        | 11 nines                      | 1 AZ (snapshots → S3)    |
| Max Object/Volume | Unlimited / 5TB               | 16TB per volume          |
| Use Case          | Files, logs, backups, website | Database, OS disks, apps |

---

## **3️⃣ When to use Intelligent-Tiering vs Glacier**

* **Intelligent-Tiering** → unknown access patterns, instant retrieval.
* **Glacier/Deep Archive** → archival, rarely accessed, cheap, retrieval takes minutes/hours.

---

## **4️⃣ Pipeline uploads artifacts but restrict public access**

* IAM Role with **s3\:PutObject** only.
* Bucket Policy **blocks all public access**.
* Use **OIDC or STS roles** instead of static keys.

---

## **5️⃣ Serve static React website from S3 with HTTPS**

* **S3** → store React build.
* **CloudFront** → HTTPS, caching, OAC to hide bucket.
* **ACM** → SSL certificate.
* Optional: **Route53** → custom domain.

---

## **6️⃣ Generate one-time secure URL**

* **Pre-signed URL** via SDK or CLI:

```python
import boto3
s3 = boto3.client('s3')
url = s3.generate_presigned_url('get_object', Params={'Bucket':'my-bucket','Key':'file.txt'}, ExpiresIn=3600)
```

* Can also implement via **Lambda + API Gateway** for serverless access without credentials.

---

## **7️⃣ Enforce all objects encrypted with KMS**

* Enable **S3 default encryption → SSE-KMS**.
* Apply **Bucket Policy** to deny uploads without `x-amz-server-side-encryption: aws:kms`.
* Optional: use **S3 Access Points + IAM conditions**.

---

## **8️⃣ GitHub Actions (OIDC) upload to S3 in another account**

* Create **IAM Role in Account B** trusted by GitHub OIDC provider.
* Role has **S3 write permissions**.
* GitHub Actions uses `aws-actions/configure-aws-credentials` with role-to-assume.
* ✅ No static keys needed.

---

## **9️⃣ S3 uploads too slow for global clients**

* **S3 Transfer Acceleration** → leverages Amazon edge locations.
* Use **Multipart Upload** → parallel uploads.
* Upload from **nearest region / edge** if possible.

---

## **10️⃣ Compliance: no one can delete logs for 7 years**

* Enable **Object Lock** at bucket creation.
* **Retention Mode = Compliance**, **7 years**.
* Enable **Versioning**.
* Optional IAM deny policies as extra safeguard.

---

## **11️⃣ Client uploads 100M+ objects/day → bucket naming & partitioning**

* Avoid sequential or timestamp-only keys.
* Use **hash/random prefixes**: `a1b2c3/2025-08-22/file.log`.
* Logical folders for lifecycle/analytics: `region1/logs/2025/08/22/uuid.log`.
* Use **Transfer Acceleration + Multipart Upload**.

---

## **12️⃣ Pipeline GitHub Actions → S3 in Account B without keys**

* Same as **8️⃣ above** → OIDC + assume role.

---

## **13️⃣ S3 spike in GET requests from unknown IPs**

* **Block public access**.
* Restrict via **Bucket Policy / IAM policy** (allow only trusted IPs or roles).
* Use **CloudFront + WAF** → rate limiting / IP blocking.
* Enable **CloudTrail / S3 Access Logs** → monitor suspicious activity.

---

## **14️⃣ Uploads >10GB always fail**

* Single PUT limit = 5GB.
* Use **Multipart Upload** for large files.
* Boto3, AWS CLI, SDKs automatically handle >5GB files via multipart.

---

✅ **Summary Table (Quick Cheatsheet)**

| Scenario                       | Key Solution                                     |
| ------------------------------ | ------------------------------------------------ |
| Public website                 | Static Website Hosting + Policy / CloudFront OAC |
| S3 vs EBS                      | Object vs Block, S3 unlimited vs EBS AZ bound    |
| Intelligent-Tiering vs Glacier | Unknown access vs archival                       |
| Pipeline upload, no public     | IAM Role + Bucket Policy                         |
| React site HTTPS               | S3 + CloudFront + ACM + Route53                  |
| One-time secure URL            | Pre-signed URL or Lambda+API Gateway             |
| Enforce KMS encryption         | Default SSE-KMS + Bucket Policy                  |
| GitHub OIDC → S3               | IAM Role with trust to OIDC, no keys             |
| Global upload speed            | Transfer Acceleration + Multipart Upload         |
| 7-year WORM logs               | Object Lock Compliance Mode + Versioning         |
| High-scale uploads             | Randomized prefixes, hash, folders               |
| Sudden GET spike               | Block public, CloudFront + WAF, logging          |
| >10GB uploads                  | Multipart Upload                                 |

---

