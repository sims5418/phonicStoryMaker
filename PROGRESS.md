# Progress Log

**Last updated**: 2026-05-12 (v3 PRD)

---

## What We've Done

### Requirements Gathering — v3 COMPLETE
PRD updated to v3.0 at `doc/1_RequirementDoc.md` (quality score: 97/100). v3 folds in the May 12 meeting amendments on top of v2 (which was driven by the May 11 meeting notes). Committed as `dd40d34` and pushed to `origin/main`.

**Key v3 changes from v2.0:**
- **Client/server separation** elevated to a first-class architectural constraint — separated codebases, independently shippable clients (web, iPhone, iPad, Android), versioned API contract
- **Scalability target added**: backend must launch at 100 MAU and scale to **10,000 MAU via horizontal scaling alone** (no rewrite)
- New **Scalability** subsection in Technical Constraints with explicit design implications (stateless app tier, queue-based generation, externalised sessions/images, observability)
- 3 new scaling-related risks added to Risk Assessment (rewrite avoidance, database bottleneck, queue saturation)
- New dependency: cloud infrastructure choice must support horizontal scaling from day one — to be decided in high-level design
- Glossary: added MAU definition

**Key v2 changes from v1.1 (unchanged history):**
- **Credit model**: weekly login credit removed; replaced by **4 credits/month auto-issued on subscription renewal**, expire at next renewal. Purchased pack credits unchanged (no expiry).
- **KPIs**: reframed as monthly; primary acquisition metric → **paid subscriptions (100+ in 6 months)**, replacing registrations. Engagement metric → **total credit-pack purchase amount** (replaces conversion rate); monthly notification open rate dropped.
- **Launch sequence**: Web (Stripe) → iPhone → iPad → Android. Backend shared from day one.
- **Safety (new section)**: parent must **review & accept** every story before it enters the catalog; reject = one free regeneration. **Report flow**: immediately hide from all viewers + auto-delete after 30 days (no human moderator queue in v2).
- **Story creation**: character name max **16 chars** (was 32); **no loading screen** — **Web Push notification** when story is ready; phonics-level chooser UI always shown even though last-used level is pre-selected.
- **Phasing**: added **Phase 0 (PoC, web only)** — auth, 2 tabs (Create + Catalog), creation, moderation, LLM, Imagen 4 (abstracted), subscription/monthly credits. Phase 0 exit = **~1 month stable in production, no major bugs** (qualitative — to be sharpened in HLD). Word Bank, Trophies, Settings tab, monthly email, credit packs, example stories all move to **Phase 1**.
- **Image generation**: must be **abstracted behind a provider-agnostic backend interface**; Imagen 4 is default but swappable.
- **Monthly email**: reads + words + **trophy status** merged into one combined monthly email.
- **Terminology**: "User Stories" → "Epics". Child is **not** an independent user in *any* version.

**Open decisions captured as TBDs in the PRD:**
- Monthly subscription price
- 1-pack and 4-pack credit prices
- Concrete Phase 0 exit metric thresholds (e.g. generation success rate)
- Named fallback image provider for the abstraction

---

## Next Steps

1. **High-level design** at `doc/highLevelDesign.md` (currently empty — only contains `/todo`)
   - **Tech stack** (frontend framework, backend language, DB) — DB choice must support horizontal scaling (read replicas, partitioning) per v3
   - **LLM API provider** (OpenAI / Anthropic / Google Gemini) — and decide whether to abstract it
   - **Image-provider abstraction interface** (required by v2)
   - **Scalability architecture (new in v3)**:
     - Stateless app tier (externalised sessions — JWT or shared store)
     - Async generation queue + worker pool (so web tier scales independently)
     - Object storage + CDN for generated images
     - Cloud platform selection (managed compute, managed DB, managed queue)
     - Observability stack (metrics, structured logs, traces) from day one
   - **Client/server split (reinforced in v3)**: define versioned API contract; clients (web, iPhone, iPad, Android) deploy independently
   - **Data model** (users, stories incl. pending-review state, images, credits incl. expirable vs. non-expirable, word bank, report state)
   - **API surface** for web (Phase 0) reusable by iOS/iPad/Android
   - **Web Push** integration (VAPID keys, service worker)
   - **Phase 0 exit thresholds** — concrete numbers (generation success rate, latency)
   - **Prototype Imagen 4 character consistency** before architecture is locked

2. **Decide pricing** (monthly subscription fee + 1-pack & 4-pack prices) — needed before billing integration

3. **Identify one fallback image provider** before Phase 0 launch to validate the abstraction

4. **Load-test plan** (new in v3): validate the 100 → 10,000 MAU horizontal-scaling path before Phase 1 launch

5. **Project plan**
   - Break Phase 0 → Phase 1 → Phase 2 → Phase 3 into milestones/sprints
   - Identify dependencies (GCP/Vertex AI, Stripe, Jolly Phonics word lists, email provider, Web Push infra, Apple Developer + Google Play accounts for later phases)

6. **Early prototype** (recommended before Phase 0 implementation): Imagen 4 character consistency via reference image — still a blocker risk
