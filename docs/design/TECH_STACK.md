# SlotBase Tech Stack Specification

## Overview

This document outlines the technology choices for SlotBase, with rationale for each decision.

---

## Core Stack Summary

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Backend** | Node.js + TypeScript | Fast iteration, type safety, shared language with frontend |
| **Framework** | NestJS | Structured, enterprise-ready, great DI |
| **Database** | PostgreSQL | ACID compliance, relational integrity for bookings |
| **Cache** | Redis | Session management, rate limiting, pub/sub |
| **Queue** | BullMQ (Redis) | Background jobs, scheduled tasks |
| **Search** | PostgreSQL Full-Text (Phase 1), Meilisearch (Phase 2+) | Start simple, upgrade when needed |
| **Frontend Web** | Next.js 14+ (App Router) | SSR, SEO, great DX |
| **Frontend Mobile** | React Native / Expo | Code sharing, rapid development |
| **API Style** | REST + WebSocket | REST for CRUD, WebSocket for real-time |
| **Authentication** | Clerk or Auth0 | Secure, managed, social login |
| **Payments** | Stripe Connect | Marketplace model, well-documented |
| **File Storage** | AWS S3 / Cloudflare R2 | R2 for cost-effective egress |
| **Video Processing** | Mux | Adaptive streaming, thumbnails |
| **AI/LLM** | OpenAI GPT-4 + Whisper | Best-in-class for text and transcription |
| **Vector DB** | pgvector (PostgreSQL) | Start integrated, scale to Pinecone if needed |
| **Hosting** | Render (Phase 1), AWS (Phase 3) | Managed simplicity early, control later |
| **CDN** | Cloudflare | Global, fast, DDoS protection |
| **Monitoring** | Sentry + Axiom | Error tracking + logs |
| **Analytics** | PostHog | Product analytics, self-hostable |

---

## Backend Architecture

### Runtime & Language

```yaml
Runtime: Node.js 20 LTS
Language: TypeScript 5.x
Package Manager: pnpm (faster, disk efficient)
```

**Why TypeScript:**
- Type safety catches bugs at compile time
- Shared types between frontend and backend
- Excellent IDE support
- Large ecosystem

### Framework: NestJS

```yaml
Framework: NestJS 10.x
Architecture: Modular monolith
Pattern: Domain-Driven Design (DDD) lite
```

**Why NestJS:**
- Opinionated structure (important for team scaling)
- Built-in dependency injection
- First-class TypeScript support
- Easy to test
- Guards, interceptors, pipes for clean architecture
- OpenAPI/Swagger generation

**Module Structure:**
```
src/
├── modules/
│   ├── identity/           # Auth, users, roles
│   ├── organization/       # Facilities, orgs
│   ├── resource/           # Courts, lanes
│   ├── booking/            # Scheduling engine
│   ├── payment/            # Stripe, offline
│   ├── subscription/       # Plans, entitlements
│   ├── coach/              # Coach management
│   ├── feedback/           # Session feedback, AI
│   ├── progress/           # Player progress
│   ├── notification/       # Email, SMS, push
│   ├── analytics/          # Usage tracking
│   └── facility-ops/       # Devices, lost & found
├── common/
│   ├── decorators/
│   ├── guards/
│   ├── interceptors/
│   ├── filters/
│   └── pipes/
├── config/
├── database/
│   ├── migrations/
│   └── seeds/
└── shared/
    ├── types/
    ├── utils/
    └── constants/
```

### API Layer

```yaml
Primary: REST API (OpenAPI 3.0 spec)
Real-time: WebSocket (Socket.io)
Internal: Event-driven (Redis pub/sub)
```

**REST API Standards:**
- Versioned: `/api/v1/`
- Resource-based URLs
- Consistent error responses
- Pagination: cursor-based
- Rate limiting: per user/facility

**WebSocket Events:**
- Booking updates (real-time availability)
- Notification delivery
- Progress updates

---

## Database Layer

### Primary Database: PostgreSQL 15+

```yaml
Database: PostgreSQL 15
ORM: Prisma (type-safe, great migrations)
Connection Pool: PgBouncer (production)
```

**Why PostgreSQL:**
- ACID compliance essential for booking conflicts
- Excellent JSON support (JSONB)
- Full-text search built-in
- pgvector for embeddings
- Row-level security for multi-tenancy
- Time-series with TimescaleDB extension (optional)

