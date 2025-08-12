---

1. Beginner Level — Deployment ka Base Samajhna

Definition:
Deployment ek Kubernetes object hai jo declaratively application ko manage karta hai — isme replicas, rollout, rollback, aur scaling ka pura control hota hai.

Key Features:

Declarative updates (YAML/JSON me desired state define hoti hai)

Self-healing (ReplicaSet ensure karta hai desired number of pods hamesha running ho)

Rollouts & Rollbacks

Scaling (manual + HPA ke sath)


Basic Workflow:

1. Deployment banate ho → Deployment ek ReplicaSet banata hai → ReplicaSet pods launch karta hai.


2. Rollout new version → Naya ReplicaSet banega, purana scale down hoga.



Basic Commands:

kubectl create deployment myapp --image=nginx
kubectl scale deployment myapp --replicas=5
kubectl set image deployment/myapp nginx=nginx:1.25 --record
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp


---

2. Advanced Level — Deployment Strategies

K8s me Deployment Strategies mainly do hoti hain:

a) Rolling Update (default)

Pods gradually replace hote hain.

No downtime.

Parameters:

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%

maxUnavailable = ek time me kitne pods unavailable ho sakte hain.
maxSurge = ek time me kitne extra pods ban sakte hain.


b) Recreate

Purane pods delete → naye pods create.

Downtime possible.

Use when backward compatibility nahi hoti.


c) Canary Deployment (via multiple Deployments or Service Mesh)

10% traffic → new version.

Monitor → gradually increase to 100%.


d) Blue-Green Deployment (via two separate environments)

Blue = current live version.

Green = new version.

Traffic switch via Service/Ingress.



---

3. Expert Level — Real-world Deployment Mastery

Pro Concepts:

Paused Rollout: Testing config before applying change.

kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp

Probes Integration: Deployment me liveness/readiness probe define karna for safe rollout.

Resource Requests/Limits: CPU/memory define karke scheduling optimize karna.

ProgressDeadlineSeconds: Rollout stuck hone par fail detect karna.

Revision History Limit: Purane rollouts ka retention limit set karna.


Example:

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


---

4. Scenario-based Interview Questions (With Top 1% Answers)

Q1: Pod CrashLoopBackOff ho raha hai after Deployment update — steps?
A:

1. kubectl describe pod → events check.


2. kubectl logs <pod> → error identify.


3. Rollback if urgent:

kubectl rollout undo deployment/myapp


4. Root cause fix → redeploy.




---

Q2: Tumhara rollout 20 min se stuck hai — reason & fix?
A:

Check readiness probes → fail to become ready = rollout stuck.

Check imagePull errors.

progressDeadlineSeconds set karke fail detect karo.

Fix and kubectl rollout resume.



---

Q3: Tumhe zero downtime deploy karna hai aur load testing ke sath safe rollout ensure karna hai — approach?
A:

RollingUpdate with low maxUnavailable (0 or 1).

ReadinessProbe + startupProbe.

HPA integration so that load pe scale ho sake.

Canary deploy in % increments.



---

Q4: Tumhare paas ek microservice hai jisme rollback cost high hai — kaunsa deployment strategy?
A:

Blue-Green so that switch back is instant (Service point to blue again).



---

Q5: Tumko ek compliance system me deploy karna hai jaha downtime allowed nahi — kya karoge?
A:

Rolling update with maxUnavailable=0.

Pre-deployment validation scripts.

Canary rollout in production mirror traffic.



---

5. Top 1% DevOps Mindset for Deployment

Safety First: Always have rollback strategy + readiness/liveness probes.

Automation: Use GitOps (Argo CD) or CI/CD pipelines to manage deploys.

Metrics-Driven: Monitor error rates, latency, resource usage during rollout.

Version Control: Keep Deployment YAMLs in Git, version images with SHA, not latest.

Environment Parity: Test rollout in staging with production-like traffic.

Minimal Blast Radius: Use canary or phased rollouts in critical services.

Chaos Test: Simulate node failure during rollout to test resilience.



---