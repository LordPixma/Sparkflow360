# Sparkflow360 — High-Level Technical Design Document

**Version:** 1.0 | **Date:** 2026-02-11 | **Status:** DRAFT

---

## Context

Sparkflow360 is a career planning SaaS web application inspired by the "Career Masterclass" workbook. The workbook provides structured frameworks (Vision, SKEA, SWOT, Circle of Concern/Influence, More/Less/Start/Stop, Action Planning) that help professionals at any level build and enhance their careers. The goal is to digitize these frameworks into an interactive, AI-enhanced web tool that is fully hosted within the Cloudflare ecosystem, following secure-by-design principles.

**Key constraint:** 100% Cloudflare-native hosting — no AWS, GCP, Azure, or self-hosted infrastructure.

---

## A. System Architecture Overview

```
[User Browser]
      │
      ▼
[Cloudflare CDN / Edge Network]
      │
      ├── [Cloudflare WAF + Rate Limiting + Bot Management]
      ├── [Cloudflare Turnstile] (bot verification on forms)
      ├── [Cloudflare Access / Zero Trust] (admin panel + Enterprise SSO)
      │
      ▼
[Cloudflare Workers — Static Assets]  →  React SPA (Vite build)
      │
      ▼
[Cloudflare Workers — API Layer]  →  Hono Framework (TypeScript)
      │
      ├── [D1 Database]           Primary relational storage (SQLite)
      ├── [KV Namespace]          Sessions, feature flags, AI cache
      ├── [R2 Bucket]             File uploads, PDF exports, avatars
      ├── [Durable Objects]       Per-user rate limiting, real-time state
      ├── [Queues]                Async jobs: email, PDF generation, AI tasks
      ├── [Workers AI]            LLM inference for career insights
      └── [Email Workers]         Transactional email (onboarding, reminders)
```

### Cloudflare Service Mapping

| Cloudflare Service | Application Purpose |
|---|---|
| **Workers** | API backend (Hono), request routing, business logic, middleware |
| **Workers Static Assets** | React SPA hosting, CDN distribution |
| **D1** | Primary database — users, SKEA, SWOT, goals, action plans |
| **KV** | Session store, feature flags, cached AI responses, rate limits |
| **R2** | Avatars, vision board images, exported PDFs, case study assets |
| **Durable Objects** | Per-user precise rate limiting, WebSocket connections |
| **Queues** | Async processing: AI batching, email dispatch, PDF generation |
| **Workers AI** | Career insight generation, SWOT suggestions, goal recommendations |
| **Access / Zero Trust** | Admin panel auth, Enterprise SSO (SAML/OIDC) |
| **Turnstile** | Bot protection on signup, login, password reset forms |
| **WAF** | Application-layer firewall, OWASP rule sets |
| **Email Workers** | Transactional emails: onboarding, review reminders, weekly digests |
| **Analytics Engine** | Custom event tracking, usage metering for billing |
| **Cache API** | Edge caching for public API responses (case studies) |

### Data Flow

1. User request hits Cloudflare edge → WAF and rate limiting applied
2. Static asset requests (HTML, JS, CSS) served from Workers Static Assets — no API invocation
3. API requests (`/api/*`) route to Hono-based Worker → JWT validated from KV → business logic → D1/R2/KV
4. AI requests enqueued via Queues (non-blocking) or called synchronously (low-latency features)
5. Background jobs (email, PDF export) processed asynchronously via Queue consumers

---

## B. Technology Stack

### Language: TypeScript

TypeScript is the native language for Cloudflare Workers. It provides type safety across the full stack (frontend + backend), has first-class Cloudflare tooling support, and the V8 isolate model in Workers gives sub-5ms cold start times.

### Frontend

| Technology | Version | Purpose |
|---|---|---|
| React | 19.x | UI framework |
| TypeScript | 5.7.x | Type-safe language |
| Vite | 6.1.x | Build tool and dev server |
| @cloudflare/vite-plugin | latest | Workers runtime integration in dev |
| TanStack Router | 1.x | Fully type-safe file-based routing |
| TanStack Query | 5.x | Server state management, caching, mutations |
| Zustand | 5.x | Client-side state management |
| Tailwind CSS | 4.x | Utility-first CSS framework |
| Zod | 4.x | Runtime schema validation + type inference |
| Recharts | 2.x | Charts for SWOT visualizations, progress tracking |
| @dnd-kit/core | 6.x | Drag-and-drop for Circle of Concern/Influence |
| react-hook-form | 7.x | Form management with Zod resolver |