**Key Extensions:**
```sql
-- Required
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";      -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";       -- Encryption
CREATE EXTENSION IF NOT EXISTS "pg_trgm";        -- Fuzzy search

-- Phase 2+
CREATE EXTENSION IF NOT EXISTS "vector";         -- AI embeddings
CREATE EXTENSION IF NOT EXISTS "timescaledb";    -- Time-series (optional)
```

### Prisma ORM

**Why Prisma:**
- Type-safe database access
- Auto-generated types from schema
- Excellent migration system
- Prisma Studio for debugging
- Good performance with query optimization

**Schema Organization:**
```
prisma/
├── schema.prisma           # Main schema
├── migrations/             # Migration history
└── seed.ts                 # Seed data
```

### Redis

```yaml
Provider: Upstash (serverless) or Redis Cloud
Version: 7.x
```

**Use Cases:**
- Session storage
- Rate limiting
- Cache (availability, user data)
- Pub/sub for real-time events
- BullMQ job queue backend

---

## Frontend Architecture

### Web: Next.js 14+

```yaml
Framework: Next.js 14 (App Router)
Styling: Tailwind CSS + shadcn/ui
State: Zustand (simple) + TanStack Query (server)
Forms: React Hook Form + Zod
```

**Why Next.js:**
- Server-side rendering for SEO
- App Router for layouts
- Server Actions for mutations
- Image optimization
- API routes for BFF pattern

**Project Structure:**
```
apps/web/
├── app/
│   ├── (auth)/             # Auth pages
│   ├── (dashboard)/        # Authenticated pages
│   │   ├── facility/
│   │   ├── coach/
│   │   ├── player/
│   │   └── parent/
│   ├── (marketing)/        # Public pages
│   └── api/                # API routes (BFF)
├── components/
│   ├── ui/                 # shadcn components
│   ├── forms/
│   ├── layouts/
│   └── features/
├── lib/
│   ├── api/                # API client
│   ├── hooks/
│   └── utils/
└── styles/
```

### Mobile: React Native + Expo

```yaml
Framework: React Native 0.73+
Tooling: Expo SDK 50+
Navigation: Expo Router
UI: Tamagui or NativeWind
```

**Why Expo:**
- Faster development cycle
- OTA updates
- Managed workflow simplifies deployment
- EAS Build for production

**Shared Code Strategy:**
```
packages/
├── shared/
│   ├── types/              # Shared TypeScript types
│   ├── utils/              # Shared utilities
│   ├── validation/         # Zod schemas
│   └── constants/
```

---

## Authentication & Authorization

### Provider: Clerk (Recommended) or Auth0

```yaml
Provider: Clerk
Features:
  - Email/password
  - Phone (SMS OTP)
  - Google OAuth
  - Apple OAuth
  - Multi-factor authentication
```

**Why Clerk:**
- Modern, developer-friendly
- Built-in UI components
- Excellent React/Next.js integration
- Organizations support (for facilities)
- Webhook support

**Authorization Model:**
```typescript
// Role-based with scope
interface UserRole {
  role: 'platform_admin' | 'facility_owner' | 'facility_admin' |
        'facility_staff' | 'coach' | 'player' | 'parent';
  scope: {
    type: 'platform' | 'organization' | 'facility';
    id: string;
  };
  permissions: string[];
}
```

**Permission System:**
```typescript
// Granular permissions
const PERMISSIONS = {
  // Booking
  'booking:create': 'Create bookings',
  'booking:read': 'View bookings',
  'booking:update': 'Modify bookings',
  'booking:delete': 'Cancel bookings',
  'booking:read:all': 'View all facility bookings',

  // Payments
  'payment:record': 'Record offline payments',
  'payment:refund': 'Process refunds',
  'payment:view:all': 'View all transactions',

  // Coach
  'coach:block:request': 'Request lane blocks',
  'coach:block:approve': 'Approve lane blocks',
  'coach:feedback:create': 'Create session feedback',
  'coach:students:manage': 'Manage student roster',

  // ... more permissions
};
```

---

## Payment Processing

### Provider: Stripe Connect

```yaml
Provider: Stripe
Mode: Connect (Express accounts)
Features:
  - Card payments
  - Apple Pay / Google Pay
  - ACH (US)
  - SEPA (EU)
  - Invoicing
```

**Integration Architecture:**
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Player    │────▶│  SlotBase   │────▶│   Stripe    │
│   (Payer)   │     │  (Platform) │     │  (Connect)  │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Facility   │
                    │  (Payee)    │
                    └─────────────┘
