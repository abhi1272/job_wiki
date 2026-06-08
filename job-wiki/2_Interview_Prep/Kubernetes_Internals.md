# Kubernetes Internals & GKE Operations

This document details core Kubernetes concepts and troubleshooting procedures. It is anchored in your hands-on experience managing GKE clusters for the No-Code AI Agent Builder at Concentrix and ITBoost deployments on AKS/AWS at ConnectWise.

[← Back to Index](../README.md) | [← Previous: Agent Builder Deep Dive](./Agent_Builder_Deep_Dive.md) | [Next: Microservices Patterns](./Microservices_Patterns.md)

---

## ⚙️ Core Kubernetes Components Explained

```text
               Ingress (External Traffic)
                         │
                      [Service]
                         │
       ┌─────────────────┴─────────────────┐
       ▼                                   ▼
  [Pod - replica 1]                   [Pod - replica 2]
   ├─ Container (App)                  ├─ Container (App)
   └─ readiness/liveness checks        └─ readiness/liveness checks
```

### 1. Pods vs. Containers
* **Explanation:** A Pod is the smallest deployable unit in Kubernetes. It represents a single instance of a running process. A Pod can contain one or more containers (e.g., your Node.js application container and a sidecar container for logging or service mesh proxies).
* **VPC Networking:** All containers within a Pod share the same Network namespace, IP address, and storage volumes. They communicate with each other using `localhost`.

### 2. Services
Kubernetes Pods are ephemeral (they can be terminated and rescheduled at any time). A **Service** is an abstraction that defines a logical set of Pods and a policy to access them.
* **ClusterIP (Default):** Exposes the Service on a cluster-internal IP. Good for internal communication (e.g., Python workers talking to the LiteLLM routing pod).
* **NodePort:** Exposes the Service on each Node's IP at a static port.
* **LoadBalancer:** Creates an external load balancer in the cloud provider (e.g., GCP Cloud Load Balancer) and assigns a public IP.

### 3. Readiness vs. Liveness Probes
* **Liveness Probe:** Checks if the container is running. If it fails, Kubernetes kills the container and restarts it based on the restart policy.
  - *Example configuration:* A simple endpoint `/healthz` returning `200 OK`.
* **Readiness Probe:** Checks if the container is ready to accept network traffic. If it fails, the container is removed from the service's endpoint pool so traffic is not routed to it.
  - *GKE context:* At Concentrix, our NestJS backend took 15 seconds to establish connection pools to MongoDB and Redis. We set a `readinessProbe` with an `initialDelaySeconds: 15` to prevent users from receiving `502 Bad Gateway` errors during rolling deployments.

---

## 📈 Autoscaling & HPA (Horizontal Pod Autoscaler)

The Horizontal Pod Autoscaler automatically scales the number of Pods in a replication controller or deployment based on metrics:
* **Metric Source:** Metrics Server collects CPU and memory usage.
* **Custom Metrics:** At Concentrix, we integrated **KEDA** (Kubernetes Event-driven Autoscaling) to scale Python RAG ingestion workers. Scaling purely on CPU was too slow for batch file uploads. KEDA polled Kafka consumer lag directly, scaling worker pods from 1 to 10 instances when queue depth spiked.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rag-worker-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: python-rag-worker
  minReplicaCount: 1
  maxReplicaCount: 15
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-service.default.svc.cluster.local:9092
      topic: document-ingestion
      consumerGroup: python-ingest-group
      lagThreshold: '10'
```

---

## 🛠️ Diagnostics & Troubleshooting (CrashLoopBackOff)

When a pod enters **`CrashLoopBackOff`**, it means the container is repeatedly starting and crashing.

### Troubleshooting Workflow (Your playbook):

1. **Check Pod Status & Events:**
   ```bash
   kubectl get pods -n production
   kubectl describe pod <pod-name> -n production
   ```
   *Look for: Warnings, Failed Mounts, Resource Limit violations (OOMKilled), or failing probes.*

2. **Retrieve Exit Logs:**
   ```bash
   kubectl logs <pod-name> --previous -n production
   ```
   *The `--previous` flag fetches logs from the crashed container before the restart, exposing database connection failures, unhandled promise rejections, or missing environment variables.*

3. **Check Resource Over-allocation (OOMKilled):**
   Run `kubectl describe pod` and check the `Last State` section. If the Reason is `OOMKilled` (Out Of Memory), the container exceeded its configured memory limits (e.g. loading a large LLM library without sufficient RAM limits).

4. **Verify Liveliness Probe Misconfigurations:**
   If the container starts fine but is killed after 30 seconds, check if the liveness probe endpoint is returning timeout errors or if the database healthcheck query is blocking.