### Backend / API Layer

| Technology | Version | Purpose |
|---|---|---|
| Hono | 4.x | Edge-native web framework for Workers |
| Drizzle ORM | 0.45.x | Type-safe ORM with first-class D1 support |
| drizzle-kit | 0.31.x | Schema migrations and introspection |
| Zod | 4.x | API request/response validation (shared with frontend) |
| jose | 6.x | JWT creation/verification (Web Crypto API, no Node deps) |
| nanoid | 5.x | Collision-resistant ID generation |

### Development Tooling

| Technology | Version | Purpose |
|---|---|---|
| Wrangler | 4.x | Cloudflare CLI for deploy, dev, D1 management |
| Vitest | 3.x | Unit and integration testing |
| @cloudflare/vitest-pool-workers | latest | Run tests inside Workers runtime |
| Playwright | 1.x | End-to-end testing |
| Biome | 1.9.x | Linter + formatter (replaces ESLint + Prettier) |
| GitHub Actions | N/A | CI/CD pipeline |

### Key Architectural Decisions

- **Hono over raw Workers fetch handler** — Complete middleware ecosystem (CORS, security headers, JWT), type-safe routes, clean `c.env` pattern for Cloudflare bindings. Most actively maintained Workers-native framework.
- **Drizzle ORM over Prisma** — First-class D1 support, zero runtime overhead, supports D1 batch API, full TypeScript inference from schema. Prisma's Workers support adds significant bundle size.
- **TanStack Router over React Router** — Fully type-safe route params, search params, and loaders, eliminating runtime navigation errors.
- **Custom JWT auth over Cloudflare Access for end users** — Access is designed for workforce identity, not consumer SaaS. Custom JWT with `HttpOnly` cookies provides standard consumer auth. Access reserved for admin panel and Enterprise SSO.
- **Single D1 database with row-level isolation** — Simpler migrations, query patterns, and cross-user analytics than per-tenant databases. Can migrate to per-tenant D1 for Enterprise if needed.

---

## C. Application Modules

Each workbook framework becomes an application module:

### Module 1: User Dashboard & Onboarding

**Purpose:** Central hub showing career planning progress across all modules.

**Key Screens:**
- Welcome / onboarding wizard (3-step guided intro)
- Dashboard with progress cards per framework module
- Progress timeline visualization
- Quick-start prompts

**API Endpoints:**
- `POST /api/auth/register` — Register (Turnstile-protected)
- `POST /api/auth/login` — Login (Turnstile-protected)
- `POST /api/auth/logout` — Invalidate session
- `GET /api/auth/me` — Current user profile
- `PATCH /api/users/:id` — Update profile
- `GET /api/dashboard/progress` — Aggregated progress

### Module 2: Vision Builder

**Purpose:** Guided experience for 2-, 5-, and 10-year career visions using Think → Explore → Decide → Act lifecycle.

**Key Screens:**
- Vision creation wizard ("What? Who? Why? Where?" prompts)
- Vision timeline (2 / 5 / 10 year horizons)
- Vision board (rich text + optional images via R2)
- Lifecycle phase tracker
- Freeform "Write your vision down" editor (workbook p.12)

**API Endpoints:**
- `GET/POST /api/visions` — List / create visions
- `GET/PATCH/DELETE /api/visions/:id` — CRUD
- `PATCH /api/visions/:id/phase` — Advance lifecycle phase
- `POST /api/visions/:id/ai-refine` — AI-assisted vision refinement

### Module 3: SKEA Model (Skills, Knowledge, Experience, Attributes)

**Purpose:** Structured inventory forming the foundation for SWOT analysis.

**Key Screens:**
- SKEA entry form with category tabs
- Guided prompts per category (workbook pp.16-17)
- Competency tagging (Observable, Measurable, Transferable, Performance-based)
- Summary dashboard with completeness score

**API Endpoints:**
- `GET/POST /api/skea` — List / create entries
- `GET/PATCH/DELETE /api/skea/:id` — CRUD
- `GET /api/skea/summary` — Category counts and completeness
- `POST /api/skea/ai-suggest` — AI-suggest entries based on job title/industry

### Module 4: SWOT Analysis

**Purpose:** Interactive personal SWOT with guided questions and SKEA cross-referencing.

