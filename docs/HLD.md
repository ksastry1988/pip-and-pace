# High-Level Design: Pip & Pace — Focus & Mindfulness App for Neurodivergent Children

**Status:** Prototype phase
**Last updated:** 2026-09-24

---

## 1. Overview & Goals

### Problem statement
Neurodivergent children (ages 4–10) — including those with ADHD, autism, and other attention or sensory-processing differences — are often underserved by mainstream focus and mindfulness apps, which assume neurotypical pacing, sensory tolerance, and reward structures. This app provides a personalized, low-friction tool that helps these children build focus and self-regulation skills through gamified exercises, structured timers, and mindfulness techniques, adapted to their individual sensory and attentional profile.

The app is named **Pip & Pace**: "Pip" is the on-screen companion character who guides the child through the experience (reducing reliance on text-heavy instructions), and "Pace" refers to the visual timer/session element that reflects the child's personalized pacing.

### Target users
- **Primary users:** Children ages 4–10, split into two age-band tiers:
  - **Early Explorers** (4–6)
  - **Builders** (7–10)
- **Secondary users:** Parents/guardians who set up the child's account and configure the onboarding questionnaire (given COPPA constraints on data collected directly from children under 13).

### Success criteria
- A child can complete an onboarding flow, receive a personalized content-weighting profile, and immediately engage with at least one adapted focus exercise.
- The experience visibly adapts to the child's profile (pacing, sensory intensity, exercise selection) rather than presenting one-size-fits-all content.
- No point in the flow implies a clinical diagnosis, score, or assessment result.
- The prototype demonstrates the full onboarding → profile → adaptive exercise → session loop end-to-end for at least one exercise type.

### Non-goals
- **No clinical assessment or diagnosis.** The app does not claim to diagnose, screen for, or clinically evaluate ADHD, autism, or any condition.
- **No numeric scoring** of the child's abilities or challenges.
- **No production-scale infrastructure** in this phase — the backend is built to prove the architecture, not to handle production load.
- **No full feature set** — only 2–3 vertical slices are implemented end-to-end; breadth is deferred.

---

## 2. Functional Requirements

### 2.1 Onboarding

| ID | Requirement |
|----|-------------|
| FR-1.1 | The system shall present an onboarding questionnaire to the parent/guardian before a child profile is created. |
| FR-1.2 | The questionnaire shall cover 8 focus-challenge categories: sustained attention, task initiation, transitions, distractibility, impulse control, multi-step instruction following, physical stillness, and environmental sensitivity. |
| FR-1.3 | Each questionnaire response shall map to a weighting on one or more app features (not a numeric score on the child). |
| FR-1.4 | The system shall generate a content-weighting profile from questionnaire responses and persist it against the child's account. |
| FR-1.5 | The onboarding flow shall allow the parent to skip or revisit individual questions without losing prior answers. |
| FR-1.6 | The system shall allow the profile to be edited/updated after initial onboarding. |

### 2.2 Profile & Personalization

| ID | Requirement |
|----|-------------|
| FR-2.1 | The system shall use the content-weighting profile to select and adapt exercises presented to the child. |
| FR-2.2 | The system shall adjust sensory intensity (sound, motion, color) of exercises based on the profile's environmental-sensitivity weighting. |
| FR-2.3 | The system shall adjust exercise pacing (step duration, transition speed) based on the profile's relevant weightings (transitions, sustained attention). |
| FR-2.4 | The profile shall never be displayed to the child or parent as a numeric score or diagnostic label. |

### 2.3 Exercises

| ID | Requirement |
|----|-------------|
| FR-3.1 | The system shall present at least one gamified focus exercise, age-band-appropriate (Early Explorers vs. Builders content). |
| FR-3.2 | Exercises shall support pausing and resuming without penalty or loss of progress. |
| FR-3.3 | Exercises shall have no "failure" state — only completion or early exit. |
| FR-3.4 | Exercise difficulty/pacing parameters shall be driven by the child's content-weighting profile. |

### 2.4 Session & Timer

