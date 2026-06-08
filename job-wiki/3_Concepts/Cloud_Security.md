# Cloud Architecture & Security Patterns

This document details key cloud security and distributed systems concepts. It reflects systems you deployed and managed using GCP (at Concentrix) and AWS/Azure (at ConnectWise/Niyuj).

[← Back to Index](../README.md) | [← Previous: Gateway Stack: LiteLLM + Langfuse + LLM Guard](./Gateway_Stack_LiteLLM.md) | [Next: System Design Patterns Cheatsheet](./System_Design_Patterns.md)

---

## 🔒 1. Workload Identity (GCP) & IAM Security
In legacy setups, applications running on Kubernetes authenticate with cloud databases by mounting a Service Account JSON file. If a container is compromised, the JSON file can be leaked, exposing the entire database.
* **Workload Identity (GKE) / IAM Roles for Service Accounts (IRSA - EKS):**
  - Connects a Kubernetes Service Account directly to a Cloud IAM Service Account.
  - No secret keys are stored in configuration files or container volumes.
  - When the pod attempts to access GCP resources (e.g. Google Cloud Storage), the Google SDK queries GKE's metadata server. The server verifies the pod's identity and returns a short-lived, auto-rotating OAuth access token.
  - **Your Experience:** At Concentrix, we configured Workload Identity for all NestJS API and Python worker pods, ensuring zero static JSON keys were stored in GKE.

---

## 🛡️ 2. AWS WAF vs. Security Groups

Cloud security is built in layers (Defense in Depth).

### AWS WAF (Web Application Firewall):
* **Layer:** Layer 7 (Application Layer).
* **Role:** Evaluates HTTP/S request payloads.
* **Use Case:** Protects against application-level attacks (SQL Injection, Cross-Site Scripting, DDoS flood attacks, bad bot traffic).
* **Your Experience:** At ConnectWise, we configured WAF rule lists at the AWS Application Load Balancer to filter and block malicious query parameters.

### Security Groups:
* **Layer:** Layer 4 (Transport Layer).
* **Role:** Acts as a virtual firewall for your virtual machines (EC2 instances).
* **Use Case:** Defines ingress and egress rules for IP addresses and ports (e.g. allowing port `443` traffic from the load balancer, but blocking direct SSH port `22` traffic from the public internet).

---

## 🗄️ 3. Pre-signed S3 / GCS URLs
Allowing users to download files directly from a private cloud storage bucket requires a secure path:
* **The Problem:** Making the bucket public is a security risk. Routing file downloads *through* the Node.js API server wastes server bandwidth and memory.
* **The Solution (Pre-signed URLs):**
  1. The client requests a file download from the Node.js API.
  2. The Node.js service generates a secure, time-limited download URL (e.g. expires in 15 minutes) using the Cloud Storage SDK and its IAM credentials.
  3. The API returns this URL to the client browser.
  4. The client browser downloads the file directly from S3/GCS using the pre-signed URL, keeping the file private while offloading network bandwidth from the backend servers.

---

## 🌐 4. CAP Theorem in Distributed Systems

In any distributed data store, you can only guarantee two out of the following three properties during a network partition:

```text
               Consistency (C)
                     /\
                    /  \
                   /    \
                  /      \
                 /Network \
                /Partition \
   Availability (A) ────── Partition Tolerance (P)
```

1. **Consistency (C):** Every read receives the most recent write or an error.
2. **Availability (A):** Every non-failing node returns a response, without guarantee that it contains the most recent write.
3. **Partition Tolerance (P):** The system continues to operate despite arbitrary message loss or node crashes.

### Real-World Decision (AP vs CP):
* Because network partitions are inevitable in physical cloud networks, systems must choose between **CP** (Consistency + Partition Tolerance) or **AP** (Availability + Partition Tolerance).
* **CP Systems (e.g. MongoDB, Redis, ZooKeeper):** If a network split occurs, the system halts writes to the partitioned nodes to prevent split-brain state inconsistencies.
* **AP Systems (e.g. Cassandra, DynamoDB):** The system accepts writes on both partitions, resolving inconsistencies later (using Vector Clocks or Last-Write-Wins), prioritizing uptime over consistency.