**Key Screens:**
- Quick warm-up: "biggest strength?", "worst threat?" (workbook pp.13-14)
- Interactive 2x2 SWOT matrix
- Per-quadrant detail views with guided questions (workbook pp.18-24)
- SKEA cross-reference panel (link SWOT entries to SKEA items)
- Snapshot/summary view (workbook p.24 layout)

**API Endpoints:**
- `GET/POST /api/swot` — List / create analyses
- `GET/PATCH/DELETE /api/swot/:id` — CRUD with all entries
- `POST /api/swot/:id/entries` — Add entry to quadrant
- `POST /api/swot/:id/entries/:eid/link-skea` — Link to SKEA entry
- `POST /api/swot/:id/ai-analyze` — AI-powered suggestions
- `GET /api/swot/:id/matrix` — 2x2 matrix view data

### Module 5: Circle of Concern / Circle of Influence

**Purpose:** Visual drag-and-drop tool mapping SWOT items into what users control vs. cannot control (Covey framework, workbook pp.25-27).

**Key Screens:**
- Dual concentric circle visualization (drag-and-drop via @dnd-kit)
- Auto-populate from SWOT (S/W → Influence, O/T → Concern)
- Proactive vs. reactive mindset assessment
- Resource allocation view (Time, Energy, Money focus)

**API Endpoints:**
- `GET/POST /api/circles` — List / add entries
- `PATCH/DELETE /api/circles/:id` — Move between circles, reposition
- `POST /api/circles/populate-from-swot/:swotId` — Auto-populate
- `GET/POST /api/circles/assessment` — Mindset assessment

### Module 6: Decision Making Tools

**Purpose:** Two frameworks from the workbook — More/Less/Start/Stop (Option 1, pp.31-33) and SWOT 2x2 strategic matrix (Option 2, p.34).

**Key Screens:**
- **Option 1:** M/L/S/S interactive grid per SWOT quadrant with visual circle diagram
- **Option 2:** SWOT 2x2 strategic matrix:
  - S+O: Natural priorities (Charis case study)
  - W+O: Attractive but challenging changes (Tom case study)
  - S+T: Easy to defend/counter (Carlos case study)
  - W+T: High-risk areas (Kim case study)
- Case study reference panel per combination

**API Endpoints:**
- `GET/POST/PATCH/DELETE /api/decisions/mlss` — More/Less/Start/Stop CRUD
- `GET/POST/PATCH/DELETE /api/decisions/matrix` — Strategic matrix CRUD
- `POST /api/decisions/ai-recommend` — AI decision recommendations

### Module 7: Action Planning (SMART Goals)

**Purpose:** Convert decisions into structured, time-bound action plans (workbook pp.41-43).

**Key Screens:**
- SMART goal wizard (Specific, Measurable, Attainable, Realistic, Time-based)
- Action Plan Template table (Goals → Objectives → Actions → Constraints → Resources → Target Dates)
- Review cycle tracker (Review → Focus → Be Accountable → Reward)
- Goal progress tracking with milestones
- Kanban-style board (To Do / In Progress / Done)

**API Endpoints:**
- `GET/POST /api/plans` — List / create plans
- `GET/PATCH/DELETE /api/plans/:id` — CRUD
- `POST /api/plans/:id/goals` — Create SMART goal
- `POST /api/plans/:id/goals/:gid/actions` — Add action step
- `POST /api/plans/:id/goals/:gid/reviews` — Add review entry
- `POST /api/plans/:id/goals/:gid/ai-smart` — AI-validate SMART criteria

### Module 8: Case Studies & Learning

**Purpose:** Real-world examples (Phil Libin, Michael Hyatt, Kim, Tom, Carlos, Charis) with reflection exercises.

**Key Screens:**
- Case study library (card grid)
- Detail view with framework connections
- "Relate to your situation" reflection form
- User-submitted case studies (Pro/Enterprise)

**API Endpoints:**
- `GET /api/case-studies` — List case studies
- `GET /api/case-studies/:id` — Detail with framework tags
- `POST/GET/PATCH /api/case-studies/:id/reflections` — User reflections

### Module 9: Personal Brand / USP

**Purpose:** Help users define their unique selling point and personal brand (workbook p.7 employee branding concept).

**Key Screens:**
- USP builder wizard
- Personal brand statement editor
- Brand-SKEA alignment view
- AI-assisted elevator pitch generator

