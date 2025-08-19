
---

# 🔹 Ephemeral Volumes in Kubernetes

👉 **Ephemeral = Short-lived**.

* Pod ke saath create hote hai aur Pod delete hote hi khatam ho jaate hai.
* Storage data **persist nahi hota**.
* Mostly use hote hai: scratch space, caching, config/data injection ke liye.

---

# 🔹 Types of Ephemeral Volumes

### 1. **emptyDir**

* **Sabse common ephemeral volume**.
* Jab Pod ek Node pe schedule hota hai tab create hota hai.
* Jab Pod delete hota hai → data bhi delete.
* Use case:

  * Temporary cache
  * Logs aggregation
  * Scratch space

```yaml
volumes:
  - name: cache
    emptyDir: {}
```

⚠️ Agar Pod reschedule hua dusre node pe → data lost.

---

### 2. **configMap / secret volumes**

* Config aur secret ko **files ke form me pod me mount karte hai**.
* Ephemeral hote hai kyunki Pod ke sath destroy ho jaate hai.

```yaml
volumes:
  - name: config
    configMap:
      name: app-config
```

---

### 3. **downwardAPI volume**

* Pod ke apne metadata (labels, annotations, resource limits) ko expose karta hai.
* Ephemeral hote hai → sirf Pod lifecycle ke liye.

```yaml
volumes:
  - name: podinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
```

---

### 4. **projected volumes**

* Multiple volume sources (ConfigMap, Secret, DownwardAPI, ServiceAccountToken) ko ek hi volume me project karne ke liye.

```yaml
volumes:
  - name: app-secrets
    projected:
      sources:
        - secret:
            name: db-secret
        - configMap:
            name: app-config
```

---

### 5. **ephemeral CSI volumes** (advanced 🚀)

* CSI drivers ke through temporary volumes.
* Example:

  * Secrets Store CSI Driver (injects secrets at runtime).
  * Special scratch volumes.
* Defined using `volumeClaimTemplate` directly inside Pod/Deployment.

```yaml
volumes:
  - name: ephemeral-vol
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
```

---

# 🔹 Summary

| Type              | Example Use Case                         | Persistent? |
| ----------------- | ---------------------------------------- | ----------- |
| **emptyDir**      | Cache, temp data, logs                   | ❌ No        |
| **configMap**     | App configs                              | ❌ No        |
| **secret**        | API keys, passwords                      | ❌ No        |
| **downwardAPI**   | Pod metadata to container                | ❌ No        |
| **projected**     | Multiple sources in one volume           | ❌ No        |
| **CSI ephemeral** | Dynamic runtime volumes (secrets, vault) | ❌ No        |

---

⚡ **Difference recap:**

* **Ephemeral Volumes** → Pod ke saath hi destroy.
* **Persistent Volumes** → Pod delete ho jaye, storage safe rahe.

