# Codex Daily Handoff Document

> **Role**: Senior Developer
> **Reports To**: Principal Architect
> **Current Sprint**: 16 - Player Booking & Production Readiness

---

## First Time? Start Here

If this is your first session, read `docs/CODEX_START_HERE.md` first.

---

## Documents Reference

| Document | Purpose |
|----------|---------|
| `CODEX_START_HERE.md` | First-time orientation |
| `SPRINT_TRACKER.md` | Sprint history & status |
| `docs/sprints/SPRINT_16_DESIGN.md` | Current sprint work instructions |
| `CODEX_HANDOFF.md` | This file - daily reference |
| `CLAUDE.md` | Full project context |

---

## Quick Start (Every Session)

### 1. Check Sprint Status
```
Read: docs/SPRINT_TRACKER.md (see which day you're on)
```

### 2. Get Today's Tasks
```
Read: docs/docs/sprints/SPRINT_16_DESIGN.md
```

### 2. Identify Current Day/Task
The Sprint 16 design document has **10 days of tasks**. Each day has:
- Clear objectives
- Specific file paths
- Code snippets to implement
- Deliverables checklist
- Verification steps

### 3. Environment Setup
```bash
# Terminal 1: API
cd /Users/srivasta/Public/source/slotbase-project/api
pnpm install
pnpm prisma migrate dev  # Apply any pending migrations
pnpm dev                 # Starts on port 3001

# Terminal 2: Web
cd /Users/srivasta/Public/source/slotbase-project/web
pnpm install
pnpm dev                 # Starts on port 3000
```

### 4. Verify Services Running
- API: http://localhost:3001/api/v1 (should return OK)
- API Docs: http://localhost:3001/docs (Swagger UI)
- Web: http://localhost:3000

---

## Project Architecture

```
slotbase-project/
├── api/                    # NestJS Backend
│   ├── src/
│   │   ├── identity/       # Users, roles, auth
│   │   ├── organization/   # Facilities, resources
│   │   ├── booking/        # Bookings, holds
│   │   ├── payment/        # Payments, transactions
│   │   ├── coach/          # Coach profiles, blocks
│   │   ├── admin/          # Admin operations
│   │   └── email/          # Resend integration
│   └── prisma/
│       ├── schema.prisma   # Database schema
│       └── seed.ts         # Seed data
│
├── web/                    # Next.js Frontend
│   └── src/
│       ├── app/            # App Router pages
│       ├── components/     # React components
│       └── lib/
│           └── api/        # API client & hooks
│
├── mobile/                 # React Native + Expo
│   └── app/               # Expo Router pages
│
└── docs/                  # Documentation
    ├── CLAUDE.md          # Project context
    ├── CODEX_HANDOFF.md   # This file
    └── docs/
        ├── sprints/       # Sprint design docs
        ├── design/        # Architecture docs
        └── api/           # API docs
```

---

## Tech Stack Reference

| Layer | Technology | Documentation |
|-------|-----------|---------------|
| **API Framework** | NestJS 11 | https://docs.nestjs.com |
| **ORM** | Prisma 6 | https://www.prisma.io/docs |
| **Database** | PostgreSQL 15 (Neon) | https://neon.tech/docs |
| **Web Framework** | Next.js 16 | https://nextjs.org/docs |
| **UI Components** | shadcn/ui | https://ui.shadcn.com |
| **State Management** | TanStack Query | https://tanstack.com/query |
| **Auth** | Clerk | https://clerk.com/docs |
| **Email** | Resend | https://resend.com/docs |

---

## Sprint 16 Task Summary

| Day | Task | Status |
|-----|------|--------|
| 1 | Database schema updates (SubscriptionUsage, TierFeature, cricket seed) | Pending |
| 2 | Availability API endpoint | Pending |
| 3 | Public facility page + availability grid | Pending |
| 4 | Booking hold creation flow | Pending |
| 5 | Booking confirmation page | Pending |
| 6 | Coach directory & selection | Pending |
| 7 | Email notifications | Pending |
| 8 | Player dashboard | Pending |
| 9 | Tier selection in onboarding | Pending |
| 10 | Tier limit enforcement & E2E testing | Pending |