**API Endpoints:**
- `GET/POST/PATCH /api/brand` — Personal brand CRUD
- `POST /api/brand/ai-pitch` — AI-generate elevator pitch
- `POST /api/brand/ai-usp` — AI-suggest USP from SKEA

---

## D. Database Schema Design

### D1 (SQLite) — Core Tables

**ID Strategy:** All PKs use `nanoid` (21-char URL-safe strings) — prevents sequential enumeration attacks.
**Timestamps:** ISO 8601 TEXT strings (D1/SQLite has no native DATETIME).
**JSON columns:** TEXT with Zod validation at application layer.

```sql
-- Users & Auth
CREATE TABLE User (
  id                   TEXT PRIMARY KEY,
  email                TEXT UNIQUE NOT NULL,
  name                 TEXT NOT NULL,
  password_hash        TEXT NOT NULL,
  password_salt        TEXT NOT NULL,
  avatar_url           TEXT,
  subscription_tier    TEXT NOT NULL DEFAULT 'free',  -- 'free' | 'pro' | 'enterprise'
  onboarding_completed INTEGER NOT NULL DEFAULT 0,
  created_at           TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at           TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE UserSession (
  id          TEXT PRIMARY KEY,
  user_id     TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  token_hash  TEXT NOT NULL,
  expires_at  TEXT NOT NULL,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Vision Builder
CREATE TABLE Vision (
  id                TEXT PRIMARY KEY,
  user_id           TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  horizon           TEXT NOT NULL,  -- '2_year' | '5_year' | '10_year'
  what              TEXT,
  who               TEXT,
  why               TEXT,
  where_location    TEXT,
  vision_statement  TEXT,
  lifecycle_phase   TEXT NOT NULL DEFAULT 'think',  -- 'think' | 'explore' | 'decide' | 'act'
  inspiration_notes TEXT,
  motivation_notes  TEXT,
  created_at        TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at        TEXT NOT NULL DEFAULT (datetime('now'))
);

-- SKEA (Skills, Knowledge, Experience, Attributes)
CREATE TABLE SkeaEntry (
  id                  TEXT PRIMARY KEY,
  user_id             TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  category            TEXT NOT NULL,  -- 'skill' | 'knowledge' | 'experience' | 'attribute'
  title               TEXT NOT NULL,
  description         TEXT,
  proficiency_level   INTEGER,  -- 1-5
  is_transferable     INTEGER NOT NULL DEFAULT 0,
  industry_sector     TEXT,
  years_of_experience INTEGER,
  tags                TEXT,  -- JSON array of competency tags
  created_at          TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at          TEXT NOT NULL DEFAULT (datetime('now'))
);

-- SWOT Analysis
CREATE TABLE SwotAnalysis (
  id         TEXT PRIMARY KEY,
  user_id    TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  title      TEXT NOT NULL,
  is_active  INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE SwotEntry (
  id               TEXT PRIMARY KEY,
  swot_analysis_id TEXT NOT NULL REFERENCES SwotAnalysis(id) ON DELETE CASCADE,
  user_id          TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  quadrant         TEXT NOT NULL,  -- 'strength' | 'weakness' | 'opportunity' | 'threat'
  content          TEXT NOT NULL,
  priority         INTEGER NOT NULL DEFAULT 0,
  sort_order       INTEGER NOT NULL DEFAULT 0,
  created_at       TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at       TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE SwotSkeaLink (
  id            TEXT PRIMARY KEY,
  swot_entry_id TEXT NOT NULL REFERENCES SwotEntry(id) ON DELETE CASCADE,
  skea_entry_id TEXT NOT NULL REFERENCES SkeaEntry(id) ON DELETE CASCADE,
  created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Circle of Concern / Circle of Influence
CREATE TABLE CircleEntry (
  id             TEXT PRIMARY KEY,
  user_id        TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  swot_entry_id  TEXT REFERENCES SwotEntry(id) ON DELETE SET NULL,
  circle_type    TEXT NOT NULL,  -- 'influence' | 'concern'
  content        TEXT NOT NULL,
  resource_focus TEXT,           -- JSON: {"time": bool, "energy": bool, "money": bool}
  mindset_type   TEXT,           -- 'proactive' | 'reactive'
  position_x     REAL,
  position_y     REAL,
  created_at     TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at     TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE MindsetAssessment (
  id              TEXT PRIMARY KEY,
  user_id         TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  proactive_score INTEGER NOT NULL,
  reactive_score  INTEGER NOT NULL,
  assessment_data TEXT,  -- JSON
  created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Decision Making
CREATE TABLE DecisionMLSS (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  swot_entry_id TEXT NOT NULL REFERENCES SwotEntry(id) ON DELETE CASCADE,
  action        TEXT NOT NULL,  -- 'more' | 'less' | 'start' | 'stop'
  description   TEXT NOT NULL,
  rationale     TEXT,
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE DecisionMatrix (
  id               TEXT PRIMARY KEY,
  user_id          TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  swot_analysis_id TEXT NOT NULL REFERENCES SwotAnalysis(id) ON DELETE CASCADE,
  combination      TEXT NOT NULL,  -- 'so' | 'wo' | 'st' | 'wt'
  insight          TEXT NOT NULL,
  question_response TEXT,
  priority_level   TEXT,  -- 'high' | 'medium' | 'low'
  created_at       TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at       TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Action Planning
CREATE TABLE ActionPlan (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  title         TEXT NOT NULL,
  career_vision TEXT,
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE SmartGoal (
  id               TEXT PRIMARY KEY,
  action_plan_id   TEXT NOT NULL REFERENCES ActionPlan(id) ON DELETE CASCADE,
  user_id          TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  goal_type        TEXT NOT NULL,  -- 'long_term' | 'short_term'
  title            TEXT NOT NULL,
  specific         TEXT,
  measurable       TEXT,
  attainable       TEXT,
  realistic        TEXT,
  time_based       TEXT,
  target_date      TEXT,
  status           TEXT NOT NULL DEFAULT 'not_started',  -- 'not_started' | 'in_progress' | 'completed' | 'paused'
  progress_percent INTEGER NOT NULL DEFAULT 0,
  sort_order       INTEGER NOT NULL DEFAULT 0,
  created_at       TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at       TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE GoalAction (
  id            TEXT PRIMARY KEY,
  smart_goal_id TEXT NOT NULL REFERENCES SmartGoal(id) ON DELETE CASCADE,
  user_id       TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  objective     TEXT NOT NULL,
  action        TEXT NOT NULL,
  constraints   TEXT,
  resources     TEXT,
  target_date   TEXT,
  status        TEXT NOT NULL DEFAULT 'pending',  -- 'pending' | 'in_progress' | 'done'
  sort_order    INTEGER NOT NULL DEFAULT 0,
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE GoalReview (
  id            TEXT PRIMARY KEY,
  smart_goal_id TEXT NOT NULL REFERENCES SmartGoal(id) ON DELETE CASCADE,
  user_id       TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  review_type   TEXT NOT NULL,  -- 'review' | 'focus' | 'accountability' | 'reward'
  notes         TEXT,
  created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Case Studies
CREATE TABLE CaseStudy (
  id             TEXT PRIMARY KEY,
  title          TEXT NOT NULL,
  person_name    TEXT NOT NULL,
  summary        TEXT NOT NULL,
  full_content   TEXT NOT NULL,
  framework_tags TEXT,  -- JSON array: ['swot', 'mlss', 'so', 'wt', etc.]
  key_lessons    TEXT,  -- JSON array
  is_system      INTEGER NOT NULL DEFAULT 1,
  created_by     TEXT REFERENCES User(id) ON DELETE SET NULL,
  created_at     TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE CaseStudyReflection (
  id              TEXT PRIMARY KEY,
  case_study_id   TEXT NOT NULL REFERENCES CaseStudy(id) ON DELETE CASCADE,
  user_id         TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  reflection_text TEXT NOT NULL,
  related_swot_id TEXT REFERENCES SwotAnalysis(id) ON DELETE SET NULL,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Personal Brand
CREATE TABLE PersonalBrand (
  id              TEXT PRIMARY KEY,
  user_id         TEXT UNIQUE NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  usp_statement   TEXT,
  brand_values    TEXT,  -- JSON array
  elevator_pitch  TEXT,
  target_audience TEXT,
  differentiators TEXT,  -- JSON array
  brand_story     TEXT,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Usage Tracking
CREATE TABLE UsageCounter (
  id      TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES User(id) ON DELETE CASCADE,
  feature TEXT NOT NULL,
  period  TEXT NOT NULL,  -- '2026-02' (monthly granularity)
  count   INTEGER NOT NULL DEFAULT 0,
  UNIQUE(user_id, feature, period)
);
```

