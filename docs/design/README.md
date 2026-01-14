# SlotBase Design Documentation

## Sports Facility Scheduling & Payments Platform

### Overview

SlotBase is a comprehensive sports facility management platform supporting tennis courts, cricket lanes, pickleball courts, basketball courts, badminton, swimming lanes, and more.

### Documentation Index

| Document | Description |
|----------|-------------|
| [Tech Stack](./TECH_STACK.md) | Technology choices and rationale |
| [System Architecture](./SYSTEM_ARCHITECTURE.md) | High-level system design |
| [Data Models](../schemas/DATA_MODELS.md) | Database schemas and relationships |
| [API Design](../api/API_DESIGN.md) | REST API specifications |
| [AI Agents](../agents/AI_AGENTS.md) | AI agent specifications, rules, conflict resolution |
| [Payment System](./PAYMENT_SYSTEM.md) | Payment processing design |
| [Coach & Feedback](./COACH_FEEDBACK_SYSTEM.md) | Coach feedback and progress tracking |
| [Notifications](./NOTIFICATION_SYSTEM.md) | Notification system design |
| [Hooks & Events](./HOOKS_EVENTS.md) | Event-driven architecture and hooks |
| [Idempotency](./IDEMPOTENCY.md) | Idempotency keys, retry semantics, caching |
| [Cascade Rules](./CASCADE_RULES.md) | Delete behavior, data integrity, GDPR compliance |
| [Error Recovery](./ERROR_RECOVERY.md) | Failure handling, saga patterns, rollback |

### Business Models Supported

1. **Facility Subscription** - Monthly SaaS fee ($49-$299/month)
2. **Player Subscription** - Optional premium features
3. **Coaching Memberships** - Bundled coaching packages
4. **Drop-in Bookings** - One-time payments
5. **Offline Payments** - Cash, check, Venmo tracking

### Key Personas

- **Facility Owner/Admin** - Manages facility, billing, staff
- **Facility Staff** - Day-to-day operations
- **Coach** - Manages lessons, provides feedback
- **Player (Adult)** - Self-managed bookings
- **Parent/Guardian** - Manages child players
- **Child Player** - Managed by parent

### Design Principles

1. **Facility-centric, player-friendly** - Facilities pay, players must love it
2. **Revenue before features** - Every feature increases revenue or reduces pain
3. **Opinionated defaults** - Works out of the box
4. **Payments are the product** - Integrated payments = stickiness
5. **Time is inventory** - Unsold slots expire forever
6. **AI-enhanced, human-approved** - AI assists, humans decide

### Phase Roadmap

- **Phase 0**: Validation & pilots (manual concierge)
- **Phase 1**: MVP (booking, payments, basic coaching)
- **Phase 2**: Monetization (subscriptions, progress tracking, AI)
- **Phase 3**: Scale (enterprise, multi-facility, advanced AI)

---

*Last Updated: 2026-01-11*
