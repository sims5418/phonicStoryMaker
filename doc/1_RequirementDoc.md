# Product Requirements Document: phonicStoryMaker

**Version**: 3.0
**Date**: 2026-05-12
**Author**: Sarah (Product Owner)
**Quality Score**: 97/100

**Changelog (v2.0 → v3.0):**
- Architecture: client/server separation reinforced as a first-class constraint — single shared backend serves all clients (web, iPhone, iPad, Android); clients are independently shippable
- New scalability target: backend must launch supporting 100 monthly active users and scale to **10,000 MAU via horizontal scaling alone** (no architectural rewrite, no major refactor)
- Risk Assessment updated with scaling-related risks
- Dependencies updated to require infrastructure choices that support horizontal scaling from day one

**Changelog (v1.1 → v2.0):**
- Credit model: weekly login credit removed; replaced by monthly auto-issued credits on subscription renewal (expire if unused)
- KPIs reframed as monthly; primary user-acquisition metric switched from registrations to paid subscriptions
- Launch sequence added: Web (Stripe) → iPhone → iPad → Android
- New Safety section: parent review-before-catalog; report-to-hide flow with 30-day auto-purge
- Character name max length: 32 → 16 chars
- Loading screen replaced by Web Push notification on generation completion
- Phase 0 (Proof of Concept) added; several items moved from Phase 1 → Phase 0
- Monthly trophy status merged into the existing monthly progress email
- Image generation must be abstracted behind a provider-agnostic interface (Imagen 4 swappable)
- Terminology: "User Stories" → "Epics"
- Child is *not* an independent user in any version (not just v1)

---

## Executive Summary

phonicStoryMaker is a web application (with iOS/iPad/Android to follow) where parents and children (ages 3–8) co-create personalized, phonics-based illustrated stories to support early reading development. Parents input a phonics level, character, theme, and optional plot; an LLM generates a tailored story and a swappable image generation API produces one cartoon illustration per sentence with consistent character art throughout.

The product addresses a real gap: generic phonics readers don't feature a child's specific characters or themes, reducing engagement. By making stories personal and visually rich, phonicStoryMaker aims to build reading habits through repeated, enjoyable sessions parents and children share together.

A freemium model — free first month, then a monthly subscription — combined with monthly auto-issued credits and purchasable credit packs drives both acquisition and retention. Progress is tracked through a word bank and trophy milestones, surfaced in a combined monthly email to parents.

---

## Problem Statement

**Current Situation**: Parents of early readers struggle to find phonics-appropriate books that are also personally engaging to their child. Commercially available readers use generic characters and fixed storylines.

**Proposed Solution**: A web app (later iOS/iPad/Android) that generates personalized phonics stories with custom characters, themes, and per-sentence illustrations — with smart defaults so repeat creation takes seconds, and a parent-review gate before any story enters the catalog.

**Business Impact**: Drives a recurring engagement loop (subscribe → receive monthly credits → create → review → read → complete), supports a subscription + credit-pack revenue model, and builds a word-learning portfolio that increases long-term retention.

---

## Success Metrics

**Primary monthly KPIs (6-month targets):**
- **Paid subscriptions**: 100+ active paying parent accounts
- **Story reads completed**: 300+ per month
- **Stories created**: 30+ per month

**Engagement signals (monthly):**
- Weekly active users (parents logging in at least once per week)
- **Total credit-pack purchase amount** (gross revenue from packs)

**Validation**: Measured via app analytics at 3-month and 6-month post-launch checkpoints.

---

## Launch Platforms

Sequenced rollout — backend API is shared across all platforms from day one.

| Order | Platform | Payment | Notes |
|---|---|---|---|
| 1 | Web (responsive) | Stripe (~3–5% fee) | Phase 0 PoC launch surface |
| 2 | iPhone (iOS native) | Apple IAP (25–30% fee) | After Phase 0 exit |
| 3 | iPad (iOS native) | Apple IAP | Layout adaptation |
| 4 | Android native | Google Play Billing | Final platform |

---

## User Personas

