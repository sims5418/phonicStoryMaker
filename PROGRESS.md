# Progress Log

**Last updated**: 2026-06-06 (cost model split into its own doc; infra sections reordered)

---

## What We've Done

### GCP Infrastructure Investigation
`doc/3_infra_investigation_gcp.md` committed as `392ccd0`. Captures:
- **AWS → GCP mapping** for every infra concern (compute, storage, queue, DB, auth, IaC)
- **Compute decisions:**
  - Main server: **GKE Standard** (1× e2-medium for Phase 0; learning-driven choice; HA gap accepted for PoC, revisit before Phase 1)
  - Story-generation worker: **Cloud Run Service + Pub/Sub push subscription** (concurrency=1, bounded max-instances)
- **Region:** `us-east1` for Phase 0 (EU region deferred until GDPR-K traffic justifies it)
- **Other picks:** Pub/Sub (queue), GCS (blob), Firebase Auth (<50k MAU free), Terraform (IaC), Go (general server), Python (LLM worker)
- **DB direction:** Firestore pencilled in, but transactional credit ledger needs its own design doc — flagged as a follow-up
- **Environments:** two-stage promotion — `gamma` (staging) → `prod`, separate GCP projects recommended
- **Environments (§9):** two-stage promotion — `gamma` (staging) → `prod`, separate GCP projects recommended
- **Self-assessment (§10):** decisions worth re-examining (GKE 1-node HA, Firestore vs Cloud SQL, Go+Python overhead) and gaps vs HLD/PRD (CDN, LB config, session strategy, Vertex AI quota, Secret Manager, etc.)

### Cost Model — split into its own doc
`doc/4_cost_model.md` extracted from the infra investigation (commit `c7bb274`, pushed). At the same time the infra doc's sections were reordered (Environments → §9, Assessment → §10). Cost model captures, under conservative **all-Apple revenue** assumption ($3.49 net/user after 30% IAP fee):
- Full GCP service breakdown — Phase 0 fixed cost **~$55/mo** across 18 line items; Phase 1+ **~$135–$200/mo** (HA, Cloud SQL, CDN added)
- Variable cost per user **~$1.50/mo** (dominated by Vertex AI)
- **Break-even: ~28–43 paying users for Phase 0**, ~101–155 at the 10k-MAU target
- 6-month PRD goal of 100 paying subscribers clears Phase 0 break-even with 2–3× headroom

### High-Level Design — P0 drafted
`doc/2_HighLevelDesign_P0.md` committed as `e996e4a`. Captures:
- Functional requirements: story creation, library, parent review
- Non-functional: LLM text, one image/sentence, poll-based status (push dropped), monthly billing, 4 credits/month
- Entities: User, StoryConfig (with `pending`/`accepted`/`reported` status), Story, PhonicWords
- System architecture: Client → LB → Server → Queue → Worker, with separate StoryConfig/User/Story/PhonicWords stores + Blob storage
- API: `POST /stories/create`, `GET /stories`, `GET /stories/{id}`, `PUT /stories/{id}` (accept/reject)
- Deep-dive TODO table (5 rows)

### P0 issue backlog filed on GitHub
One issue per HLD TODO row, all titled with `[P0]` prefix per May 20 decision:
- #4 — Story Creation — DB, blob storage, worker, queue
- #5 — Phonics Word DB — research and populate
- #6 — Notification — client poll-based status
- #7 — Payment — provider integration design
- #8 — Credits — system design

### Requirements Gathering — v3 (prior)
PRD at `doc/1_RequirementDoc.md` (quality 97/100). v3 (commit `dd40d34`) adds:
- Client/server separation as a first-class constraint (web + iPhone + iPad + Android clients, shared versioned API)
- Scalability target: launch at 100 MAU, scale to **10,000 MAU via horizontal scaling alone**
- Stateless app tier, queue-based generation, externalised sessions/images, observability from day one

v2 (prior, commit `2854319`) introduced: monthly auto-issued credits (4/mo, expire at renewal); paid-subscription KPI; web→iPhone→iPad→Android launch order; parent review + report flow; Phase 0 (PoC, web only); abstracted image provider; "Epics" terminology.

**Open PRD TBDs (still open):**
- Monthly subscription price
- 1-pack and 4-pack credit prices
- Concrete Phase 0 exit metric thresholds
- Named fallback image provider

---

## Next Steps

1. **Transaction system design doc** — credit / subscription / payment ledger needs its own design (idempotency keys, Pub/Sub at-least-once handling, ledger schema). Blocks final DB pick (Firestore vs Cloud SQL).

2. **Work remaining P0 issues** (#4–#8):
   - #4 Story Creation — partially covered by infra doc (queue + worker + blob storage decided); DB design still open pending the transaction doc above
   - #5 Phonics Word DB — open
   - #6 Notification (poll-based) — open
   - #7 Payment integration — open; needs pricing decision first
   - #8 Credits — folds into the transaction system design

3. **Fill infra-doc gaps before build** (from §10.3):
   - Cloud CDN in front of GCS
   - Cloud Load Balancer config
   - Session strategy (JWT vs shared store)
   - Vertex AI / Imagen 4 quota request (lead time is days, not minutes)
   - Stripe boundary, Web Push (VAPID + service worker)
   - Secret Manager, billing budget alerts, CI/CD on Cloud Build
   - Single GCP project vs separate `gamma`/`prod` projects (lean toward separate)

4. **HLD gaps not in the TODO table** — consider filing follow-up issues for:
   - Auth mechanism details (Firebase Auth picked; JWT vs session — still TBD, must be stateless-compatible)
   - Parent review UX + reject → free-regeneration loop (PRD v2 safety section)
   - Phase 0 exit thresholds (concrete numbers for generation success rate, latency)
   - Image-provider abstraction interface + named fallback provider
   - Load-test plan validating 100 → 10,000 MAU horizontal-scaling path

5. **Decide pricing** (1-pack & 4-pack) — monthly is $4.99 (locked); pack prices still TBD. Needed before billing integration (#7).

6. **Prototype Imagen 4 character consistency** before architecture is locked — still a blocker risk.

7. **Project plan** — break Phase 0 → 1 → 2 → 3 into milestones once design issues are resolved. Dependencies: GCP/Vertex AI, Stripe, Jolly Phonics word lists, email provider, Apple Developer + Google Play accounts (later phases).

8. **Rotate the GitHub PAT** that was pasted in chat earlier today.