**Update this table as you complete each day's tasks.**

---

## Key Files to Know

### API
- `api/prisma/schema.prisma` - Database schema (source of truth)
- `api/src/booking/booking.service.ts` - Core booking logic
- `api/src/booking/booking.controller.ts` - Booking endpoints
- `api/src/organization/facility.controller.ts` - Facility endpoints

### Web
- `web/src/lib/api/client.ts` - API client with auth
- `web/src/lib/api/hooks/` - TanStack Query hooks
- `web/src/components/ui/` - shadcn/ui components
- `web/src/app/(dashboard)/` - Authenticated routes

---

## Coding Standards

### API (NestJS)
```typescript
// Controllers: Handle HTTP, delegate to services
@Get(':id')
async getFacility(@Param('id') id: string) {
  return this.facilityService.findById(id);
}

// Services: Business logic, database access
async findById(id: string) {
  const facility = await this.prisma.facility.findUnique({
    where: { id },
    include: { resources: true },
  });
  if (!facility) throw new NotFoundException('Facility not found');
  return facility;
}

// DTOs: Validate input
export class CreateBookingDto {
  @IsUUID()
  facilityId: string;

  @IsDateString()
  startAt: string;
}
```

### Web (Next.js)
```typescript
// Pages: Server components by default
export default async function FacilityPage({ params }) {
  const facility = await getFacility(params.slug);
  return <FacilityDetail facility={facility} />;
}

// Client components: Use 'use client' directive
'use client';
import { useQuery } from '@tanstack/react-query';

export function AvailabilityGrid({ facilityId }) {
  const { data, isLoading } = useQuery({
    queryKey: ['availability', facilityId],
    queryFn: () => apiClient.get(`/facilities/${facilityId}/availability`),
  });
  // ...
}
```

---

## Common Commands

### Database
```bash
cd api
pnpm prisma migrate dev --name description   # Create migration
pnpm prisma db push                          # Push without migration
pnpm prisma db seed                          # Run seed script
pnpm prisma studio                           # Visual database browser
```

### Testing
```bash
# API
cd api
pnpm test                    # Unit tests
pnpm test:e2e               # E2E tests

# Web
cd web
pnpm test                   # Playwright E2E
pnpm test:ui                # Playwright with UI
```

### Linting
```bash
pnpm lint                   # Run ESLint
pnpm format                 # Run Prettier (API only)
```

---

## Git Workflow

```bash
# Before starting work
git pull origin main

# After completing a day's tasks
git add .
git commit -m "Sprint 16 Day X: [Brief description]

- Added availability endpoint
- Created facility page component
- Implemented booking hold flow"

git push origin main
```

---

## Troubleshooting

### API won't start
1. Check `.env` file exists with all required vars
2. Run `pnpm prisma generate` to regenerate client
3. Check PostgreSQL connection string

### Web won't connect to API
1. Verify API is running on port 3001
2. Check `NEXT_PUBLIC_API_URL` env var
3. Check CORS settings in API

### Auth not working
1. Verify Clerk keys in both API and Web `.env`
2. Check Clerk dashboard for webhook status
3. Clear browser cookies and re-login

---

## Questions?

If requirements are unclear:
1. Check the Sprint 16 design doc first
2. Look at existing similar implementations
3. Note the question and continue with best judgment
4. Flag for Principal Architect review

---

## Daily Checklist

- [ ] Read current day's tasks in SPRINT_16_DESIGN.md
- [ ] Pull latest code
- [ ] Start API and Web servers
- [ ] Implement day's deliverables
- [ ] Test each feature manually
- [ ] Commit with descriptive message
- [ ] Update task status in this document
- [ ] Push changes

---

*Last Updated: January 14, 2026*
*Sprint: 16*
*Day: 1 (Not Started)*