### Primary: Parent / Caregiver
- **Role**: Creates stories, reviews & accepts them, reads with child, tracks progress
- **Goals**: Help child build reading skills in a fun, low-friction way
- **Pain Points**: Generic readers aren't engaging; finding level-appropriate content takes effort; trust that AI-generated content is age-appropriate
- **Technical Level**: Everyday smartphone/web user; not technical

### Secondary: Child (ages 3–8)
- **Role**: Listens/reads along with parent, earns trophies
- **Goals**: Enjoy the story, recognize their character's name, learn words
- **Technical Level**: **Not an independent user in any version** — parent operates the app

---

## Safety & Content Moderation

Two complementary layers protect the experience:

### Pre-display moderation (automated)
- Inputs (character name, theme, plot) screened for violence, profanity, inappropriate content
- LLM story output screened before image generation begins
- Celebrity / public-figure name blocklist + LLM-based name detection on character input

### Parent gating (human-in-the-loop)
- **Review before catalog**: After generation completes, the parent is shown the story and must explicitly accept it before it is added to the Story Catalog. Until accepted, it exists only in a transient "pending review" state.
- **Reject = one free regeneration**: If the parent rejects a generated story, the credit is consumed but they get a single free regeneration attempt for that submission.
- **Report flow**: From any story in the catalog, the parent can report it. Reported stories are **immediately hidden from all viewers** and **auto-deleted from the database after 30 days**. No human moderator queue in v2; report counts and patterns are logged for future tuning.

---

## Epics & Acceptance Criteria

### Epic 1: Create a Personalized Story

**As a** parent
**I want to** create a story by choosing a phonics level, character, theme, and optional plot
**So that** my child gets a personalized, level-appropriate reading experience

**Acceptance Criteria:**
- [ ] Parent can select a Jolly Phonics level from a defined list every time (the level chooser UI is always shown, even though the previously used level is pre-selected as default)
- [ ] Parent can enter a character name (**max 16 characters**; celebrity names blocked; filtered for inappropriate content)
- [ ] Parent can select a theme or choose "Random" (previously used theme is pre-selected as default)
- [ ] Parent can enter an optional 1-sentence story plot or choose "Random"
- [ ] On submission, 1 credit is deducted; **no loading screen is shown** — the parent can navigate away
- [ ] Parent receives a **Web Push notification** when the story is ready to review
- [ ] LLM output is screened for violent or inappropriate content before being presented for review
- [ ] If generation fails, the credit is refunded and an error message is shown with a retry option

### Epic 2: Review and Accept a Generated Story

**As a** parent
**I want to** review each generated story before it enters the catalog
**So that** I can ensure the content and images are appropriate before my child sees them

**Acceptance Criteria:**
- [ ] After generation, the parent sees the full story (text + per-sentence images) in a review view
- [ ] Parent can **Accept** → story is added to the Story Catalog
- [ ] Parent can **Reject** → story is discarded; one free regeneration of the same submission is offered (no additional credit charged)
- [ ] Rejected stories are not persisted to the catalog
- [ ] Only after acceptance does the story participate in word-bank and read-count tracking

### Epic 3: Review Phonics Words Before Reading

**As a** parent
**I want to** see the phonics words used in the story before reading it with my child
**So that** my child can preview and practice the words first

**Acceptance Criteria:**
- [ ] After acceptance, a phonics word list page is available before the reading view
- [ ] Each word corresponds to the selected Jolly Phonics level
- [ ] Parent can proceed to the story from this page

### Epic 4: Read and Complete a Story

**As a** parent
**I want to** read the story with my child and mark it as completed
**So that** progress is tracked and trophies are updated

**Acceptance Criteria:**
- [ ] Story is displayed with one illustration per sentence
- [ ] A "Mark as Completed" button is available on the story page
- [ ] Completing a story increments the completed reads count
- [ ] Words from the completed story are added to the child's word bank

### Epic 5: Receive Monthly Credits via Subscription

**As a** subscribed parent
**I want to** automatically receive story credits each month on my renewal date
**So that** I can keep creating stories without buying packs every time

