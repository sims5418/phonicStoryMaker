# GCP Infrastructure Investigation

> Mapping AWS-familiar building blocks to GCP equivalents, with cost and decision notes for the phonicStoryMaker stack.

---

## Table of Contents

1. [Compute / Instance Picking](#1-compute--instance-picking)
2. [Metrics & Logging](#2-metrics--logging)
3. [Monitoring & Alarms](#3-monitoring--alarms)
4. [Queue](#4-queue)
5. [Database](#5-database)
6. [Authentication](#6-authentication)
7. [Storage](#7-storage)
8. [Language & IaC](#8-language--iac)

---

## 1 Compute / Instance Picking

### 1.1 AWS ↔ GCP Equivalents

| AWS              | GCP                            |
| ---------------- | ------------------------------ |
| EC2              | Compute Engine (GCE)           |
| ECS              | GKE                            |
| Fargate          | Cloud Run *or* GKE Autopilot   |

### 1.2 Pricing & Trade-offs (via Gemini)

#### Budget Summary — 2 vCPU / 8 GB RAM

For a standard **2 vCPU / 8 GB RAM** workload running in a major US region for a full month (730 hours), pricing shifts based on management overhead and resource utilization:

- **Compute Engine (GCE) & GKE Standard** — the most cost-effective baseline for steady, 24/7 workloads, because you pay for the underlying VM directly. GKE Standard usually requires multiple nodes for HA, which multiplies the cost.
- **GKE Autopilot** — ~50% premium, because Google fully manages the nodes, security, and scaling. You pay exactly for what your Pod requests.
- **Cloud Run** — most flexible, potentially cheapest. With "always-allocated" CPU it beats standard VM pricing for continuous traffic; with scale-to-zero it can cut the bill to a fraction.

#### Cost & Fitment Comparison

| GCP Service             | Est. Monthly Cost (2 vCPU / 8 GB)         | Billing Metric                                          | Best For                                                                       | Anti-Patterns                                                                                       |
| ----------------------- | ----------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| **Compute Engine (GCE)**| **~$66**                                  | Per VM instance size + disk                             | Traditional monolithic apps, fixed 24/7 heavy workloads, custom OS kernels.    | Highly unpredictable traffic spikes, or microservices needing frequent automated scaling.           |
| **GKE Standard**        | **~$66** *(× node count)*                 | Per node VM + disk                                      | Standard Kubernetes microservices requiring deep cluster customization.        | Small, single-container apps — a 1-node cluster wastes money on K8s overhead.                       |
| **GKE Autopilot**       | **~$104**                                 | Per Pod resource request per second                     | Teams wanting the Kubernetes API without managing OS/nodes.                    | 24/7 compute at massive scale where you could save 50%+ by optimizing your own VMs.                 |
| **Cloud Run (always-on)** | **~$54**                                | Per instance duration                                   | Web apps / APIs that need rapid scaling but have a constant baseline.          | Long-running background workers (e.g., streaming data processing) that never drop in load.          |
| **Cloud Run (on-demand)** | **~$17** *(or lower)*                   | Only during active request processing (scales to zero) | Internal tools, dev environments, APIs with unpredictable/intermittent traffic. | Stateful legacy apps that can't tolerate instance recycles, or continuous background processing.    |

> **Note:** All estimates include a baseline **50 GB balanced disk** and **100 GB internet egress** (~$13.50 combined).

### 1.3 Initial Decision

#### Main server — always on

- **Choice:** **GKE Standard** *(kept primarily as a learning exercise — see §10.2 for the honest trade-off)*
- **Reasons:**
  - Autoscaling out of the box
  - Less maintenance than GCE — auto node updates with rolling update setting, no manual `apt-get update`
  - **Hands-on Kubernetes experience** — the primary motivator for choosing this over Cloud Run
  - First cluster's management fee is free
- **Cost-saving tactic:** start on a small node (e.g. **e2-medium**, ~$24/mo + disk) instead of the e2-standard-2 baseline in §1.2. Allocatable RAM is ~2.5–3 GB after GKE system overhead — enough for a small Go API in Phase 0. Resize up when load demands.
- **Accepted trade-off:** running 1 node means no HA (node failure / auto-upgrade = downtime). Acceptable for Phase 0 PoC; revisit before Phase 1 launch.

#### Story-generation instance — on-demand, mostly idle

- **Choice:** **Cloud Run Service + Pub/Sub push subscription**
- **Pattern:** Pub/Sub message → push to Cloud Run HTTP endpoint → worker processes → publishes completion
- **Config:**
  - `concurrency = 1` (one generation per instance — never accept 80 parallel generations on a single container)
  - `max-instances` bounded (avoid runaway scaling under retry storms)
  - Request timeout sized for full generation (text + ~10 images)
- **Reason:** Lowest maintenance effort; scale-to-zero matches the bursty workload; Pub/Sub gives at-least-once delivery with retries

#### Region

- **Phase 0:** **`us-east1`** (South Carolina) — single region, lowest-latency US East coverage, cheapest US tier
- All resources (GKE cluster, Cloud Run, GCS bucket, Pub/Sub, Cloud SQL when added) pinned to this region for Phase 0
- Multi-region / EU region (`europe-west2`) to be considered when GDPR-K traffic justifies it

---

## 2 Metrics & Logging

**AWS:** CloudWatch → **GCP:** Cloud Logging + Cloud Monitoring

| Tier               | Free Allowance         |
| ------------------ | ---------------------- |
| Cloud Logging      | 50 GB / month          |
| Log retention      | 30 days                |
| Native metrics     | Free                   |
| Metric ingestion   | Up to 150 M / month    |

→ For our expected volume, **basically free**.

---

## 3 Monitoring & Alarms

**Cloud Monitoring alerts** (GCP-native alerting on the metrics above).

---

## 4 Queue

**AWS:** SQS → **GCP:** Pub/Sub

---

## 5 Database

| Purpose         | Choice              |
| --------------- | ------------------- |
| AWS DynamoDB →  | Cloud Firestore     |
| Local / file    | SQLite file         |

> **Follow-up:** The **credit / subscription / payment transaction system** needs its own design document. Requirements (idempotent operations, transactional ledger, at-least-once Pub/Sub redelivery, concurrent load correctness — see PRD §Scalability) are not solved by picking a database alone. Schema, idempotency-key strategy, and ledger semantics will be designed in a separate doc before implementation begins.

---

## 6 Authentication

**AWS:** Cognito → **GCP:** Firebase Authentication

- Free tier: **< 50k monthly active users**

---

## 7 Storage

**AWS:** S3 → **GCP:** Google Cloud Storage (GCS)

---

## 8 Language & IaC

| Layer            | Choice                                  |
| ---------------- | --------------------------------------- |
| IaC              | Terraform *(replaces CloudFormation)*   |
| General server   | Go                                      |
| LLM server       | Python                                  |

---

## 9 Environments

Two-environment promotion pipeline:

| Environment | Purpose | Naming |
| --- | --- | --- |
| **gamma** | Pre-production / staging. Full end-to-end testing on real GCP infra (real Cloud Run, GKE, Pub/Sub, etc.) with **test-mode** payment providers, **isolated database**, and synthetic users. Last gate before prod. | `*-gamma` resource suffixes; separate GCP project recommended |
| **prod** | Live customer-facing environment. Real payments, real users. | `*-prod` suffixes; separate GCP project |

**Promotion flow:** code change → CI → deploy to **gamma** → run smoke/integration tests → manual promote to **prod**.

**Open questions to settle before build:**
- Single GCP project with resource prefixes, or separate projects per env? *(Separate projects is the GCP-idiomatic answer — cleaner IAM boundary, simpler billing breakdown, no accidental cross-env access.)*
- Do we need a developer-local "dev" environment beyond emulators? *(Deferred per §10.3 — emulator story is out of scope for now.)*

---

## 10 Assessment vs HLD / PRD

A self-review against `doc/1_RequirementDoc.md` (v3) and `doc/2_HighLevelDesign_P0.md`.

### 10.1 What holds up

- **Cloud Run for story generation** — bursty queue-triggered workload (15s text + 10s × N images) fits Cloud Run's scale-to-zero model. Pub/Sub push subscription or Eventarc triggers it cleanly.
- **GCS for blobs** — directly satisfies PRD §Scalability ("object storage for generated images").
- **Pub/Sub** — clean SQS analog; HLD already assumes a queue tier.
- **Firebase Auth** — 50k MAU free tier clears the 10k MAU target; parent-only account model maps fine.
- **Terraform** — CloudFormation has no GCP equivalent; Terraform also keeps multi-cloud options open.
- **Cloud Logging + Monitoring free tier** — accurate for expected volume.

### 10.2 Decisions worth re-examining

#### Main server: GKE Standard vs Cloud Run (min-instances=1)

GKE Standard with **1 node** is allowed and costs ~$66/month at the 2 vCPU / 8 GB baseline, or as low as **~$24/mo on an e2-medium**. That puts it in the same ballpark as Cloud Run always-on (~$54). **Cost is roughly a wash**; the real trade-off is HA and ops surface.

| Factor                  | GKE Standard (1 node)                               | Cloud Run (min=1)                            |
| ----------------------- | --------------------------------------------------- | -------------------------------------------- |
| Monthly cost            | ~$24 (e2-medium) – ~$66 (e2-standard-2)             | ~$54 (2 vCPU baseline; less if smaller)      |
| HA out of the box       | ❌ Node failure → pod down until replacement        | ✅ Managed; instance failure is transparent  |
| Ops surface             | kubectl, manifests, ingress, node-pool upgrades     | Just deploy a container                      |
| Horizontal scale path   | Add nodes / HPA                                     | Bump max-instances; automatic                |
| K8s learning value      | High                                                | None                                         |

**Decision: keep GKE Standard, primarily for the K8s learning value.** Cloud Run would be the lower-risk, lower-ops pick at similar cost, but the team explicitly wants hands-on Kubernetes experience and Phase 0's small scale makes this a low-stakes environment to absorb the learning curve. The single-node HA gap is accepted for Phase 0 and **must be revisited before Phase 1 launch** (likely by moving to ≥2 nodes across zones).

#### Database: Firestore vs Cloud SQL (Postgres)

PRD §Scalability requires *"idempotent credit operations and a transactional credit ledger to remain correct under concurrent load."* HLD entities (User ↔ StoryConfig ↔ Story, plus a credit ledger and subscription state) look more **relational** than document-store-shaped.

- **Firestore** supports transactions but only on a bounded doc set; query model is thin (no joins, limited filters).
- **Cloud SQL (Postgres)** is the idiomatic GCP answer for transactional cores; proven read replicas + connection pooling at the 10k MAU level.
- **SQLite file** is fine for static, read-only reference data (PhonicWords) baked into the worker image — but should be called out explicitly; it is not a horizontally scalable primary store.

**Recommendation:** Cloud SQL (Postgres) for the transactional core. Use Firestore only if a real doc-store use case appears.

#### Language stack: Go + Python

Two-language stacks mean two CI pipelines, two dependency systems, two deploy stories. Python is justified if the LLM/worker pulls in a Python-specific ecosystem (LangChain, certain SDKs); otherwise Go can call Vertex AI and most LLM APIs directly. Worth making this a conscious "Python buys us X" decision rather than a default.

### 10.3 Gaps vs HLD / PRD

Items the PRD or HLD explicitly require that this doc has not yet addressed:

| Missing                                                       | Source                                               |
| ------------------------------------------------------------- | ---------------------------------------------------- |
| **Cloud CDN** in front of GCS                                 | PRD §Scalability: *"CDN-fronted for read traffic"*   |
| **Load Balancer** (Cloud Load Balancing)                      | HLD diagram: `Client → LB → Server`                  |
| **Session strategy** — JWT vs shared store (Memorystore?)     | PRD: *"sessions externalised"*                       |
| **Vertex AI / Imagen 4** project + quota setup                | PRD Dependencies §1                                  |
| **Stripe** integration boundary                               | Phase 0 requirement                                  |
| **Web Push infra** (VAPID keys, service worker)               | Phase 0 requirement                                  |
| **Secret Manager** for API keys (LLM, Stripe, VAPID)          | Implied by integrations                              |
| **Environment strategy** (dev / staging / prod) + CI/CD       | Standard ops hygiene                                 |
| **Region choice**                                             | Cost + latency for target user geography             |
| **Alert specifics** — queue depth, generation success rate     | PRD Risk Register                                    |
| **Backup / DR** for credit ledger                             | Implied by "transactional credit ledger"             |

### 10.4 Suggested next moves

1. **Compute decided** — GKE Standard for main (learning-oriented; start on e2-medium), Cloud Run for the worker. **Add a Phase 1 follow-up: move to ≥2 nodes across zones for HA before public launch.**
2. **Re-decide DB** — justify Firestore against the credit-ledger requirement, or swap to Cloud SQL Postgres for the transactional core.
3. **Fill the gaps** — CDN, LB, session store, Secret Manager, Vertex AI, payments boundary.
4. **Add environments + CI/CD** before this doc becomes a build plan.
