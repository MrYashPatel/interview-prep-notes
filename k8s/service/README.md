
---

🚀 Mastering Kubernetes Services (Beginner → Expert → Top 1%)


---

1. Basics of Kubernetes Services

👉 A Service in Kubernetes is an abstraction layer that provides stable networking (a consistent IP and DNS name) for a group of Pods.

Types of Services

1. ClusterIP – Default, exposes internally within the cluster.


2. NodePort – Exposes externally on each node’s IP + a static port.


3. LoadBalancer – Integrates with cloud provider LB (AWS ELB, GCP LB, etc.).


4. ExternalName – Maps service to an external DNS name.


5. Headless Service – (clusterIP: None) – exposes Pod IPs directly, often used with StatefulSets & DNS-based service discovery.




---

2. Beginner Level – Scenario Questions

🔹 Q1: You deployed 3 replicas of nginx. You created a ClusterIP Service.
👉 How does the Service distribute traffic?

Answer: It uses kube-proxy (iptables or IPVS mode) to perform round-robin load balancing across available Pod endpoints.


---

🔹 Q2: If a Pod behind a Service is deleted, what happens?

Answer: The Service updates its Endpoints object automatically (via kube-controller-manager). Traffic is re-routed only to healthy Pods.


---

🔹 Q3: Why use a Service instead of Pod IPs directly?

Answer: Pod IPs are ephemeral (change on restart). Services provide a stable virtual IP and DNS name (myservice.default.svc.cluster.local).


---

3. Intermediate Level – Scenario Questions

🔹 Q4: You exposed a microservice with NodePort, but your client cannot connect.
👉 How do you debug?

Answer:

1. Verify kubectl get svc shows a valid NodePort (30000–32767).


2. Check kubectl get nodes -o wide and try <NodeIP>:<NodePort>.


3. Ensure NetworkPolicy / firewall allows that port.


4. If in cloud, check security group rules (AWS/GCP/Azure).




---

🔹 Q5: When do you use Headless Services?

Answer:

For StatefulSets like Cassandra, Kafka, Elasticsearch, where each Pod needs a unique DNS name.

DNS resolves directly to Pod IPs (pod-0.svc.cluster.local).

Useful for service discovery with applications that want direct Pod-to-Pod communication.



---

🔹 Q6: What’s the difference between ClusterIP and ExternalName?

Answer:

ClusterIP: For in-cluster service communication (creates virtual IP).

ExternalName: Creates CNAME record mapping to external DNS (e.g., db.external.com) – no kube-proxy involvement.



---

4. Advanced Level – Scenario Questions

🔹 Q7: You created a Service but it has no endpoints. Why?

Answer:

Pods don’t have matching labels for the Service selector.

Pods are in CrashLoopBackOff.

Readiness probe is failing → Pod not added to Service.


👉 Debug with:

kubectl describe svc myservice
kubectl get endpoints myservice
kubectl get pods -l app=myapp


---

🔹 Q8: How does kube-proxy implement Service networking?

Answer:

In iptables mode: Creates DNAT rules to forward Service IP → Pod IP.

In IPVS mode: Uses kernel’s IPVS load balancer for higher performance.

In userspace mode (deprecated): Proxies traffic via userspace daemon.



---

🔹 Q9: In AWS, you created a LoadBalancer Service, but your external IP is <pending>. Why?

Answer:

Cluster is not on a cloud provider with LB integration.

Missing cloud-provider configuration in kubelet.

IAM role permissions not allowing ELB creation.



---

5. Expert Level – Scenario Questions (Top 1%)

🔹 Q10: You deployed a Service for your API, but traffic sometimes goes to Pods on other nodes, causing latency. How do you fix this?

Answer: Use Session Affinity (ClientIP) or Topology Aware Routing:

spec:
  sessionAffinity: ClientIP
  topologyKeys:
  - "kubernetes.io/hostname"

👉 Ensures requests prefer local node Pods for reduced latency.


---

🔹 Q11: In production, your Service is facing uneven traffic distribution. What could be the reason?

Answer:

Readiness probes failing on some Pods → fewer endpoints.

Pod anti-affinity causing Pods to concentrate on fewer nodes.

Network policy / CNI plugin blocking some Pods.

IPVS mode misconfiguration.



---

🔹 Q12: You want zero downtime deployment for a Service. How do you achieve it?

Answer:

Use readinessProbe so traffic only routes to healthy Pods.

Use rollingUpdate strategy in Deployment.

Ensure graceful termination by setting terminationGracePeriodSeconds and preStop hook to drain connections.



---

🔹 Q13: How to expose a multi-tenant Service with different domains?

Answer:

Use Ingress + ClusterIP Service for path/host-based routing.

Example:

tenant1.example.com → Service A

tenant2.example.com → Service B




---

🔹 Q14 (Killer Scenario):
Your Service is exposed via LoadBalancer in AWS. Traffic is coming in, but some Pods don’t receive traffic at all. What’s happening?

Answer:

AWS ELB health checks only pass on ready Pods.

Readiness probe misconfigured → LB never forwards traffic.

Or Security Group rules block node-to-node communication.



---

🔹 Q15: Can multiple Services point to the same Pod?

Answer: Yes. As long as Pod labels match multiple Service selectors, the Pod will have multiple Endpoints and respond to multiple Service IPs/DNS names.


---
