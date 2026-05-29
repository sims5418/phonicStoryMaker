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

- **Choice:** **GKE Standard** *(kept primarily as a learning exercise — see §9.2 for the honest trade-off)*
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

## 9 Assessment vs HLD / PRD

A self-review against `doc/1_RequirementDoc.md` (v3) and `doc/2_HighLevelDesign_P0.md`.

### 9.1 What holds up

- **Cloud Run for story generation** — bursty queue-triggered workload (15s text + 10s × N images) fits Cloud Run's scale-to-zero model. Pub/Sub push subscription or Eventarc triggers it cleanly.
- **GCS for blobs** — directly satisfies PRD §Scalability ("object storage for generated images").
- **Pub/Sub** — clean SQS analog; HLD already assumes a queue tier.
- **Firebase Auth** — 50k MAU free tier clears the 10k MAU target; parent-only account model maps fine.
- **Terraform** — CloudFormation has no GCP equivalent; Terraform also keeps multi-cloud options open.
- **Cloud Logging + Monitoring free tier** — accurate for expected volume.

### 9.2 Decisions worth re-examining

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

### 9.3 Gaps vs HLD / PRD

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

### 9.4 Suggested next moves

1. **Compute decided** — GKE Standard for main (learning-oriented; start on e2-medium), Cloud Run for the worker. **Add a Phase 1 follow-up: move to ≥2 nodes across zones for HA before public launch.**
2. **Re-decide DB** — justify Firestore against the credit-ledger requirement, or swap to Cloud SQL Postgres for the transactional core.
3. **Fill the gaps** — CDN, LB, session store, Secret Manager, Vertex AI, payments boundary.
4. **Add environments + CI/CD** before this doc becomes a build plan.

---

## 10 Cost Model — Unit Economics & Scale

The dominant cost is **LLM + Imagen API calls per story**, not GCP infrastructure. This section sizes the real $ picture against the planned **$4.99/month subscription**.

### 10.1 Assumptions

| Variable | Value | Source |
| --- | --- | --- |
| Subscription price | **$4.99/mo** | Pricing decision |
| **Customer channel — operating assumption** | **100% Apple IAP** (worst-case) | Conservative planning baseline |
| Apple IAP fee | 30% → you net **$3.49** | Apple policy |
| Monthly credits issued | 4 / user (expire on renewal) | PRD Epic 5 |
| Cost per story (Imagen 4: ~10 images × $0.04 + LLM text ~$0.005) | **~$0.40** | Public Imagen 4 pricing |
| Realistic credit utilization | ~65% (≈2.6 stories/user/mo) | Assumption — needs validation |
| Reject + free-regen overhead | +10% | PRD Epic 2 |
| Generation failure refund overhead | ~5% × $0.40 | PRD §Story Creation |

> **Worst-case model:** all paying customers are assumed to be on Apple IAP (30% fee). Web/Stripe and Android users are bonus revenue and not modeled. This gives a conservative floor for every margin and break-even figure below.

### 10.2 Revenue per paying user

| Channel | Gross | Provider fee | Net to us |
| --- | --- | --- | --- |
| **Stripe (Web, Phase 0–1)** | $4.99 | ~3% | **~$4.84** |
| **Apple IAP (iOS, Phase 2+)** | $4.99 | 30% | **$3.49** |
| **Google Play (Android, Phase 3+)** | $4.99 | 30% (15% after year 1) | **$3.49 → $4.24** |

### 10.3 Variable cost per user per month — all sources

Every cost that scales with users, broken out by service.

#### 10.3a Vertex AI (dominant)

| Utilization scenario | Stories/mo | Vertex AI cost/mo |
| --- | --- | --- |
| 100% of monthly credits | 4.0 | $1.60 |
| Realistic (65%) | 2.6 | $1.04 |
| Realistic + 10% rejection regens | 2.9 | $1.16 |
| Realistic + regens + failure overhead | 3.0 | **~$1.20** |
| Pessimistic upper bound | 4.4 | **~$1.80** |

#### 10.3b Other per-user GCP variable costs

| Service | Driver | Per-user/mo |
| --- | --- | --- |
| Cloud Run worker compute | 30 s @ 2 vCPU / 1 GB × ~3 generations | <$0.01 |
| GCS storage (mature, ~50 MB accumulated/user) | $0.020 / GB-mo | ~$0.001 |
| GCS egress (image serving, ~50 MB/user/mo to internet) | $0.12 / GB | ~$0.006 |
| Pub/Sub | a few KB per generation, free tier covers | $0 |
| Firestore reads/writes (Phase 0) | well inside free tier at 100 MAU | $0 |
| Cloud SQL queries (Phase 1+) | included in fixed instance cost | $0 |
| Cloud Logging beyond free tier (at scale) | log volume per user | <$0.005 |
| Cloud Monitoring, Cloud Trace | inside free tier | $0 |
| Cloud CDN cache-fill / egress (post-CDN) | reduces GCS egress at scale | net **negative** |
| **Subtotal — non-Vertex variable GCP** | | **~$0.02** |

