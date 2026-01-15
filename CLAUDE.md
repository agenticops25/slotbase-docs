# SlotBase - Project Context for Claude

> This file preserves project context across Claude Code sessions. Update as the project evolves.

## Project Overview

**SlotBase** is a facility-centric sports scheduling & payments platform supporting:
- Cricket nets, bowling machine lanes, tennis courts, pickleball courts, basketball courts, badminton, swimming lanes

**Pilot Facility**: Cricket Academy (Indoor nets, bowling machines, practice lanes)

**Platform Components:**
- Web app (facility admin + public booking)
- Mobile apps (iOS + Android via React Native/Expo)
- Calendar-based scheduling
- Payments (online via Stripe + offline recording)
- Subscriptions (facility-paid and optional player-paid)

## Leadership Perspective

Claude acts as the founding leadership team:
- **CEO**: Product vision, customer pain, pricing psychology
- **CTO**: System architecture, scalability, mobile + web strategy
- **CFO**: Monetization, payments, risk, compliance, revenue predictability
- **Principal Architect**: Data models, module boundaries, correctness

## Core Business Constraints

### Payment Model (CRITICAL)
- Stripe is **OPTIONAL** during facility onboarding
- Facilities can start immediately with offline payments only
- Three modes: Offline-only, Stripe-enabled, Hybrid
- When Stripe enabled, use Stripe Connect (facility's own account)
- Must support recording cash/Venmo/Zelle/check payments

### Subscription Model
- **Facility-paid**: $49-$299+/month for platform access with player limits
- **Player-paid**: Optional overflow or premium features
- Facilities may bundle into coaching programs
- Drop-in bookings supported

### Pricing Tiers
| Tier | Price | Active Players | Resources | Features |
|------|-------|----------------|-----------|----------|
| Starter | $49/mo | 50 | 4 | Basic scheduling, offline payments |
| Growth | $129/mo | 150 | 10 | + Stripe, basic analytics |
| Pro | $299/mo | 500 | 25 | + AI insights, video, multi-location |
| Enterprise | Custom | Unlimited | Unlimited | Full suite, dedicated support |

## Tech Stack Summary

### Backend
- **Runtime**: Node.js 20+ with TypeScript (strict mode)
- **Framework**: NestJS (modular monolith architecture)
- **ORM**: Prisma with PostgreSQL
- **Cache/Queue**: Redis + BullMQ
- **API**: REST with OpenAPI spec

### Frontend
- **Web**: Next.js 14+ (App Router, Server Components)
- **Mobile**: React Native + Expo (SDK 50+)
- **State**: TanStack Query + Zustand
- **UI**: Tailwind CSS (web), NativeWind (mobile)

### Infrastructure
- **Database**: PostgreSQL 15+ (Neon/Supabase free tier for MVP)
- **Cache**: Upstash Redis (free tier for MVP)
- **Storage**: Cloudflare R2 / AWS S3
- **Hosting**: Render (Phase 1) → AWS (Phase 3)
- **Auth**: Clerk or Auth0

### AI/ML
- **LLM**: OpenAI GPT-4 (feedback enhancement, insights)
- **Transcription**: Whisper API
- **Embeddings**: text-embedding-3-small with pgvector

### Payments
- **Provider**: Stripe Connect (Express accounts)
- **Platform Fee**: 0.5-2% based on tier
- **Offline**: Full support for cash/check/Venmo/Zelle recording

## 8 AI Agents

1. **Scheduling Fairness Agent** - Ensures equitable slot distribution
2. **Capacity & Limits Agent** - Enforces subscription limits
3. **Feedback Enhancement Agent** - AI-enhances coach session notes
4. **Progress Insights Agent** - Generates player progress summaries
5. **Video Analysis Agent** - Extracts highlights from session videos
6. **Smart Notification Agent** - Optimizes notification timing/channel
7. **Payment Enforcement Agent** - Manages payment compliance
8. **No-Show Detection Agent** - Tracks and flags no-show patterns

## Design Documents

All design documentation lives in `/docs/`:

| Document | Path | Description |
|----------|------|-------------|
| Overview | [docs/design/README.md](docs/design/README.md) | Main index, business models, personas, principles |
| Tech Stack | [docs/design/TECH_STACK.md](docs/design/TECH_STACK.md) | Complete technology choices with rationale |
| System Architecture | [docs/design/SYSTEM_ARCHITECTURE.md](docs/design/SYSTEM_ARCHITECTURE.md) | Module specs, data flows, deployment |
| AI Agents | [docs/agents/AI_AGENTS.md](docs/agents/AI_AGENTS.md) | 8 agent specifications with rules + conflict resolution |
| Data Models | [docs/schemas/DATA_MODELS.md](docs/schemas/DATA_MODELS.md) | Complete Prisma schema |
| Hooks & Events | [docs/design/HOOKS_EVENTS.md](docs/design/HOOKS_EVENTS.md) | Event-driven architecture, webhooks |
| Payment System | [docs/design/PAYMENT_SYSTEM.md](docs/design/PAYMENT_SYSTEM.md) | Stripe Connect, offline payments |
| API Design | [docs/api/API_DESIGN.md](docs/api/API_DESIGN.md) | REST endpoints, WebSocket API |
| Coach Feedback | [docs/design/COACH_FEEDBACK_SYSTEM.md](docs/design/COACH_FEEDBACK_SYSTEM.md) | Feedback workflow, AI enhancement |
| Notifications | [docs/design/NOTIFICATION_SYSTEM.md](docs/design/NOTIFICATION_SYSTEM.md) | Multi-channel notification system |
| Idempotency | [docs/design/IDEMPOTENCY.md](docs/design/IDEMPOTENCY.md) | Idempotency keys, retry semantics |
| Cascade Rules | [docs/design/CASCADE_RULES.md](docs/design/CASCADE_RULES.md) | Delete behavior, GDPR compliance |
| Error Recovery | [docs/design/ERROR_RECOVERY.md](docs/design/ERROR_RECOVERY.md) | Failure handling, saga patterns |

## Key User Personas

1. **Facility Admin** - Manages facility, resources, staff, settings
2. **Coach** - Schedules sessions, blocks time, sends feedback
3. **Player** - Books sessions, views progress, makes payments
4. **Parent/Guardian** - Manages multiple children, handles payments
5. **Front Desk Staff** - Day-to-day operations, check-ins

## Implementation Phases

### Phase 0: Validation (Design Complete ✅)
- Market validation, pilot facility agreements
- Core design documents

### Phase 1: MVP (Next)
- Facility onboarding & management
- Resource management (courts/lanes)
- Basic scheduling & booking
- Offline payment recording
- Coach & player management
- Basic notifications

### Phase 2: Monetization
- Stripe Connect integration
- Subscription management
- Platform fee collection
- Advanced analytics

### Phase 3: Scale
- Multi-location support
- AI-powered insights
- Video analysis
- Enterprise features

## Current Status

**Phase: Sprint 16 - Player Booking & Production Readiness** (January 2026)

**Completed (Sprints 1-15):**
- API: NestJS backend with 7 domain modules (Identity, Organization, Booking, Payment, Coach, Admin, Email)
- Web: Next.js frontend with facility admin dashboard, coach dashboard
- Mobile: React Native/Expo scaffold with auth flow
- Database: PostgreSQL on Neon with comprehensive 60+ table schema
- Auth: Clerk integration across all platforms
- Deployment: Render (API), Vercel (Web), EAS (Mobile)

**Sprint 16 Goals (Current):**
1. Player-facing booking flow (Web)
2. Facility availability grid
3. Open Play + Lesson booking
4. Coach directory & selection
5. Email notifications (booking confirmations)
6. Subscription tier selection in onboarding
7. Free tier limit enforcement (20 players, 3 resources)

**Pilot Facility:** Cricket Academy (working closely with pilot customer)

**Team Structure:**
- Principal Architect: Design authority, sprint planning
- Codex: Senior Developer, implementation

**Sprint 16 Design:** `docs/docs/sprints/SPRINT_16_DESIGN.md`

## Project Structure (Current)

```
slotbase-project/
├── api/                     # NestJS backend (separate git repo)
│   ├── src/
│   ├── prisma/
│   └── package.json
├── web/                     # Next.js web app (separate git repo)
│   ├── src/
│   └── package.json
├── mobile/                  # React Native + Expo (separate git repo)
│   ├── app/
│   └── package.json
└── docs/                    # Design documentation
    ├── design/              # Design documents
    ├── api/                 # API documentation
    ├── agents/              # AI agent specs
    └── schemas/             # Data model docs
```

**Git Repositories:**
- `github.com/agenticops25/slotbase-api`
- `github.com/agenticops25/slotbase-web`
- `github.com/agenticops25/slotbase-mobile`

## Key Design Decisions

1. **Modular Monolith** - Start simple, extract services later if needed
2. **PostgreSQL as source of truth** - Correctness over eventual consistency
3. **Stripe Optional** - Never block facilities from starting
4. **Event-Driven** - Domain events enable future scaling
5. **Mobile-First** - Most users will be on mobile
6. **AI Enhancement** - Not AI replacement (coach feedback, not AI coach)

## Engineering Philosophy

- **Understand before changing** - Read existing patterns and code before writing new code
- **Simplify ruthlessly** - Delete complexity that doesn't earn its keep; three clear lines beat one clever abstraction
- **Solve the real problem** - Address root causes, not symptoms; question assumptions about what's actually needed
- **Plan before building** - Sketch the approach mentally first; if you can't explain it simply, you don't understand it yet

## DO NOT

- Make Stripe mandatory for facility signup
- Allow double-booking under any circumstance
- Store raw credit card numbers
- Build complex multi-tenancy before PMF
- Over-engineer before validation
- Skip offline payment support

## Important Constraints

- COPPA compliance for players under 13
- PCI DSS compliance for payment data
- Timezone handling for multi-location facilities
- Idempotency for all payment operations
- Audit trail for all financial transactions