### Key Indexes

```sql
CREATE INDEX idx_user_email ON User(email);
CREATE INDEX idx_session_user ON UserSession(user_id);
CREATE INDEX idx_session_token ON UserSession(token_hash);
CREATE INDEX idx_session_expires ON UserSession(expires_at);
CREATE INDEX idx_vision_user ON Vision(user_id);
CREATE INDEX idx_skea_user_category ON SkeaEntry(user_id, category);
CREATE INDEX idx_swot_user ON SwotAnalysis(user_id);
CREATE INDEX idx_swot_entry_analysis ON SwotEntry(swot_analysis_id, quadrant);
CREATE INDEX idx_swot_entry_user ON SwotEntry(user_id);
CREATE INDEX idx_circle_user ON CircleEntry(user_id, circle_type);
CREATE INDEX idx_decision_mlss_user ON DecisionMLSS(user_id);
CREATE INDEX idx_decision_matrix_swot ON DecisionMatrix(swot_analysis_id);
CREATE INDEX idx_plan_user ON ActionPlan(user_id);
CREATE INDEX idx_goal_plan ON SmartGoal(action_plan_id);
CREATE INDEX idx_goal_user ON SmartGoal(user_id);
CREATE INDEX idx_action_goal ON GoalAction(smart_goal_id);
CREATE INDEX idx_review_goal ON GoalReview(smart_goal_id);
CREATE INDEX idx_reflection_case ON CaseStudyReflection(case_study_id);
CREATE INDEX idx_reflection_user ON CaseStudyReflection(user_id);
CREATE INDEX idx_usage_user_feature ON UsageCounter(user_id, feature, period);
```