**Acceptance Criteria:**
- [ ] On each monthly subscription renewal date, the parent's account is auto-credited with **4 credits**
- [ ] **Unused monthly credits expire** when the next renewal date arrives (do not roll over)
- [ ] Credit balance is visible in the UI at all times
- [ ] Credit pack purchases (below) are *separate* from monthly credits and do **not** expire

### Epic 6: Buy Additional Credit Packs

**As a** parent
**I want to** buy more story credits when I want extra
**So that** I can keep creating stories beyond my monthly allocation

**Acceptance Criteria:**
- [ ] Two pack options are available: 1 story for $X, or 4 stories for $X (prices configurable, **TBD**)
- [ ] Purchase flow is accessible from Settings
- [ ] After purchase, credits are added to balance immediately
- [ ] When credit balance is 0, the Create Story tab is still accessible (no purchase prompt overlay) — the parent can navigate to Settings to buy a pack if they choose

### Epic 7: Track Learning Progress

**As a** parent
**I want to** see which phonics words my child has learned and how many stories we've completed
**So that** I can understand their progress over time

**Acceptance Criteria:**
- [ ] Word Bank shows all unique phonics words from completed stories, divided by Jolly Phonics level
- [ ] Within each level section, words can be sorted by: most completed, least completed, alphabetical
- [ ] A trophy page (in Settings) shows earned and locked achievement milestones
- [ ] A **combined monthly email** summarises reads completed, words learned, and trophy status for the month

### Epic 8: Browse Story Catalog

**As a** parent
**I want to** revisit stories we've accepted and try example stories
**So that** we can re-read favourites and get started easily on first use

**Acceptance Criteria:**
- [ ] Story Catalog shows all accepted user-created stories
- [ ] Stories can be sorted by last read or alphabetically
- [ ] 3 example stories provided by the app are always visible in the catalog
- [ ] Each story has a "Report" action; reported stories are hidden from all viewers immediately and auto-deleted after 30 days
- [ ] Tapping a story opens it directly to the reading view

---

## Functional Requirements

### Feature 1: Story Creation

**Inputs (4 fields):**

| Field | Required | Default Behavior | Random Option |
|---|---|---|---|
| Phonics Level (Jolly Phonics) | Yes | Last used level pre-selected; chooser UI always shown | No |
| Character | Yes | Last used character | No |
| Theme | Yes | Last used theme | Yes |
| Story Plot (1 sentence/phrase) | No | Empty | Yes |

**Character input rules:**
- **Max 16 characters**
- Celebrity / real public figure names blocked
- Profanity and inappropriate content filtered

**Generation flow:**
1. Parent submits inputs → 1 credit deducted
2. User input screened for violent/inappropriate content
3. LLM generates story text aligned to selected Jolly Phonics level
4. LLM output screened for violent/inappropriate content
5. A "character reference image" is generated once for the story
6. Image generation provider (Imagen 4 default; abstracted) generates one cartoon illustration per sentence using the character reference for consistency
7. Parent receives **Web Push notification** that the story is ready
8. Parent opens the **Review view** (see Feature 2)

**Error handling:**
- If generation fails at any step, credit is refunded automatically
- Parent shown an error message with retry option
- If content screening blocks the output, parent is notified and credit is refunded

### Feature 2: Story Review Gate

- Shown only to the parent; story is **not in the catalog** at this stage
- Parent sees the full story (text + all per-sentence images)
- **Accept** → moves story to catalog, makes it visible to child
- **Reject** → discards the draft, offers one free regeneration of the same submission (no extra credit charge); a second regeneration would require a new credit

### Feature 3: Phonics Word Preview Page

- Accessible after acceptance, before the reading view
- Lists all phonics-level words appearing in the story
- "Start Reading" button proceeds to the story

### Feature 4: Story Reading View

- Story displayed with one illustration per sentence
- Illustrations in consistent cartoon style with character continuity
- "Mark as Completed" button at end of story

### Feature 5: Story Catalog

- Lists all accepted user-created stories
- Includes 3 app-provided example stories (always visible, not deletable, cannot be reported)
- Sort options: last read, alphabetical
- Each story card shows title, creation date, and completion count
- **Report action** on each story card

