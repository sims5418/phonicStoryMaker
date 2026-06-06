# Cost Model — Unit Economics & Scale

> Sizes the real $ picture of the phonicStoryMaker stack against the planned **$4.99/month subscription**. The dominant cost is **LLM + Imagen API calls per story**, not GCP infrastructure.

---

## 1 Assumptions

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

## 2 Revenue per paying user

| Channel | Gross | Provider fee | Net to us |
| --- | --- | --- | --- |
| **Stripe (Web, Phase 0–1)** | $4.99 | ~3% | **~$4.84** |
| **Apple IAP (iOS, Phase 2+)** | $4.99 | 30% | **$3.49** |
| **Google Play (Android, Phase 3+)** | $4.99 | 30% (15% after year 1) | **$3.49 → $4.24** |

## 3 Variable cost per user per month — all sources

Every cost that scales with users, broken out by service.

### Vertex AI (dominant)

| Utilization scenario | Stories/mo | Vertex AI cost/mo |
| --- | --- | --- |
| 100% of monthly credits | 4.0 | $1.60 |
| Realistic (65%) | 2.6 | $1.04 |
| Realistic + 10% rejection regens | 2.9 | $1.16 |
| Realistic + regens + failure overhead | 3.0 | **~$1.20** |
| Pessimistic upper bound | 4.4 | **~$1.80** |

### Other per-user GCP variable costs

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

### Total variable per user

| Scenario | Vertex AI | Other GCP | **Total** |
| --- | --- | --- | --- |
| Mid-case | $1.20 | $0.02 | **~$1.22** |
| Mid-case (rounded for margin math below) | — | — | **$1.50** |
| Pessimistic | $1.80 | $0.05 | **~$1.85** |

Non-Vertex GCP variable is rounding error vs. the AI APIs. Margin calculations below use **$1.50** as the mid-case variable cost per user.

## 4 Gross margin per paying user (all Apple)

| Channel | Net revenue | Variable cost (mid: $1.50) | Margin $ | Margin % |
| --- | --- | --- | --- | --- |
| **Apple IAP** *(operating assumption)* | **$3.49** | $1.50 | **$1.99** | **57%** |
| Pessimistic (variable cost $1.85) | $3.49 | $1.85 | $1.64 | 47% |
| Best case (variable cost $1.22) | $3.49 | $1.22 | $2.27 | 65% |

> **Verdict (Apple-only):** margin per user is **$1.64 – $2.27**. Healthy but tighter than a blended-channel model — every dollar of fixed cost takes ~50% more users to absorb than under a Stripe-heavy mix.

## 5 Total monthly profit at scale (all Apple)

| MAU | Phase | Revenue/user | Variable cost/user | Fixed cost (§6) | **Total net/mo** |
| --- | --- | --- | --- | --- | --- |
| 100 | Phase 0 | $3.49 | $1.50 | $55 | **+$144** |
| 1,000 | Phase 1 (1k) | $3.49 | $1.50 | $135 | **+$1,855** |
| 10,000 | Phase 1 target | $3.49 | $1.50 | $200 | **+$19,700** |

*Numbers ignore first-month-free drag (modeled separately in the §6 break-even).*

## 6 Break-even analysis (all Apple)

**Break-even** = fixed monthly costs ÷ contribution margin per paying user.
*Contribution margin = $3.49 net − $1.50 variable = **$1.99/user** (mid-case).*

### Phase 0 fixed monthly costs — full GCP service breakdown

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

### Phase 1+ added fixed costs (HA, Cloud SQL, CDN, scale)

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

### Variable cost contribution (per paying user, mid-case)

| Source | Per-user/mo |
| --- | --- |
| Vertex AI (Imagen + LLM) | $1.20 |
| All other per-user GCP (Cloud Run, GCS, egress, logging) | $0.02 |
| **Total (used in margin math)** | **$1.50** *(rounded with headroom)* |

### Break-even by phase

| Phase | Fixed cost | Margin/user (Apple) | **Break-even paying users** |
| --- | --- | --- | --- |
| **Phase 0** | $55 | $1.99 | **~28** |
| **Phase 1 (1k MAU)** | $135 | $1.99 | **~68** |
| **Phase 1 target (10k MAU)** | $200 | $1.99 | **~101** |

### With first-month-free drag

PRD gives every new user a free first month. If ~20% of MAU sit in their free month at any time, those users cost ~$1.50 in variable spend but contribute **$0** in revenue.

Effective margin/user = 0.8 × $1.99 − 0.2 × $1.50 = **$1.29**

| Phase | Fixed cost | Effective margin/user | **Break-even paying users** |
| --- | --- | --- | --- |
| Phase 0 | $55 | $1.29 | **~43** |
| Phase 1 (1k MAU) | $135 | $1.29 | **~105** |
| Phase 1 target (10k MAU) | $200 | $1.29 | **~155** |

### Verdict

- **Phase 0 breaks even at ~28–43 paying users** (with vs. without free-month drag). The PRD's 6-month target of 100 paying subscribers clears this with ~2–3× headroom — slimmer than the mixed-channel model showed, but still positive.
- **Phase 1 (1k MAU) breaks even at ~68–105 paying users.** Hitting the 1,000-MAU target with ≥10% paying conversion clears it.
- **Phase 1 target (10k MAU) breaks even at ~101–155 paying users.** With 10k MAU at >2% paying, comfortable.
- **Apple's 30% cut effectively doubles the break-even count vs. a Stripe-heavy mix.** Any path to drive even partial web/Stripe signups (e.g. a "subscribe on web" prompt before iOS install) shifts these numbers materially.
- **Excluded from this break-even:** team salaries, marketing, Apple Developer fees ($99/yr), legal/compliance, domain/email/Web Push services. *Business* break-even depends mostly on those, not on infra.

## 7 Sensitivities & risks

- **First month free** (PRD): every signup costs ~$1.50 in month 1 with $0 revenue. Effectively a $1.50 CAC floor; recovered after **~1 paid month** on Apple. If churn after free month exceeds ~50%, payback breaks down.
- **Image count drift:** stories average ~10 sentences = 10 images. If stories trend longer (12–15 sentences), variable cost jumps proportionally — a 15-image story costs $0.60, eroding Apple margin from $1.99 → $1.39 and pushing Phase 0 break-even from ~28 to ~40 users.
- **Imagen 4 price changes** are the single biggest external risk to this model. Mitigated by the provider-agnostic image abstraction (PRD §Architecture) — cheaper providers (e.g. SDXL self-hosted) become viable if Imagen 4 prices rise.
- **Credit-pack purchases** are pure upside, not modeled here. If even 10% of users buy one extra pack/month, margin per user rises materially.
- **All-Apple is a deliberately conservative model.** Real-world will include some Stripe-Web users at $3.34 margin each (~67% higher). Every 10% of MAU shifted from Apple to Stripe reduces Phase 1 break-even by ~7 users.

## 8 Cost-control tasks worth doing on day one

- **Billing budget alerts** at $100 / $500 / $1,000 — one stuck Imagen retry loop can burn fast.
- **Per-user generation rate limits** so a single account can't run away with API spend.
- **Cap on regeneration loops** — PRD allows 1 free regen on reject; enforce that strictly.
- **Daily cost dashboards** broken down by service (Imagen, LLM, GCS egress, Cloud Run) so anomalies show up within hours.