### KV Namespace Usage

| Namespace | Key Pattern | TTL | Purpose |
|---|---|---|---|
| `SF360_SESSIONS` | `session:{token_hash}` | 24h | Edge session validation |
| `SF360_FEATURE_FLAGS` | `flags:{tier}` | 1h | Feature gating per tier |
| `SF360_AI_CACHE` | `ai:{hash(prompt+userId)}` | 24h | Cache AI inference results |
| `SF360_RATE_LIMITS` | `rate:{userId}:{endpoint}` | 60s | Rate limit counters |

### R2 Bucket Strategy

| Bucket | Path Pattern | Access |
|---|---|---|
| `sf360-uploads` | `{userId}/avatars/{filename}` | Private (signed URLs) |
| `sf360-uploads` | `{userId}/vision-boards/{filename}` | Private (signed URLs) |
| `sf360-exports` | `{userId}/exports/{timestamp}.pdf` | Private (signed URLs, 7-day expiry) |
| `sf360-assets` | `case-studies/{studyId}/{filename}` | Public (CDN-cached) |

---

## E. Security Architecture

### Authentication Flow

**Consumer users:** Custom JWT-based auth with `HttpOnly`, `Secure`, `SameSite=Strict` cookies.
**Admin panel:** Cloudflare Access / Zero Trust.
**Enterprise SSO:** Cloudflare Access with customer IdP (Okta, Azure AD, Google Workspace).

**Registration:**
1. Turnstile token validated server-side via siteverify API
2. Password hashed with `scrypt` (N=32768, r=8, p=1) + unique 16-byte salt via Web Crypto API
3. User created in D1, welcome email dispatched via Queue
4. JWT created (HMAC-SHA256), session stored in KV, cookie set

**Session Validation (every API request):**
1. Hono middleware extracts JWT from `HttpOnly` cookie
2. Signature verified via `jose` library
3. Session existence/expiry checked in KV
4. User ID + tier attached to request context

### Authorization (RBAC)

| Feature | Free | Pro | Enterprise |
|---|---|---|---|
| All modules (limited quotas) | Yes | — | — |
| All modules (unlimited) | — | Yes | Yes |
| AI suggestions | 5/month | 200/month | Unlimited |
| PDF export | No | Yes | Yes |
| Personal Brand module | No | Yes | Yes |
| Team workspaces | No | No | Yes |
| Custom SSO | No | No | Yes |

Enforced via Hono middleware checking `user.subscription_tier` against feature permission map in KV.

### API Security