```

**Key Stripe Features Used:**
- Payment Intents (card payments)
- Setup Intents (save cards)
- Subscriptions (recurring)
- Connect Express (onboarding)
- Webhooks (event handling)
- Stripe Tax (optional)

---

## AI & Machine Learning

### LLM Provider: OpenAI

```yaml
Provider: OpenAI
Models:
  - GPT-4 Turbo: Complex reasoning, feedback enhancement
  - GPT-3.5 Turbo: Simple tasks, cost-effective
  - Whisper: Voice transcription
  - text-embedding-3-small: Embeddings
```

**Use Cases by Model:**

| Use Case | Model | Cost Tier |
|----------|-------|-----------|
| Voice transcription | Whisper | Low |
| Text cleanup | GPT-3.5 Turbo | Low |
| Feedback enhancement | GPT-4 Turbo | Medium |
| Insight generation | GPT-4 Turbo | Medium |
| Embeddings | text-embedding-3-small | Low |

### AI Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      AI Service Layer                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │  Transcription  │  │   Enhancement   │  │   Analysis   │ │
│  │     Agent       │  │     Agent       │  │    Agent     │ │
│  └────────┬────────┘  └────────┬────────┘  └──────┬───────┘ │
│           │                    │                   │         │
│           ▼                    ▼                   ▼         │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    OpenAI API Client                     ││
│  │         (with retry, rate limiting, caching)            ││
│  └─────────────────────────────────────────────────────────┘│
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Vector Storage: pgvector

**Why pgvector (Phase 1-2):**
- No additional infrastructure
- Good enough for moderate scale
- Easy to query with SQL

**Migration Path (Phase 3+):**
- Pinecone for scale
- Dedicated vector search

---

## File Storage & Media

### Storage: Cloudflare R2

```yaml
Provider: Cloudflare R2
Backup: AWS S3
CDN: Cloudflare
```

**Why R2:**
- S3-compatible API
- Zero egress fees
- Integrated with Cloudflare CDN
- Cost-effective

**Storage Structure:**
```
slotbase-media/
├── facilities/
│   └── {facility_id}/
│       ├── logo/
│       └── photos/
├── feedback/
│   └── {session_id}/
│       ├── videos/
│       ├── photos/
│       └── thumbnails/
├── profiles/
│   └── {user_id}/
│       └── avatar/
└── documents/
    └── {facility_id}/
        └── waivers/
```

### Video Processing: Mux

```yaml
Provider: Mux
Features:
  - Adaptive bitrate streaming
  - Automatic thumbnails
  - Video analytics
  - Direct uploads
```

**Why Mux:**
- Specialized for video
- Easy integration
- Good pricing
- Handles transcoding

---

## Background Jobs & Queues

### Queue: BullMQ

```yaml
Queue: BullMQ 5.x
Backend: Redis
Dashboard: Bull Board
```

**Job Types:**
```typescript
// Job queue definitions
const QUEUES = {
  // High priority - immediate processing
  'notifications': { priority: 1 },
  'payments': { priority: 1 },

  // Medium priority
  'ai-processing': { priority: 2 },
  'email-sending': { priority: 2 },

  // Low priority - can wait
  'analytics': { priority: 3 },
  'reports': { priority: 3 },
  'cleanup': { priority: 3 },
};
```

**Scheduled Jobs:**
```typescript
const SCHEDULED_JOBS = {
  // Every minute
  'check-booking-reminders': '* * * * *',
  'process-no-shows': '* * * * *',

  // Every hour
  'sync-stripe-subscriptions': '0 * * * *',
  'cleanup-expired-holds': '0 * * * *',

  // Daily
  'generate-daily-reports': '0 6 * * *',
  'check-subscription-renewals': '0 0 * * *',

  // Weekly
  'generate-progress-summaries': '0 9 * * 1',
};
```

---

## Observability

### Error Tracking: Sentry

```yaml
Provider: Sentry
Features:
  - Error tracking
  - Performance monitoring
  - Release tracking
  - Source maps
```

### Logging: Axiom

```yaml
Provider: Axiom
Features:
  - Structured logging
  - Full-text search
  - Dashboards
  - Alerts
```

**Log Format:**
```typescript
interface LogEntry {
  timestamp: string;
  level: 'debug' | 'info' | 'warn' | 'error';
  message: string;
  context: {
    requestId: string;
    userId?: string;
    facilityId?: string;
    module: string;
    action: string;
  };
  metadata?: Record<string, unknown>;
}
```

### Metrics & Analytics: PostHog

```yaml
Provider: PostHog
Features:
  - Product analytics
  - Feature flags
  - Session replay
  - A/B testing