### Feature 6: Word Bank

- Divided into sections by Jolly Phonics level
- Each word shows encounter count (times seen across completed stories)
- Sort options per section: most completed, least completed, alphabetical
- Only populated from completed stories

### Feature 7: Trophy & Achievement System

**Words Learned Milestones:** 10, 50, 100, 150, 200, 250, 300… (every 50 thereafter)

**Reads Completed Milestones:** 10, 30, 60, 90, 120… (every 30 thereafter)

- Trophy page (Settings) shows earned and locked trophies
- Visual celebration animation on milestone unlock

### Feature 8: Credit System

| Credit Type | Amount | Expiry | Trigger |
|---|---|---|---|
| First-time signup bonus | 2 credits | Never | One-time, on account creation |
| Monthly subscription credit | **4 credits/month** | At next renewal date | Auto-issued on each subscription renewal |
| Purchase pack (small) | 1 credit | Never | Paid; price configurable (TBD) |
| Purchase pack (value) | 4 credits | Never | Paid; price configurable (TBD) |
| Generation failure refund | 1 credit | Never | Automatic on any generation failure |

- Credit balance always visible
- Monthly subscription fee required from month 2 (amount **TBD**); first month free
- Internally, expirable monthly credits should be consumed *before* non-expiring purchased credits

### Feature 9: Combined Monthly Email Notification

- Sent once per month to parent's registered email
- Contains: reads completed that month, unique words learned that month, **trophy status** (earned milestones, next milestone targets)

### Feature 10: Web Push Notifications

- Used to notify the parent when a generated story is ready to review
- Requires parent permission grant on first generation submission
- Fallback if permission denied: parent must check the app manually; we may also surface an in-app "story ready" badge

### Feature 11: App Navigation (Tab Structure — Phase 1 onward)

| Tab | Contents |
|---|---|
| **Create Story** | Story creation form (phonics level chooser, character, theme, plot inputs) |
| **Story Catalog** | Accepted user stories + 3 app example stories; sort controls; report action |
| **Word Bank** | Words grouped by Jolly Phonics level; sort controls per section |
| **Settings** | Trophies, contact us, terms of use, account management, credit purchase |

### Feature 12: Account & Auth

- Parent account registration and login
- Secure session management
- Email required (used for monthly notifications)
- No child profile setup required

### Out of Scope (v2)
- Multiple child profiles per parent account
- PDF export of stories
- Audio / text-to-speech playback
- Social sharing
- Human moderator queue for reported stories (auto-hide + auto-delete only)

---

## Technical Constraints

### Performance
- Story text generation: target < 15 seconds
- Image generation: target < 10 seconds per image; total story generation time noted in PRD as a UX risk mitigated by Web Push (parent can leave the app while generating)
- Page load: < 3 seconds for catalog / word bank views

### Scalability
- **Launch capacity**: backend must comfortably support **100 monthly active users (MAU)** at Phase 0 launch
- **Target capacity**: backend must scale to **10,000 MAU** via **horizontal scaling alone** — adding instances, scaling the database tier, and adjusting infrastructure config — without requiring a major code rewrite, schema overhaul, or architectural refactor
- Design implications (to be detailed in high-level design):
  - **Stateless application servers** (sessions externalised — JWT or a shared session store)
  - **Database choice / topology** must support read replicas, connection pooling, and partitioning at the 10k-MAU level
  - **Async job processing**: long-running story generation must run via a queue / worker pool, not in-request, so web tier can scale independently
  - **Object storage** for generated images (not local disk), CDN-fronted for read traffic
  - **Idempotent credit operations** and transactional credit ledger to remain correct under concurrent load
  - **Observability** (metrics, structured logs, traces) from day one so scale bottlenecks are diagnosable

### Security & Compliance
- **COPPA (US)**: Children's data must not be collected directly; parent account holds all data
- **GDPR-K (EU)**: Parental consent model; data minimisation for child-related content
- Authentication: secure password hashing, HTTPS only, secure session tokens
- No third-party ad tracking
- **Content moderation**: input and output screened for violence, inappropriate content, and celebrity name usage
- **Report flow**: reported stories must be hidden from all viewers immediately and purged from the database within 30 days