| ID | Requirement |
|----|-------------|
| FR-4.1 | The system shall provide a visual (non-purely-numeric) representation of elapsed/remaining time during a session. |
| FR-4.2 | The system shall persist session state so a session can be resumed after interruption. |
| FR-4.3 | Transitions between session states (e.g., exercise → break) shall be previewed to the child before they occur (no abrupt changes). |

### 2.5 Age-Tier Logic

| ID | Requirement |
|----|-------------|
| FR-5.1 | The system shall determine the child's age tier (Early Explorers 4–6, Builders 7–10) at account creation. |
| FR-5.2 | Content, UI complexity, and instruction phrasing shall differ by age tier. |

---

## 3. Non-Functional Requirements

### Performance
- NFR-1.1: Exercise screens shall render interactive within 2 seconds on a mid-range device/connection.
- NFR-1.2: Session state saves shall complete without blocking the UI (async persistence).

### Accessibility
- NFR-2.1: The app shall meet WCAG 2.1 AA for color contrast, text sizing, and keyboard/switch-device navigation.
- NFR-2.2: All sensory intensity settings (sound, motion, color/flashing) shall be independently adjustable, including a "reduced motion / reduced stimulation" global mode.
- NFR-2.3: Instructions shall be presented in both text and audio/icon form to reduce reliance on reading ability alone.
- NFR-2.4: No content shall rely on time pressure as the sole interaction mechanism.

### Privacy & COPPA Compliance
- NFR-3.1: The system shall collect no personal information directly from children; account setup and questionnaire responses are provided by a parent/guardian.
- NFR-3.2: The backend shall minimize data collection to what is functionally necessary (profile weightings, session progress) and avoid collecting identifiable data beyond what's needed for account function.
- NFR-3.3: Parental consent shall be captured and stored before any child data is persisted.
- NFR-3.4: Data at rest and in transit shall be encrypted.
- NFR-3.5: The system shall provide a mechanism for parents to view, export, or delete their child's data.

### Offline Support
- NFR-4.1: In-progress exercise/session state shall be cached client-side so a brief connectivity loss does not interrupt an active session.
- NFR-4.2: Full offline mode is out of scope for the prototype; this is a resilience requirement, not a feature.

### Scalability
- NFR-5.1: The backend architecture shall separate profile logic, exercise content, and session state into distinct services/modules so new exercise types or age tiers can be added without restructuring core logic.
- NFR-5.2: The data model shall not hard-code the 8 focus-challenge categories in a way that prevents adding or adjusting categories later.

---

## 4. High-Level Design

### 4.1 System Architecture

```
┌─────────────────────┐        ┌──────────────────────┐        ┌──────────────────┐
│   Web Frontend       │  HTTPS │   Backend API         │        │   Database        │
│   (React SPA)         │◄──────►│   (REST/GraphQL)      │◄──────►│   (PostgreSQL)     │
│                       │        │                        │        │                    │
│ - Onboarding UI       │        │ - Onboarding service   │        │ - accounts         │
│ - Exercise UI         │        │ - Profile engine        │        │ - profiles         │
│ - Session/Timer UI    │        │ - Exercise engine        │        │ - sessions         │
│ - Parent dashboard    │        │ - Session service         │        │ - exercise_content │
└─────────────────────┘        └──────────────────────┘        └──────────────────┘
```

- **Frontend:** Single-page web app, responsible for rendering age-tier-appropriate UI, applying sensory settings client-side, and managing local caching of active session state.
- **Backend API:** Stateless service layer exposing endpoints for onboarding submission, profile retrieval, exercise content delivery, and session persistence.
- **Database:** Relational store for accounts, profiles, session state, and exercise content metadata.

### 4.2 Component Breakdown

| Component | Responsibility |
|-----------|----------------|
| **Onboarding Module** (frontend) | Renders questionnaire, captures parent responses, submits to backend. |
| **Profile Engine** (backend) | Converts questionnaire responses into a content-weighting profile; exposes profile to other services. |
| **Exercise Engine** (backend) | Selects and parameterizes exercises based on a child's profile and age tier. |
| **Timer/Session Module — "Pace"** (frontend + backend) | Manages visual timer state client-side; persists session progress via the Session Service. Surfaced to the child as "Pace," a visual (non-numeric) pacing element. |
| **Companion/Guide Module — "Pip"** (frontend) | Renders the on-screen companion character used to deliver instructions and transition previews (per FR-4.3), reducing reliance on text. |
| **API Layer** (backend) | Auth, request validation, routing between frontend and services; enforces parental-consent gating. |