```

---

## Infrastructure

### Phase 1: Managed Platform

```yaml
Compute: Render
Database: Neon PostgreSQL
Redis: Upstash
```

**Why Managed (Phase 1):**
- Zero DevOps overhead
- Auto-scaling
- Easy deployments
- Affordable at small scale

### Phase 3+: AWS

```yaml
Compute: ECS Fargate or EKS
Database: RDS PostgreSQL
Redis: ElastiCache
CDN: CloudFront
```

**Migration Triggers:**
- Need for VPC/private networking
- Compliance requirements
- Cost optimization at scale
- Multi-region deployment

---

## Development Tooling

### Repository Structure (Monorepo)

```yaml
Tool: Turborepo
Package Manager: pnpm
```

```
slotbase/
├── apps/
│   ├── api/                # NestJS backend
│   ├── web/                # Next.js web app
│   ├── mobile/             # React Native app
│   └── admin/              # Internal admin (optional)
├── packages/
│   ├── database/           # Prisma schema & client
│   ├── shared/             # Shared types & utils
│   ├── ui/                 # Shared UI components
│   └── config/             # Shared configs (ESLint, TS)
├── docs/                   # Documentation
├── scripts/                # Build & deploy scripts
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

### Code Quality

```yaml
Linting: ESLint + Prettier
Type Checking: TypeScript strict mode
Testing:
  - Unit: Vitest
  - Integration: Vitest + Supertest
  - E2E: Playwright
Pre-commit: Husky + lint-staged
```

### CI/CD

```yaml
CI: GitHub Actions
CD:
  - Preview: Vercel (web), Render (API)
  - Production: Render → AWS (later)
```

---

## Security Considerations

### Data Protection

- Encryption at rest (database)
- Encryption in transit (TLS 1.3)
- PII encryption (sensitive fields)
- Regular backups with encryption

### API Security

- Rate limiting (per user, per IP)
- Input validation (Zod schemas)
- SQL injection prevention (Prisma)
- XSS prevention (React, Content-Security-Policy)
- CORS configuration

### Compliance

- GDPR (EU users)
- CCPA (California)
- COPPA (children under 13)
- PCI DSS (via Stripe)

---

## Cost Estimates (Phase 1)

| Service | Monthly Cost | Notes |
|---------|-------------|-------|
| Render (API) | $20-50 | Based on usage |
| Upstash Redis | $10 | Serverless |
| Clerk | $25 | Up to 10k MAU |
| Stripe | 2.9% + $0.30 | Per transaction |
| Cloudflare R2 | $5-20 | Storage + requests |
| Mux | $0.025/min | Video storage |
| OpenAI | $50-200 | Based on usage |
| Sentry | $26 | Team plan |
| Axiom | $25 | Starter |
| PostHog | $0 | Free tier |
| **Total** | ~$160-350/mo | Before scale |

---

## Technology Decision Log

| Decision | Chosen | Alternatives Considered | Rationale |
|----------|--------|------------------------|-----------|
| Backend Language | TypeScript | Go, Python | Team velocity, type safety |
| Backend Framework | NestJS | Fastify, Express | Structure, enterprise features |
| Database | PostgreSQL | MongoDB | ACID for bookings, relational |
| ORM | Prisma | TypeORM, Drizzle | DX, type safety, migrations |
| Auth | Clerk | Auth0, Supabase Auth | Modern DX, React integration |
| Payments | Stripe | Square, PayPal | Developer experience, Connect |
| AI Provider | OpenAI | Anthropic, Google | Best models, Whisper |
| Hosting | Render | Railway, Fly.io | Balance of simplicity and control |
| Queue | BullMQ | RabbitMQ, SQS | Simple, Redis-based |

---

## What We're NOT Using (And Why)

| Technology | Why Not |
|------------|---------|
| GraphQL | Adds complexity; REST sufficient for our needs |
| MongoDB | Need ACID transactions for booking conflicts |
| Kubernetes | Overkill until 100+ facilities |
| Kafka | BullMQ sufficient for our event volume |
| Microservices | Monolith-first; extract when needed |
| Self-hosted auth | Security risk; use managed provider |
| Custom video processing | Mux handles edge cases we'd miss |

---

*Last Updated: 2024-01-10*
*Version: 1.0*
