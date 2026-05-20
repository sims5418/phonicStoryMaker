# Progress Log

**Last updated**: 2026-05-20 (HLD P0 + issues filed)

---

## What We've Done

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

1. **Work the P0 issues** (#4–#8) — each is a design task; deliverable is a design doc / decision recorded in the HLD.

2. **HLD gaps not in the TODO table** — consider filing follow-up issues for:
   - Authentication mechanism (session vs JWT; must be stateless-compatible)
   - Parent review UX + reject → free-regeneration loop (PRD v2 safety section)
   - Phase 0 exit thresholds (concrete numbers for generation success rate, latency)
   - Image-provider abstraction interface + named fallback provider
   - Load-test plan validating 100 → 10,000 MAU horizontal-scaling path

3. **Decide pricing** (monthly subscription + 1-pack & 4-pack) — needed before billing integration (#7).

4. **Prototype Imagen 4 character consistency** before architecture is locked — still a blocker risk.

5. **Project plan** — break Phase 0 → 1 → 2 → 3 into milestones once design issues are resolved. Dependencies: GCP/Vertex AI, Stripe, Jolly Phonics word lists, email provider, Apple Developer + Google Play accounts (later phases).

6. **Push `main`** — local is 1 commit ahead of `origin/main` (the HLD commit).

7. **Rotate the GitHub PAT** that was pasted in chat earlier today.
