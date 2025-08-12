
# Kubernetes Deployments — From Beginner to Expert

---

## **1. Beginner Level — Deployment ka Base Samajhna**

### **Definition**
Deployment ek **Kubernetes object** hai jo declaratively application ko manage karta hai — isme replicas, rollout, rollback, aur scaling ka pura control hota hai.

---

### **Key Features**
- **Declarative updates** (YAML/JSON me desired state define hoti hai)
- **Self-healing** (ReplicaSet ensure karta hai desired number of pods hamesha running ho)
- **Rollouts & Rollbacks**
- **Scaling** (manual + Horizontal Pod Autoscaler ke sath)

---

### **Basic Workflow**
1. **Deployment create karte ho** → Deployment ek **ReplicaSet** banata hai → ReplicaSet pods launch karta hai.
2. **Rollout new version** → Naya ReplicaSet banega, purana scale down hoga.

---

### **Basic Commands**
```bash
# Deployment create karna
kubectl create deployment myapp --image=nginx

# Scale deployment
kubectl scale deployment myapp --replicas=5

# Update image
kubectl set image deployment/myapp nginx=nginx:1.25 --record

# Rollout status check
kubectl rollout status deployment/myapp

# Rollback to previous version
kubectl rollout undo deployment/myapp
````

---

## **2. Advanced Level — Deployment Strategies**

### **K8s me Deployment Strategies mainly do hoti hain:**

---

#### **a) Rolling Update (default)**

* Pods gradually replace hote hain.
* **No downtime**.
* Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

* **maxUnavailable** = ek time me kitne pods unavailable ho sakte hain.
* **maxSurge** = ek time me kitne extra pods ban sakte hain.

---

#### **b) Recreate**

* Purane pods delete → naye pods create.
* **Downtime possible**.
* Use when **backward compatibility** nahi hoti.

---

#### **c) Canary Deployment** (via multiple Deployments or Service Mesh)

* 10% traffic → new version.
* Monitor → gradually increase to 100%.


Okay — let’s break this into **concept + YAML code** so you see exactly how Canary works in Kubernetes using two Deployments with different `version` labels (`v1` for stable, `v2` for canary).

---

## **1️⃣ Concept**

* **Deployment v1** → Old stable version of your app. Label: `version: v1`
* **Deployment v2** → New canary version (only small % of traffic). Label: `version: v2`
* **Service** → Instead of pointing to just one version, it selects *both* Deployments (same `app` label, different `version`).
* **Traffic Split** → You control how much traffic goes to each version by **adjusting replicas** in each Deployment. Example:

  * `v1` → 9 replicas (90% traffic)
  * `v2` → 1 replica (10% traffic)

---

## **2️⃣ YAML Example**

### **Stable Deployment (v1)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 80
```

---

### **Canary Deployment (v2)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: myapp:2.0
        ports:
        - containerPort: 80
```

---

### **Service (Same for both)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

---

Istio me canary deployment traffic split **sidecar proxy (Envoy)** ke through hota hai, jo aapke pod ke sath inject hota hai.
Normal Kubernetes me aapko alag Service banana padta hai ya Deployment scale karke manage karna padta hai,
lekin Istio me aap **VirtualService** + **DestinationRule** ka use karke percentage-based traffic routing kar sakte ho.

---

### Example: Istio Canary Deployment (10% → 100% rollout)

#### 1️⃣ Two versions of your app

* **Deployment v1**: old stable version
* **Deployment v2**: new canary version
  Dono ke label alag honge (e.g., `version: v1` aur `version: v2`).

---

#### 2️⃣ DestinationRule

Ye rule Istio ko batata hai ki service ke kaunse versions hain aur unko kaise identify karna hai.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

---

#### 3️⃣ VirtualService

Ye rule request ko percentage me split karega.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - my-app
  http:
  - route:
    - destination:
        host: my-app
        subset: v1
      weight: 90
    - destination:
        host: my-app
        subset: v2
      weight: 10
```

---

#### 4️⃣ Rollout Process

1. **Start** → 90% traffic v1, 10% v2
   (new version production me aa gaya but sirf 10% users dekh rahe)
2. **Monitor** → Metrics check karo (latency, error rate, CPU/mem)
3. **If stable** → Weight change:

   * 50% v1, 50% v2
4. **If still stable** → 0% v1, 100% v2 (full rollout)

---

#### 5️⃣ Benefits in Istio

* **Zero downtime** — traffic shift hota hai live.
* **Quick rollback** — sirf VirtualService me weight change karo, turant purana version wapas aa jata hai.
* **Fine-grained control** — path-based, header-based ya user-based routing possible hai (sirf percentage hi nahi).

---

#### **d) Blue-Green Deployment** (via two separate environments)

* **Blue** = current live version.
* **Green** = new version.
* Traffic switch via Service/Ingress.

---

## **3. Expert Level — Real-world Deployment Mastery**

### **Pro Concepts**

* **Paused Rollout**: Testing config before applying change.

```bash
kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp
```

* **Probes Integration**: Liveness/Readiness probes define karna for safe rollout.
* **Resource Requests/Limits**: CPU/memory define karke scheduling optimize karna.
* **ProgressDeadlineSeconds**: Rollout stuck hone par fail detect karna.
* **Revision History Limit**: Purane rollouts ka retention limit set karna.

---

### **Example Deployment YAML**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 60
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## **4. Scenario-based Interview Questions (With Top 1% Answers)**

---

### **Q1: Pod CrashLoopBackOff ho raha hai after Deployment update — steps?**

**Answer:**

1. `kubectl describe pod` → events check karo.
2. `kubectl logs <pod>` → error identify karo.
3. Rollback if urgent:

   ```bash
   kubectl rollout undo deployment/myapp
   ```
4. Root cause fix → redeploy.

---

### **Q2: Tumhara rollout 20 min se stuck hai — reason & fix?**

**Answer:**

* Check readiness probes → fail to become ready = rollout stuck.
* Check imagePull errors.
* `progressDeadlineSeconds` set karke fail detect karo.
* Fix and:

  ```bash
  kubectl rollout resume deployment/myapp
  ```

---

### **Q3: Zero downtime deploy + load testing ke sath safe rollout kaise karoge?**

**Answer:**

* RollingUpdate with `maxUnavailable=0` or `1`.
* ReadinessProbe + StartupProbe.
* HPA integration for load handling.
* Canary rollout in % increments.

---

### **Q4: High rollback cost microservice me kaunsa deployment strategy?**

**Answer:**

* **Blue-Green** → Instant switch back to old version.

---

### **Q5: Compliance system me downtime allowed nahi — kya karoge?**

**Answer:**

* RollingUpdate with `maxUnavailable=0`.
* Pre-deployment validation scripts.
* Canary rollout in production mirror traffic.

---

## **5. Top 1% DevOps Mindset for Deployment**

* **Safety First** → Always have rollback strategy + readiness/liveness probes.
* **Automation** → Use GitOps (Argo CD) or CI/CD pipelines to manage deploys.
* **Metrics-Driven** → Monitor error rates, latency, resource usage during rollout.
* **Version Control** → Keep Deployment YAMLs in Git, version images with SHA, not `latest`.
* **Environment Parity** → Test rollout in staging with production-like traffic.
* **Minimal Blast Radius** → Use canary or phased rollouts in critical services.
* **Chaos Test** → Simulate node failure during rollout to test resilience.

```

---