### Architecture
- **Separated client and server codebases** from day one — server code is identical across all clients; clients (web, iPhone, iPad, Android) are independently developed, deployed, and versioned, communicating with the server only through a versioned API
- A single backend API serves **web, iPhone, iPad, and Android** clients; no client-specific server logic
- Web-first; mobile-responsive layout for Phase 0
- **Image generation must be abstracted behind a provider-agnostic interface** in the backend; Imagen 4 is the default implementation, but the system must allow swapping providers (e.g. OpenAI image API, Stable Diffusion, others) without changes to higher-level code
- LLM provider may be similarly abstracted (TBD in high-level design)
- Architecture must satisfy the Scalability requirements above (stateless app tier, queue-based generation, externalised storage and sessions) so the 100 → 10,000 MAU growth path requires no rewrite

### Integration
- **LLM API**: story text generation + content screening (provider TBD)
- **Google Imagen 4 (Vertex AI)**: default per-sentence illustration generation with reference image for character consistency
- **Content moderation API** (or LLM-based): input/output safety screening
- **Payment processors**: Stripe for web (Phase 0); Apple IAP and Google Play Billing for native apps later
- **Email service**: monthly notification delivery
- **Web Push service**: story-ready notification (Phase 0 web)

### Configuration
- Credit pack prices must be configurable without code deployment
- Monthly credit allocation (4) must be configurable
- Monthly subscription price must be configurable

---

## MVP Scope & Phasing

### Phase 0: Proof of Concept (Web only)
**Scope:**
- Parent account registration & login
- **2-tab navigation only**: Create Story, Story Catalog
- Story creation (4 inputs, random options, smart defaults)
- Input and output content moderation
- LLM story text generation
- Per-sentence image generation (Imagen 4 behind provider-agnostic interface)
- Parent review-before-catalog gate (Epic 2)
- Web Push notification on generation complete
- Monthly subscription model (Stripe; first month free)
- Monthly credit auto-issue (4 credits/month)

**Phase 0 exit criterion**: The PoC runs in public production for **~1 month with stable operation and no major bugs**. Specific stability bar: no P0/P1 incidents, generation success rate above an agreed threshold (to be set in high-level design), no critical safety/moderation regressions. *(This is qualitative — to be sharpened with concrete metrics during high-level design.)*

### Phase 1: Full Feature Launch (Web)
- Adds Word Bank tab and Settings tab (full 4-tab navigation)
- Word Bank (by phonics level, sortable)
- Trophy & achievement page (in Settings)
- Combined monthly progress + trophy email
- 3 app-provided example stories in catalog
- Credit pack purchase flow (1-pack, 4-pack)
- Report flow on Story Catalog
- Phonics Word Preview Page (Epic 3)

### Phase 2: Native iPhone
- Native iOS (iPhone) app on shared backend
- Apple IAP for subscriptions and credit packs
- iOS native push notifications

### Phase 3: iPad and Android
- iPad layout adaptation
- Android native app
- Google Play Billing