#### 10.3c Total variable per user

| Scenario | Vertex AI | Other GCP | **Total** |
| --- | --- | --- | --- |
| Mid-case | $1.20 | $0.02 | **~$1.22** |
| Mid-case (rounded for margin math below) | — | — | **$1.50** |
| Pessimistic | $1.80 | $0.05 | **~$1.85** |

Non-Vertex GCP variable is rounding error vs. the AI APIs. Margin calculations below use **$1.50** as the mid-case variable cost per user.

### 10.4 Gross margin per paying user (all Apple)

| Channel | Net revenue | Variable cost (mid: $1.50) | Margin $ | Margin % |
| --- | --- | --- | --- | --- |
| **Apple IAP** *(operating assumption)* | **$3.49** | $1.50 | **$1.99** | **57%** |
| Pessimistic (variable cost $1.85) | $3.49 | $1.85 | $1.64 | 47% |
| Best case (variable cost $1.22) | $3.49 | $1.22 | $2.27 | 65% |

> **Verdict (Apple-only):** margin per user is **$1.64 – $2.27**. Healthy but tighter than a blended-channel model — every dollar of fixed cost takes ~50% more users to absorb than under a Stripe-heavy mix.

### 10.5 Total monthly profit at scale (all Apple)

| MAU | Phase | Revenue/user | Variable cost/user | Fixed cost (§10.6) | **Total net/mo** |
| --- | --- | --- | --- | --- | --- |
| 100 | Phase 0 | $3.49 | $1.50 | $55 | **+$144** |
| 1,000 | Phase 1 (1k) | $3.49 | $1.50 | $135 | **+$1,855** |
| 10,000 | Phase 1 target | $3.49 | $1.50 | $200 | **+$19,700** |

*Numbers ignore first-month-free drag (modeled separately in §10.6 break-even).*

### 10.6 Break-even analysis (all Apple)

**Break-even** = fixed monthly costs ÷ contribution margin per paying user.
*Contribution margin = $3.49 net − $1.50 variable = **$1.99/user** (mid-case).*

#### Phase 0 fixed monthly costs — full GCP service breakdown

| Service | Cost/mo | Notes |
| --- | --- | --- |
| GKE Standard — cluster management fee | **$0** | First cluster free |
| GKE Standard — 1× e2-medium node | **$24** | Baseline VM (4 GB / 2 vCPU burst) |
| GKE — boot disk (50 GB balanced PD) | **$5** | |
| Cloud Load Balancer — base + 1 forwarding rule | **$18** | HTTPS LB hourly |
| Cloud Load Balancer — extra forwarding rules | **$7** | Up to 5 included |
| Static external IP (attached) | **$0** | Free while in use |
| Cloud Run worker (idle) | **$0** | Scales to zero |
| Pub/Sub | **$0** | 10 GB/mo free, our volume tiny |
| GCS — storage | **<$1** | Minimal data at 100 MAU |
| GCS — egress (internet) | **<$1** | Low at 100 MAU |
| Firestore | **$0** | Free tier covers (50k reads/day) |
| Firebase Authentication | **$0** | Free under 50k MAU |
| Cloud Logging | **$0** | First 50 GB/mo free |
| Cloud Monitoring | **$0** | Native metrics free |
| Cloud Trace | **$0** | First 2.5M spans/mo free |
| Artifact Registry | **~$1** | Container image storage |
| Secret Manager | **$0** | First 6 secrets free |
| Cloud Build (CI) | **$0** | 120 build-min/day free |
| Vertex AI (Imagen + LLM, idle) | **$0** | Pay per request — see variable |
| **Phase 0 fixed total** | **~$55/mo** | |

#### Phase 1+ added fixed costs (HA, Cloud SQL, CDN, scale)

| Service | Added cost/mo | Notes |
| --- | --- | --- |
| 2nd GKE node + disk (HA across zones) | +$29 | 2× e2-medium for zone redundancy |
| Cloud SQL (db-g1-small, regional) | +$50 | Transactional store for credit ledger |
| Cloud CDN base | +$0 | Pay-per-request, ~$0.02/GB cache-egress |
| Cloud NAT (if private GKE adopted) | +$32 *(optional)* | Only if locking down networking |
| Memorystore Redis (if shared sessions needed) | +$30 *(optional)* | Skip if JWT-based sessions |
| Logging beyond free tier (at 10k MAU) | +$0–$30 | May exceed 50 GB/mo |
| **Phase 1 (1k MAU) total** | **~$135/mo** | |
| **Phase 1 target (10k MAU) total** | **~$200/mo** | (depending on logging + optional services) |

#### Variable cost contribution (per paying user, mid-case)