### 4.3 Data Model (initial)

```
Account
  - id
  - parent_email
  - consent_given_at
  - created_at

ChildProfile
  - id
  - account_id (FK)
  - age_tier (enum: early_explorer, builder)
  - content_weights (JSON: { category: weight, ... } for the 8 focus-challenge categories)
  - created_at
  - updated_at

Session
  - id
  - child_profile_id (FK)
  - exercise_id (FK)
  - state (enum: in_progress, paused, completed, exited_early)
  - elapsed_seconds
  - started_at
  - updated_at

ExerciseContent
  - id
  - age_tier
  - base_duration_seconds
  - sensory_defaults (JSON: { sound, motion, color })
  - pacing_parameters (JSON)
```

`content_weights` is stored as an open JSON map rather than fixed columns, so categories can be added or adjusted without a schema migration (supports NFR-5.2).

### 4.4 Tech Stack Recommendation

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Frontend | React + TypeScript | Fast to scaffold, strong accessibility tooling (ARIA support, testing libraries), team familiarity assumed. |
| Backend | Node.js (Express or Fastify) + TypeScript | Shared language with frontend reduces context-switching for a solo/small-team prototype; fast to iterate. |
| Database | PostgreSQL | JSON column support (for `content_weights`) plus relational integrity for accounts/sessions; easy local dev setup. |
| Hosting (prototype) | Single deployable (e.g., Render, Fly.io, or Vercel + managed Postgres) | Minimizes DevOps overhead; not meant to reflect production topology. |

**Prototype vs. production trade-off:** This stack favors development speed and clarity over scalability. A production version would likely split the Profile Engine and Exercise Engine into separately deployable services, add a caching layer for exercise content, and introduce a proper auth provider (e.g., managed identity service) instead of a lightweight custom auth flow. Those are deferred as out of scope for the prototype (see NFR-5.1 for how the current design keeps that split easy to do later).

### 4.5 Key User Flows

**Flow 1: First-time onboarding**
1. Parent creates account, provides consent.
2. Parent completes onboarding questionnaire (8 categories).
3. Backend Profile Engine generates content-weighting profile.
4. Child profile created; parent is shown a plain-language summary (not a score) of the personalization applied.

**Flow 2: Running an exercise**
1. Child (or parent, for younger tier) selects an activity from the home screen.
2. Frontend requests exercise content from Exercise Engine, passing profile ID.
3. Backend returns exercise parameterized by profile (pacing, sensory defaults).
4. Frontend renders exercise; session record created with state `in_progress`.
5. On completion or early exit, session state is updated accordingly.

**Flow 3: Resuming a session**
1. Child returns to the app with an existing `paused` or `in_progress` session.
2. Frontend checks for cached local session state; falls back to backend session record if not present.
3. Exercise resumes from stored `elapsed_seconds`, with a preview transition (per FR-4.3) rather than an abrupt jump back in.

---

## 5. Open Questions / Risks

- **Authentication approach:** Is a custom email/password flow acceptable for the prototype, or should a managed auth provider be used even at this stage (affects COPPA consent-capture design)?
- **Exercise content authoring:** Will exercise content (instructions, assets) be authored manually for the prototype, or does content need a lightweight CMS even now?
- **Parent vs. child interaction model:** For Early Explorers (4–6), is the parent expected to operate the app alongside the child, or does the child interact independently? This affects UI complexity assumptions.
- **Sensory default calibration:** The initial `sensory_defaults` per exercise are placeholders — real values need input from someone with occupational-therapy or neurodivergent-UX expertise to avoid guessing at appropriate defaults.
- **Data retention policy:** No specific retention/deletion timeline is defined yet beyond "parent can request deletion" (NFR-3.5) — a concrete policy is needed before any real user data is collected.
- **Risk:** Without clinical or OT input, feature adaptations (pacing, sensory levels) may be well-intentioned but miscalibrated; consider a review step with a subject-matter expert before broader testing.
