# Codex - Start Here

> **You are**: Senior Developer on SlotBase project
> **Your manager**: Principal Architect (provides design specs)
> **Current Sprint**: 16

---

## Your First Command

```
Read this file first, then read the Sprint 16 design document.
```

---

## Project Location

```
/Users/srivasta/Public/source/slotbase-project/
├── api/      # NestJS backend (port 3001)
├── web/      # Next.js frontend (port 3000)
├── mobile/   # React Native (not needed for Sprint 16)
└── docs/     # Documentation (you are here)
```

---

## Documents to Read (In Order)

| Order | Document | Purpose |
|-------|----------|---------|
| 1 | `docs/CODEX_START_HERE.md` | This file - orientation |
| 2 | `docs/SPRINT_TRACKER.md` | See what's done vs. pending |
| 3 | `docs/docs/sprints/SPRINT_16_DESIGN.md` | **Your work instructions** |
| 4 | `docs/CODEX_HANDOFF.md` | Reference guide & checklist |
| 5 | `docs/CLAUDE.md` | Full project context (if needed) |

---

## Sprint 16 Summary

**Goal**: Build the player-facing booking flow so players can:
1. View a cricket facility's availability
2. Select a time slot
3. Create a booking hold (5 min)
4. Confirm booking with payment method
5. Receive email confirmation
6. See their bookings in a dashboard

**Pilot Facility**: Cricket Academy (nets, bowling machines)

---

## Where to Start

### Day 1 Tasks (Start Here)

**Objective**: Update database schema and seed data

**Files to modify**:
1. `api/prisma/schema.prisma` - Add 2 new models
2. `api/prisma/seed.ts` - Update with cricket data

**What to add to schema.prisma**:
- `SubscriptionUsage` model (tracks usage per billing period)
- `TierFeature` model (defines limits per subscription tier)

**What to update in seed.ts**:
- Change "Pilot Tennis Club" → "Pilot Cricket Academy"
- Change "Tennis Court" resources → "Cricket Net" resources
- Add tier feature seed data

**Detailed code**: See `docs/docs/sprints/SPRINT_16_DESIGN.md` → Day 1

**Commands to run after changes**:
```bash
cd api
pnpm prisma migrate dev --name add_subscription_features
pnpm prisma db seed
```

---

## Daily Workflow

```
1. cd /Users/srivasta/Public/source/slotbase-project
2. Read SPRINT_16_DESIGN.md for current day's tasks
3. Start servers:
   - Terminal 1: cd api && pnpm dev
   - Terminal 2: cd web && pnpm dev
4. Implement the day's deliverables
5. Test using verification steps in design doc
6. Commit:
   git add .
   git commit -m "Sprint 16 Day X: [description]"
7. Update SPRINT_TRACKER.md with completion status
```

---

## Key Rules

1. **Follow the design doc exactly** - Code snippets are provided
2. **Test as you go** - Each day has verification steps
3. **Commit after each day** - Don't batch multiple days
4. **Cricket, not tennis** - All examples use cricket terminology
5. **Web only this sprint** - Mobile work is deferred

---

## If You Get Stuck

1. Check existing similar code in the codebase
2. Refer to the design doc's code snippets
3. Look at tech docs (NestJS, Prisma, Next.js)
4. Note the blocker and continue with next task
5. Flag for Principal Architect review

---

## Quick Reference

### API Patterns
```typescript
// Controller
@Get(':id/availability')
@Public()
async getAvailability(@Param('id') id: string, @Query() query: GetAvailabilityDto) {
  return this.service.getAvailability(id, query.date);
}

// Service
async getAvailability(facilityId: string, date: string) {
  // Business logic here
}
```

### Web Patterns
```typescript
// Server Component (default)
export default async function Page({ params }) {
  const data = await fetchData(params.id);
  return <Component data={data} />;
}

// Client Component
'use client';
export function InteractiveComponent() {
  const { data } = useQuery({ queryKey: ['key'], queryFn: fetchFn });
  return <div>{data}</div>;
}
```

---

## Start Now

1. Open `docs/docs/sprints/SPRINT_16_DESIGN.md`
2. Go to **Day 1: Database Schema Updates**
3. Follow the tasks step by step
4. Come back here if you need orientation

**Good luck!** 🏏