### Future Considerations
- Multiple child profiles per parent account
- PDF export
- Audio playback / text-to-speech
- Teacher/classroom accounts
- Curriculum integration beyond Jolly Phonics
- Human moderator queue for reported content

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation Strategy |
|---|---|---|---|
| Image generation slow (>30s per story) | High | Medium | Web Push removes user-blocking wait; parent can leave the app |
| Character inconsistency across images | Medium | High | Reference image technique; prompt engineering; prototype before Phase 0 launch |
| Imagen 4 deprecated or quota-limited | Medium | High | Provider-agnostic image abstraction in backend; document at least one fallback (e.g. OpenAI image API) before Phase 0 launch |
| COPPA / GDPR-K compliance complexity | Medium | High | Legal review before launch; parent-only account model; data minimisation |
| LLM generates age-inappropriate content | Low | High | Input + output content screening; parent review gate as last line of defense |
| Celebrity name bypass (indirect references) | Medium | Medium | Blocklist + LLM-based name detection; parent review gate catches what automation misses |
| Apple IAP 25–30% fee compresses margin | High | Medium | Phase 0/1 on web/Stripe captures higher-margin users first; iOS pricing modeled around fee |
| Subscription churn after free month | Medium | High | Demonstrate value in month 1 via example stories and smooth onboarding |
| Web Push permission denied → parent misses notifications | Medium | Medium | In-app "story ready" badge as fallback; email fallback considered for Phase 1+ |
| Stale monthly credits frustrate parents | Low | Medium | UI shows expiry date; consume expirable credits before non-expirable purchased credits |
| Scaling past launch capacity requires rewrite | Medium | High | Enforce stateless app tier, queued generation, and externalised image/session storage from Phase 0; load-test the 10k-MAU path before Phase 1 |
| Database becomes scaling bottleneck before 10k MAU | Medium | High | Choose a database with proven horizontal scaling (read replicas, partitioning); include connection pooling and migration tooling in HLD |
| Generation worker queue saturates during traffic spikes | Medium | Medium | Auto-scaling worker pool; observability/alerts on queue depth; communicate ETA to parent if backlog grows |

---

## Dependencies & Blockers

**Dependencies:**
- **Google Vertex AI access**: Imagen 4 API requires GCP project setup and quota approval
- **LLM API selection**: story generation provider must be chosen before backend development begins
- **Image-provider abstraction design**: must be defined before Phase 0 backend implementation
- **Jolly Phonics word lists**: required per level to drive generation prompts and word bank logic
- **Stripe account**: required for Phase 0 web subscription and credit pack billing
- **Apple Developer + Google Play accounts**: required before Phase 2/3
- **Email service**: required for Phase 1 monthly notification
- **Web Push infrastructure (VAPID keys, service worker)**: required for Phase 0
- **Cloud infrastructure choices** must support horizontal scaling from day one: stateless compute (containers / managed app platform), managed database with replica support, object storage + CDN, managed queue/worker service — selection to be made in high-level design

**Known Blockers / TBDs:**
- Monthly subscription price (**TBD** — needed before billing integration begins)
- Credit pack prices for 1-pack and 4-pack (**TBD** — needed before billing integration; configurable)
- Imagen 4 character consistency must be prototyped to validate quality before architecture is finalised
- Phase 0 exit criterion needs concrete metric thresholds (set during high-level design)
- At least one fallback image provider should be identified before Phase 0 launch to validate the abstraction

---

## Appendix

### Glossary
- **Jolly Phonics**: A systematic synthetic phonics programme; defines levels of phonics complexity used to calibrate story vocabulary
- **Phonics Word**: A word in the story that belongs to the selected Jolly Phonics level
- **Story Acceptance**: Parent explicitly approves a generated story; only accepted stories enter the catalog
- **Story Completion**: Parent marks an accepted story as read with the child; triggers word bank update and read count
- **Character Reference Image**: A single illustration generated at story creation, used to maintain character consistency across per-sentence images
- **Credit**: A token allowing one story creation; consumed on submission, refunded on generation failure
- **Monthly Credit**: An auto-issued credit granted on subscription renewal that expires at the next renewal date
- **Purchased Credit**: A credit acquired via a paid pack; does not expire
- **Word Bank**: Aggregated list of all phonics words encountered across completed stories, organised by Jolly Phonics level
- **Web Push**: Browser-native push notification mechanism (Web Push API + VAPID + service worker)
- **MAU (Monthly Active User)**: A parent account that logs into the app at least once within a rolling 30-day window

### References
- Jolly Phonics: https://www.jollylearning.co.uk
- Google Imagen 4 (Vertex AI): https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview
- COPPA compliance: https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions
- Web Push (MDN): https://developer.mozilla.org/en-US/docs/Web/API/Push_API
- High-level design (in progress): `doc/highLevelDesign.md`
- Meeting notes: `doc/MeetingNotes`

---

*This PRD was created through interactive requirements gathering with quality scoring to ensure comprehensive coverage of business, functional, UX, and technical dimensions.*
