
---

## 1. **EFS vs EBS vs S3 → When to use?**

**EBS (Elastic Block Store)**

* Block storage attached to a single EC2 instance (like a virtual hard disk).
* Use case: Databases, boot volumes, transactional workloads needing low latency.
* Limitation: Tied to one AZ (though you can snapshot/restore elsewhere).

**EFS (Elastic File System)**

* Managed NFS (POSIX-compliant file system), shared across multiple EC2, EKS pods, or Lambda.
* Use case: Shared storage for applications, CMS, microservices needing concurrent read/write.
* Scales automatically, pay-per-use.

**S3 (Simple Storage Service)**

* Object storage (not block or file).
* Highly durable (11 9’s), globally accessible.
* Use case: Backups, static website hosting, big data, logs, images, artifacts.
* Not for low-latency transactional workloads.

👉 **Rule of Thumb:**

* **EBS** → single instance, high-performance workloads.
* **EFS** → multiple instances/pods need shared file access.
* **S3** → large-scale, unstructured object storage.

---

## 2. **Can EFS be used for Windows?**

* **No** ❌ → EFS is **POSIX-compliant (Linux-only)**.
* Windows uses **Amazon FSx for Windows File Server** (SMB protocol).
  👉 Interview Tip: "EFS = Linux (NFS), FSx = Windows (SMB)."

---

## 3. **Default Throughput vs Provisioned Throughput (EFS)**

* **Default (Elastic Throughput / Bursting Mode)**

  * Throughput scales with the amount of data stored.
  * Example: 1 TB stored → baseline \~50 MB/s throughput, bursts up to 100 MB/s.
  * Good for normal workloads where throughput needs grow with storage.

* **Provisioned Throughput**

  * You set throughput manually, independent of storage size.
  * Example: Even if you store only 100 GB, you can provision 1 GB/s throughput.
  * Useful for small datasets with high performance needs (e.g., analytics, build pipelines).

---

## 4. **What does Lifecycle Policy do?**

* **EFS Lifecycle Management Policy** → Moves **infrequently accessed files** to **EFS Infrequent Access (IA) storage class** automatically after N days (7, 14, 30, 60, 90).
* Saves cost (EFS-IA is \~92% cheaper).
* When file is accessed again → auto-moves back to standard.

👉 Interview Phrase: “EFS lifecycle policy = automatic tiering to reduce storage cost.”

---

## 5. **What is required to mount EFS?**

To mount an EFS file system on EC2/EKS pods, you need:

1. **Security Group**

   * Allow inbound **NFS (TCP/2049)** on the EFS mount target SG.
   * Ensure EC2 instance SG allows outbound to EFS SG.

2. **NFS Client**

   * Install NFS client (`nfs-utils` or `amazon-efs-utils`) on EC2 / container.
   * Example mount command:

     ```bash
     sudo mount -t nfs4 -o nfsvers=4.1 fs-12345678.efs.us-east-1.amazonaws.com:/ /mnt/efs
     ```

3. **IAM Role (Optional)**

   * If using **EFS Access Points + IAM authorization**, you need IAM policies.
   * Example: Allow `elasticfilesystem:ClientMount`, `ClientWrite`.

---

✅ **Interview-Ready Short Answer**:

* **EBS vs EFS vs S3** → Block vs File vs Object storage, different workloads.
* **EFS for Windows?** → No, use FSx.
* **Throughput** → Default scales with size, Provisioned is manual override.
* **Lifecycle policy** → Move files to IA to cut cost.
* **Mounting needs** → SG (2049 open), NFS client, optional IAM for access points.

---

Perfect, bhai — let’s tackle these **EFS interview-style questions** with precise, scenario-ready answers.

---

## 1️⃣ **How do you mount EFS on multiple AZ EC2 instances?**

* **EFS is AZ-agnostic** → you can create **mount targets in each AZ** where EC2 instances exist.
* Steps:

  1. Create **EFS file system**.
  2. Create **mount targets in each AZ/subnet**.

     * Security Group allows inbound **TCP 2049 (NFS)**.
  3. On each EC2 instance, use **DNS name of EFS** to mount:

     ```bash
     sudo mount -t nfs4 -o nfsvers=4.1 fs-12345678.efs.us-east-1.amazonaws.com:/ /mnt/efs
     ```

  * EFS DNS resolves to the correct mount target in the AZ of the EC2 instance.
* ✅ Result: All EC2s across AZs access the same file system **concurrently**.

---

## 2️⃣ **What happens if one AZ goes down – will EFS still work?**

* **Yes, EFS is highly available**:

  * Data is stored **redundantly across multiple AZs**.
  * Even if **one AZ fails**, EC2 instances in other AZs can still access the file system.
* Only caveat: EC2 instances in the **downed AZ lose access** until the AZ recovers.
* Interview Tip: “EFS is multi-AZ by design → no single point of failure for storage.”

---

## 3️⃣ **EFS Lifecycle Management “IA class” – when does a file move?**

* Files move to **Infrequent Access (EFS-IA)** after **N days of no access**.
* N can be configured: **7, 14, 30, 60, 90 days**.
* Example: File last read 10 days ago, IA lifecycle = 7 days → moved to IA class.
* Accessing file in IA → automatically moved back to **Standard class**.

---

## 4️⃣ **EFS Access Points – how do they simplify multi-user access?**

* Access Points are **entry points with preconfigured POSIX identity** (UID/GID).
* Use case: Multi-user workloads where each user/app needs **isolated directory**.
* Benefits:

  * No need to manage UID/GID inside container manually.
  * Can enforce **root squashing** → users cannot escape their directories.
  * Simplifies **multi-tenant workloads in EKS/ECS**.

---

## 5️⃣ **How to integrate EFS with ECS Fargate / EKS pods?**

### **ECS Fargate:**

* Create **EFS file system + Access Point**.
* In ECS Task Definition → define **volume with EFSConfig**:

  ```json
  "volumes": [{
      "name": "efs-volume",
      "efsVolumeConfiguration": {
          "fileSystemId": "fs-12345678",
          "rootDirectory": "/app-data",
          "transitEncryption": "ENABLED",
          "authorizationConfig": {
              "accessPointId": "fsap-12345678",
              "iam": "ENABLED"
          }
      }
  }]
  ```
* Mount in container → container sees EFS storage like local volume.

### **EKS Pods:**

* Use **EFS CSI Driver**:

  1. Install `amazon-efs-csi-driver`.
  2. Create **StorageClass**, **PersistentVolume (PV)**, **PersistentVolumeClaim (PVC)** pointing to EFS.
  3. Pods use PVC → automatically mount EFS.

* ✅ Benefits: Shared storage across multiple pods, dynamic scaling, multi-AZ availability.

---
