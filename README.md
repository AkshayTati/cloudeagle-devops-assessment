# CloudEagle DevOps Engineering Assessment

![CloudEagle Target Architecture](assets/architecture-diagram.png)

This repository contains the engineering deliverables for the DevOps Engineer technical assessment. It proposes a production-grade software delivery pipeline and a target cloud infrastructure architecture for the Spring Boot `sync-service`.

The designs within this repository prioritize **immutable infrastructure**, **zero-trust security boundaries**, and **FinOps-aware scaling**, ensuring the architecture meets the demands of a high-growth SaaS environment while adhering to strict startup cost constraints.

---

## 📁 Repository Structure & Navigation

The assessment is divided into two core deliverables:

### Part 1: Deployment & CI/CD Design
* 📜 **[`Jenkinsfile`](./Jenkinsfile)**: The declarative multi-branch pipeline engine featuring PR verification, immutable artifact storage, decoupled deployment gates, and an out-of-band emergency rollback mechanism.
* 🏗️ **[`docs/CI-CD-DESIGN.md`](./docs/CI-CD-DESIGN.md)**: The comprehensive delivery governance document detailing the environment branching strategy, zero-trust secrets injection via GCP Secret Manager, and the Blue/Green deployment topology.

### Part 2: Infrastructure Design
* ☁️ **[`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md)**: The target infrastructure framework. This document utilizes a conditional decision matrix to evaluate compute, database, and networking trade-offs before establishing a definitive **Baseline Architecture** (GKE Autopilot + MongoDB Atlas) optimized for CloudEagle's current growth phase.
* 🗺️ **[`assets/architecture-diagram.png`](./assets/architecture-diagram.png)**: *(Optional: If you created a diagram, reference it here. If not, delete this bullet point)* The visual topology of the proposed GCP network, compute, and data layers.

---

## 🚀 Core Engineering Principles

Rather than prescribing static, single-tier solutions, this architecture approaches both automation and infrastructure through a matrix of operational trade-offs:

1. **Compile Once, Deploy Everywhere:** The deployment lifecycle mandates that a single, unalterable release binary (`.jar`) is built during merge verification. It is tagged uniquely with the build number and Git SHA, and archived to Google Cloud Storage. Lower and upper environments pull this identical artifact to entirely eliminate environmental drift.
2. **Zero-Trust Perimeters:** Plaintext credentials and static key files are strictly banned. Compute tiers leverage native Google IAM identity mappings (Service Accounts / Workload Identity) to pull secrets directly into volatile application memory at boot time.
3. **Metric-Driven Infrastructure:** Compute, database clustering, and observability strategies are treated as conditional branches. For example, serverless runtimes (Cloud Run) are evaluated against strict application prerequisites (idempotency, processing timeouts) to ensure infrastructure choices never compromise data layer integrity.