| Source | Per-user/mo |
| --- | --- |
| Vertex AI (Imagen + LLM) | $1.20 |
| All other per-user GCP (Cloud Run, GCS, egress, logging) | $0.02 |
| **Total (used in margin math)** | **$1.50** *(rounded with headroom)* |

#### Break-even by phase

| Phase | Fixed cost | Margin/user (Apple) | **Break-even paying users** |
| --- | --- | --- | --- |
| **Phase 0** | $55 | $1.99 | **~28** |
| **Phase 1 (1k MAU)** | $135 | $1.99 | **~68** |
| **Phase 1 target (10k MAU)** | $200 | $1.99 | **~101** |

#### With first-month-free drag

PRD gives every new user a free first month. If ~20% of MAU sit in their free month at any time, those users cost ~$1.50 in variable spend but contribute **$0** in revenue.

Effective margin/user = 0.8 × $1.99 − 0.2 × $1.50 = **$1.29**

| Phase | Fixed cost | Effective margin/user | **Break-even paying users** |
| --- | --- | --- | --- |
| Phase 0 | $55 | $1.29 | **~43** |
| Phase 1 (1k MAU) | $135 | $1.29 | **~105** |
| Phase 1 target (10k MAU) | $200 | $1.29 | **~155** |

#### Verdict

- **Phase 0 breaks even at ~28–43 paying users** (with vs. without free-month drag). The PRD's 6-month target of 100 paying subscribers clears this with ~2–3× headroom — slimmer than the mixed-channel model showed, but still positive.
- **Phase 1 (1k MAU) breaks even at ~68–105 paying users.** Hitting the 1,000-MAU target with ≥10% paying conversion clears it.
- **Phase 1 target (10k MAU) breaks even at ~101–155 paying users.** With 10k MAU at >2% paying, comfortable.
- **Apple's 30% cut effectively doubles the break-even count vs. a Stripe-heavy mix.** Any path to drive even partial web/Stripe signups (e.g. a "subscribe on web" prompt before iOS install) shifts these numbers materially.
- **Excluded from this break-even:** team salaries, marketing, Apple Developer fees ($99/yr), legal/compliance, domain/email/Web Push services. *Business* break-even depends mostly on those, not on infra.

---

### 10.7 Sensitivities & risks

- **First month free** (PRD): every signup costs ~$1.50 in month 1 with $0 revenue. Effectively a $1.50 CAC floor; recovered after **~1 paid month** on Apple. If churn after free month exceeds ~50%, payback breaks down.
- **Image count drift:** stories average ~10 sentences = 10 images. If stories trend longer (12–15 sentences), variable cost jumps proportionally — a 15-image story costs $0.60, eroding Apple margin from $1.99 → $1.39 and pushing Phase 0 break-even from ~28 to ~40 users.
- **Imagen 4 price changes** are the single biggest external risk to this model. Mitigated by the provider-agnostic image abstraction (PRD §Architecture) — cheaper providers (e.g. SDXL self-hosted) become viable if Imagen 4 prices rise.
- **Credit-pack purchases** are pure upside, not modeled here. If even 10% of users buy one extra pack/month, margin per user rises materially.
- **All-Apple is a deliberately conservative model.** Real-world will include some Stripe-Web users at $3.34 margin each (~67% higher). Every 10% of MAU shifted from Apple to Stripe reduces Phase 1 break-even by ~7 users.

### 10.8 Cost-control tasks worth doing on day one

- **Billing budget alerts** at $100 / $500 / $1,000 — one stuck Imagen retry loop can burn fast.
- **Per-user generation rate limits** so a single account can't run away with API spend.
- **Cap on regeneration loops** — PRD allows 1 free regen on reject; enforce that strictly.
- **Daily cost dashboards** broken down by service (Imagen, LLM, GCS egress, Cloud Run) so anomalies show up within hours.

---

## 11 Environments

Two-environment promotion pipeline:

| Environment | Purpose | Naming |
| --- | --- | --- |
| **gamma** | Pre-production / staging. Full end-to-end testing on real GCP infra (real Cloud Run, GKE, Pub/Sub, etc.) with **test-mode** payment providers, **isolated database**, and synthetic users. Last gate before prod. | `*-gamma` resource suffixes; separate GCP project recommended |
| **prod** | Live customer-facing environment. Real payments, real users. | `*-prod` suffixes; separate GCP project |

**Promotion flow:** code change → CI → deploy to **gamma** → run smoke/integration tests → manual promote to **prod**.

**Open questions to settle before build:**
- Single GCP project with resource prefixes, or separate projects per env? *(Separate projects is the GCP-idiomatic answer — cleaner IAM boundary, simpler billing breakdown, no accidental cross-env access.)*
- Do we need a developer-local "dev" environment beyond emulators? *(Deferred per §9.3 — emulator story is out of scope for now.)*
