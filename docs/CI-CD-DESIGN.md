# CI/CD Architecture & Deployment Design: sync-service

This document outlines the software delivery lifecycle, automation framework, and deployment topology designed for the Spring Boot `sync-service`. The architecture prioritizes immutable artifacts, zero-trust security boundaries, and high-availability release mechanisms.

---

## 1. Branching Strategy & Environment Mapping

The repository utilizes an environment-scoped branching strategy to isolate blast radiuses and ensure stable deployment velocity.

| Git Branch | Target Environment | Deployment Trigger |
| :--- | :--- | :--- |
| `feature/*` | Local / Ephemeral Dev | Manual / Local validation only. |
| `develop` | QA | Automated on merge/push. |
| `release/*` | Staging | Automated on branch creation (UAT release candidate). |
| `main` | Production | Decoupled from merge; requires manual approval gate. |

### Preventing Accidental Production Deployments
To eliminate the risk of human error or automated misfires reaching production, the following guardrails are enforced:

1. **Branch Protection Rules:** Direct pushes to `main` and `release/*` are programmatically blocked via repository settings. All code changes must originate from a Pull Request.
2. **Mandatory Peer Review:** Merges to `main` require a minimum of two approved code reviews and successful execution of the PR validation pipeline.
3. **Decoupled Promotion Gate:** Merging code into `main` does not trigger an automatic deployment to live production instances. It merely signals that the codebase is ready. Deployment is executed via an explicit `input` checkpoint in the Jenkins pipeline, requiring a manual click from an authorized operator.

---

## 2. Jenkins Pipeline Lifecycle & Rollback Mechanics

The pipeline is built on the core DevOps paradigm of **"Compile Once, Deploy Everywhere."** We avoid compiling code separately for different environments to prevent configuration or environmental drift.

### PR Lifecycle vs. Merge Lifecycle
* **On Pull Request (Ephemeral Validation):** The pipeline runs lightweight, non-mutating validation steps. This includes compiling the source code (`./mvnw clean compile`), executing unit/integration tests, and running static code analysis (SonarQube) for security vulnerabilities. No artifacts are preserved, and no infrastructure state changes.
* **On Merge (Immutable Production Build):** The pipeline compiles the source, executes tests, and packages a definitive, unalterable executable `.jar`. This binary is assigned a unique tracking tag structured as `v1.0.[Jenkins_Build_Number]-[Short_Git_SHA]`. The artifact is instantly archived in a version-controlled Google Cloud Storage (GCS) bucket, serving as our single source of truth for deployments.

### Automated Rollback Strategy
If a deployment fails downstream health checks or encounters critical production runtime errors, the pipeline includes an out-of-band, parameterized rollback routine:
1. The operator triggers the pipeline manually, selecting the target environment and entering the last known stable version tag (e.g., `v1.0.14-a1b2c3d4`).
2. The pipeline completely bypasses git checkout, testing, and compilation stages to minimize Mean Time to Recovery (MTTR).
3. Jenkins fetches the specified immutable `.jar` directly from the GCS bucket and immediately deploys it to the target instances, reverting the environment state in seconds.

---

## 3. Configuration Management & Secrets Security

Hardcoding credentials or infrastructure states inside source control is strictly prohibited. The application decouples configuration from code adhering to Twelve-Factor App principles.

### Environment-Specific Configurations
We leverage **Spring Boot Profiles** (`application-qa.yml`, `application-prod.yml`) for structural, non-sensitive environmental defaults (such as logging levels or internal service endpoints). The active profile is injected at runtime by the startup script via system properties:

java -jar sync-service.jar --spring.profiles.active=prod

### Zero-Trust Secrets Handling
Sensitive strings—specifically the MongoDB connection URI and third-party API keys—are never stored on disk, inside configuration files, or within environment variables visible to the host system.
1. All database passwords and private keys are provisioned inside **GCP Secret Manager**.
2. The GCP Compute Engine VMs run under a dedicated, custom IAM Service Account with minimal privileges, restricted strictly to the required secret resource path (`roles/secretmanager.secretAccessor`).
3. At application boot, the Spring Boot runtime authenticates via the Google Instance Metadata Service (IMDSv2) to fetch the secrets directly into memory. This ensures credentials never touch static storage or Git repositories.

---

## 4. Deployment Strategy: Blue/Green Topology

For a critical data synchronizer like `sync-service`, minimizing operational disruption and maintaining database consistency is critical. We implement a **Blue/Green Deployment Strategy** for our production environment.

### Evaluation of Alternatives
* **Why Recreate was rejected:** Recreate drops all active connections and introduces a visible window of downtime while the new Java Virtual Machine (JVM) undergoes a cold start. This violates high-availability requirements.
* **Why Rolling Update was rejected:** Rolling updates update VM instances sequentially, meaning older and newer versions of the application run simultaneously. For a service performing background mutations against MongoDB, multi-version concurrency can easily result in data race conditions, duplicate synchronization records, or collection state corruption.
* **Why Blue/Green was selected:** It provides total environment isolation. We maintain the current stable environment (**Blue**) while spinning up or updating an identical standby environment (**Green**). It guarantees that only one version of the application code interacts with the production database at any given point during a release.

### Zero-Downtime Traffic Shifting Mechanics
1. **Target Provisioning:** The Jenkins pipeline applies the new immutable artifact version template to the standby (Green) Managed Instance Group (MIG).
2. **Health Verification:** The application undergoes automated internal health checking and validation probes while isolated from live user traffic.
3. **Atomic Cutover:** Once health parameters pass, an automated directive updates the routing map of the **Google Cloud HTTPS Load Balancer (GLB)**. Traffic shifts instantly and atomically from the Blue target group to the Green target group. 
4. The Blue group remains in a standby state for a designated holding window to allow for immediate, zero-downtime failback if anomalies are detected post-release.