- **Rate Limiting:** 100 req/min per IP (WAF), 300 req/min per user (KV), 10 req/min per user on AI endpoints (Durable Object), 5 attempts/min on auth endpoints
- **Input Validation:** Every endpoint validated with Zod schemas
- **SQL Injection Prevention:** Drizzle ORM parameterized queries exclusively
- **CORS:** Restricted to `sparkflow360.com` origins, credentials enabled
- **Content sanitization:** HTML entities escaped on all user-generated content

### Security Headers (Hono middleware)

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';
  img-src 'self' data: https://*.r2.dev; connect-src 'self' https://challenges.cloudflare.com;
  frame-src https://challenges.cloudflare.com; object-src 'none'; base-uri 'self'; form-action 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

### OWASP Top 10 Mitigations

| Threat | Mitigation |
|---|---|
| A01 Broken Access Control | RBAC middleware, per-resource ownership checks, `user_id` WHERE clauses |
| A02 Cryptographic Failures | scrypt hashing, TLS 1.3, AES-256 at rest, HMAC-SHA256 JWTs |
| A03 Injection | Parameterized queries (Drizzle), Zod input validation |
| A04 Insecure Design | Threat modeling, principle of least privilege, secure defaults |
| A05 Security Misconfiguration | CSP/HSTS/X-Frame-Options, minimal permissions, no debug in prod |
| A06 Vulnerable Components | Dependabot scanning, minimal dependency surface |
| A07 Auth Failures | Turnstile bot protection, rate limiting, session expiry, secure cookies |
| A08 Data Integrity Failures | Signed JWTs, SRI for CDN assets, build verification |
| A09 Logging Failures | Structured JSON logging, security event auditing via Analytics Engine |
| A10 SSRF | No user-controlled URL fetching, deny-list for internal ranges |

---

## F. AI-Powered Features (Workers AI)

Primary model: `@cf/meta/llama-3.3-70b-instruct-fp8-fast`
Embeddings: `@cf/baai/bge-m3`

### AI Features

1. **SWOT Suggestions** — Based on user's SKEA + existing SWOT, generate 3-5 overlooked entries per quadrant
2. **Smart Goal Validation** — Analyze draft goals against SMART criteria, suggest improvements
3. **Career Insight Reports** — Cross-module pattern analysis with themes, blind spots, priority recommendations (async via Queue)
4. **Elevator Pitch Generator** — Generate 30s/60s/2min pitches from USP + SKEA + brand values
5. **Vision Refinement** — Suggest clearer, more compelling language for vision statements
6. **Decision Recommendations** — Generate M/L/S/S and matrix recommendations from SWOT data

### AI Architecture

```
[API Endpoint] → [Validate + Assemble Prompt] → [Check KV Cache]
                                                      │
                                                [Cache Hit?]
                                                Yes → Return cached
                                                No  → [Workers AI Binding] → [Store in KV] → [Return]
```

Non-urgent features (Career Insight Report) go through Queues for async processing, results stored in D1, user notified via dashboard indicator.

---

## G. Scalability & Performance

- **Edge distribution:** All requests handled at nearest Cloudflare PoP (330+ cities). Sub-50ms for cached content.
- **Zero cold starts:** V8 isolates initialize in <5ms vs 100-500ms for containers
- **Static assets:** `Cache-Control: public, max-age=31536000, immutable` with content-hashed filenames from Vite
- **API caching:** Workers Cache API for public endpoints (case studies), 1h TTL
- **D1 read replicas:** Automatic based on traffic patterns, reducing global read latency
- **Code splitting:** Vite produces tree-shaken, code-split bundles per route
- **Brotli compression:** Enabled by default on Cloudflare edge

---

## H. Multi-tenancy & SaaS Billing

### Subscription Tiers

| | Free | Pro ($12/mo) | Enterprise (Custom) |
|---|---|---|---|
| Visions | 1 | Unlimited | Unlimited |
| SKEA entries | 25 | Unlimited | Unlimited |
| SWOT analyses | 1 | Unlimited | Unlimited |
| Action Plans | 1 | Unlimited | Unlimited |
| AI suggestions | 5/month | 200/month | Unlimited |
| PDF export | No | Yes | Yes |
| Personal Brand | No | Yes | Yes |
| Team workspaces | No | No | Yes |
| Custom SSO | No | No | Yes |
| Data retention | 6 months | Unlimited | Unlimited + SLA |

### Billing

**Stripe** is the only external (non-Cloudflare) service — Cloudflare has no native payment processing. Stripe Checkout sessions created by the Worker, webhooks received at `/api/webhooks/stripe` (signature-verified), subscription tier updated in D1 + KV.

