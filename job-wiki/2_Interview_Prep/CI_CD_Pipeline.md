# CI/CD Pipeline Architecture & Rollout Strategies

This document details production-grade CI/CD patterns and deployment strategies. It reflects systems you configured and optimized using GitLab CI/CD at ConnectWise and GitHub Actions/Azure DevOps workflows.

[← Back to Index](../README.md) | [← Previous: MySQL at Scale](./MySQL_At_Scale.md) | [Next: React Hooks & Performance](./React_Hooks_Performance.md)

---

## 🛠️ Typical GitLab CI/CD Pipeline Flow

A pipeline is divided into stages that run sequentially. Within each stage, jobs run in parallel to maximize build speeds:

```text
[ Lint & Unit Test ] ──> [ Static Analysis ] ──> [ Build Docker Image ] ──> [ Helm Deploy (K8s) ] ──> [ Smoke Test ]
```

### 1. Verification Stage (Lint & Test)
* **Tasks:** Code linting (ESLint/Prettier), TypeScript compilation check, unit testing (Jest/Supertest).
* **Speed Hack:** Use caching for `node_modules` (or lockfiles) and run parallelized Jest workers to reduce this stage execution time to under **3 minutes**.

### 2. Security Scan Stage (Static Analysis)
* **Tasks:** Run dependency scanners (npm audit/Snyk) to check for CVEs and Git Guardian to block committed secrets.
* **Failure Rule:** Fail the pipeline if critical vulnerability packages are found.

### 3. Packaging Stage (Build)
* **Tasks:** Build a production Docker image and push it to a private container registry (e.g., GCP Artifact Registry).
* **Optimization:** Use multi-stage Docker builds to reduce image size (e.g., compile TS in a large builder image, then copy only JS files and node production dependencies into a lightweight Node Alpine runner image).

### 4. Continuous Deployment Stage (GitOps Helm)
* **Tasks:** Update GKE deployment configuration templates. We execute helm upgrade scripts, replacing the image tag version with the Git Commit SHA (`$CI_COMMIT_SHA`) for deterministic rollouts.

---

## 🚀 Rollout Strategies: Zero-Downtime Deployment

### 1. Rolling Updates (Kubernetes Default)
* **How:** Kubernetes replaces old Pod instances with new ones step-by-step.
* **Mechanism:** It respects `maxSurge` (how many extra pods can be created during rollouts) and `maxUnavailable` (how many pods can be offline).
* **Requirement:** Applications must support backward-compatible database schemas because old and new versions of the application code run concurrently.

### 2. Blue-Green Deployments
* **How:** Maintain two identical production environments: "Blue" (currently serving traffic) and "Green" (new code).

```text
               [User Requests]
                      │
                      ▼
               [Load Balancer]
                 │          │
                 ▼          ▼
             [Blue v1]  [Green v2]
             (Active)   (Staging)
```

1. Deploy new version to **Green**.
2. Run automated integration smoke tests on Green.
3. Switch Load Balancer routing to Green.
4. Keep Blue idle. If bugs are found post-release, switch routing back to Blue immediately.
5. **Pros:** Instantaneous rollback. No concurrent versions.
6. **Cons:** Expensive (doubles infrastructure resources during deployment).

### 3. Canary Deployments
* **How:** Slowly roll out a change to a small subset of users before pushing it to the entire cluster.
1. Route **5%** of user traffic to the new "Canary" version.
2. Monitor application performance: watch for HTTP 500 spikes, latency metrics, and log errors.
3. If metrics remain healthy, ramp up routing to 25%, 50%, and eventually 100%.

---

## 🔄 Automated Failures & Rollback Rules

At ConnectWise, we configured automated rollbacks to prevent bad code from staying in production:

1. **Deployment Health Check:**
   When Helm deploys a new container, the Kubernetes deployment controller waits for the `readinessProbe` to pass. If the new pods fail to report ready within 5 minutes (due to database connection failures, crash loops, or crashes), the deploy times out.

2. **Helm Rollback Command:**
   If the pipeline smoke tests fail, or Kubernetes deployment fails, the CI pipeline catches the error and executes:
   ```bash
   helm rollback <release-name> <previous-revision-id>
   ```
   This immediately reverts the replica sets back to the previous stable state in under 5 seconds.