---

## I. Monitoring & Observability

- **Workers Analytics Engine:** Custom events for signups, module completion, AI usage, exports
- **Cloudflare Web Analytics:** Privacy-first page view and Core Web Vitals tracking (no cookies)
- **Structured logging:** JSON format via `console.log()`, streamed via Wrangler tail (dev) or Logpush (prod)
- **Error tracking:** Hono error handler with `@sentry/cloudflare` integration (optional)
- **Health checks:** `GET /api/health` verifying Worker + D1 + KV + R2 connectivity
- **Scheduled Worker:** Cron trigger every 5 minutes for synthetic health checks

---

## J. CI/CD & Environments

### Environments

| Environment | Domain | Database | Purpose |
|---|---|---|---|
| dev | `localhost:5173` | Local (miniflare) | Local development |
| staging | `staging.sparkflow360.com` | `sf360-db-staging` | Pre-production testing |
| production | `app.sparkflow360.com` | `sf360-db-production` | Live |

### GitHub Actions Pipeline

1. **lint-and-test:** `npm ci` → `biome check` → `vitest run` → `tsc --noEmit`
2. **e2e-test:** Playwright tests (depends on #1)
3. **deploy-staging:** D1 migrations → `wrangler deploy --env staging` (on push to `staging`)
4. **deploy-production:** D1 migrations → `wrangler deploy --env production` (on push to `main`)

### Database Migrations

- Drizzle Kit generates SQL files in `drizzle/migrations/`
- Applied via `wrangler d1 migrations apply` in CI/CD before deploy
- D1 Time Travel provides rollback (restore to any minute in last 30 days)
- Destructive migrations use two-phase approach: stop using column first, then drop it

---

## K. Project Structure

```
sparkflow360/
├── src/
│   ├── client/                    # React frontend
│   │   ├── components/            # Shared UI components
│   │   ├── modules/               # Feature modules (vision, skea, swot, etc.)
│   │   ├── hooks/                 # Custom React hooks
│   │   ├── stores/                # Zustand stores
│   │   ├── routes/                # TanStack Router file-based routes
│   │   ├── lib/                   # Client utilities
│   │   └── App.tsx
│   ├── server/                    # Hono API backend
│   │   ├── routes/                # API route handlers per module
│   │   ├── middleware/            # Auth, RBAC, security headers, rate limiting
│   │   ├── services/              # Business logic layer
│   │   └── worker.ts             # Worker entry point
│   └── shared/                    # Shared types and Zod schemas
│       ├── schemas/               # Zod validation schemas (used by both client & server)
│       └── types/                 # TypeScript type definitions
├── drizzle/
│   ├── schema.ts                  # Drizzle ORM schema definitions
│   └── migrations/                # Generated SQL migration files
├── tests/
│   ├── unit/                      # Vitest unit tests
│   ├── integration/               # Vitest integration tests (Workers runtime)
│   └── e2e/                       # Playwright E2E tests
├── wrangler.jsonc                 # Cloudflare Workers configuration
├── vite.config.ts                 # Vite build configuration
├── drizzle.config.ts              # Drizzle Kit configuration
├── biome.json                     # Biome linter/formatter config
├── tsconfig.json                  # TypeScript configuration
├── package.json
└── .github/workflows/deploy.yml   # CI/CD pipeline
```

---

## L. Implementation Plan

### Phase 1: Foundation
- Project scaffolding with Wrangler + Vite + React + Hono
- D1 database setup with Drizzle schema and initial migrations
- Authentication system (register, login, sessions, Turnstile)
- Security middleware (CSP, CORS, rate limiting, RBAC)
- Basic dashboard shell with routing

### Phase 2: Core Modules
- Vision Builder
- SKEA Model
- SWOT Analysis (with SKEA cross-referencing)
- Circle of Concern / Circle of Influence

### Phase 3: Decision & Action
- Decision Making Tools (M/L/S/S + Strategic Matrix)
- Action Planning with SMART Goals
- Case Studies & Learning
- Personal Brand / USP

### Phase 4: AI & Polish
- Workers AI integration for all AI features
- PDF export via Queues
- Email Workers for notifications and reminders
- Dashboard progress visualizations

### Phase 5: SaaS & Launch
- Stripe billing integration
- Feature gating and usage metering
- Staging and production environment setup
- E2E test suite
- Performance optimization and security audit
