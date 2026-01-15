# SlotBase Sprint Tracker

> Track completed sprints and current progress

---

## Sprint History

| Sprint | Theme | Status | Key Deliverables |
|--------|-------|--------|------------------|
| 1 | Project Setup | ✅ Done | Repos created, NestJS/Next.js/Expo initialized, Prisma setup |
| 2 | Database Schema | ✅ Done | Core schema (Users, Facilities, Resources, Bookings) |
| 3 | Authentication | ✅ Done | Clerk integration (API + Web + Mobile), JWT validation |
| 4 | Identity Module | ✅ Done | User CRUD, roles service, profile completion |
| 5 | Organization Module | ✅ Done | Facility CRUD, resources, operating hours, holidays |
| 6 | Booking Core | ✅ Done | Booking holds, confirmations, cancellations |
| 7 | Booking Advanced | ✅ Done | Check-in, no-show, booking events audit trail |
| 8 | Payment Module | ✅ Done | Offline payment recording (cash, check, Venmo, Zelle) |
| 9 | Coach Module | ✅ Done | Coach profiles, blocks, affiliations, students |
| 10 | Admin Module | ✅ Done | Invitations, role management, facility suspension |
| 11 | Email Module | ✅ Done | Resend integration, invitation emails |
| 12 | Web Dashboard | ✅ Done | Facility admin dashboard, resource management |
| 13 | Coach Dashboard | ✅ Done | Coach schedule view, block management |
| 14 | Mobile Scaffold | ✅ Done | Expo setup, auth flow, tab navigation |
| 15 | Onboarding Wizard | ✅ Done | Facility creation wizard (multi-step) |
| **16** | **Player Booking** | 🔄 Current | Player booking flow, availability grid, tier enforcement |
| 17 | Notifications | 📋 Planned | Push notifications, SMS/WhatsApp |
| 18 | Stripe Connect | 📋 Planned | Online payments integration |
| 19 | Recurring Bookings | 📋 Planned | RRULE patterns, series management |
| 20 | Analytics | 📋 Planned | Dashboards, utilization metrics |

---

## Sprint 16 Progress (Current)

**Theme**: Player Booking & Production Readiness
**Start Date**: January 14, 2026
**Pilot Sport**: Cricket

### Day-by-Day Status

| Day | Task | Status | Completed By | Date |
|-----|------|--------|--------------|------|
| 1 | Database schema updates | ✅ Done | Codex | Jan 14 |
| 2 | Availability API endpoint | ✅ Done | Codex | Jan 14 |
| 3 | Public facility page | ✅ Done | Codex | Jan 15 |
| 4 | Booking hold flow | ✅ Done | Codex | Jan 15 |
| 5 | Booking confirmation page | ✅ Done | Codex | Jan 15 |
| 6 | Coach directory | ⬜ Not Started | - | - |
| 7 | Email notifications | ⬜ Not Started | - | - |
| 8 | Player dashboard | ⬜ Not Started | - | - |
| 9 | Tier selection | ⬜ Not Started | - | - |
| 10 | Testing & enforcement | ⬜ Not Started | - | - |

**Status Legend**: ⬜ Not Started | 🔄 In Progress | ✅ Done | ❌ Blocked

---

## What's Built (Sprints 1-15)

### API Modules
```
api/src/
├── identity/        ✅ Users, roles, auth guard
├── organization/    ✅ Facilities, resources, hours
├── booking/         ✅ Holds, bookings, events
├── payment/         ✅ Offline payments, transactions
├── coach/           ✅ Profiles, blocks, students
├── admin/           ✅ Invitations, suspension
└── email/           ✅ Resend templates
```

### Web Pages
```
web/src/app/
├── (auth)/              ✅ Sign in/up (Clerk)
├── complete-profile/    ✅ Role selection
├── onboard/             ✅ Facility wizard
├── (dashboard)/
│   ├── facility/        ✅ Admin dashboard
│   │   ├── bookings/    ✅ Booking management
│   │   ├── resources/   ✅ Resource CRUD
│   │   ├── settings/    ✅ Facility settings
│   │   └── payments/    ✅ Payment recording
│   └── coach/           ✅ Coach dashboard
│       ├── schedule/    ✅ Week view
│       ├── blocks/      ✅ Block management
│       └── students/    ✅ Student list
└── facility/[slug]/     ✅ PUBLIC PAGE (Sprint 16 Day 3)
```

### Mobile Screens
```
mobile/app/
├── (auth)/          ✅ Sign in/up
├── (tabs)/          ✅ Tab structure
│   ├── index        ⬜ Home (placeholder)
│   ├── bookings     ⬜ My bookings (placeholder)
│   ├── book         ⬜ New booking (placeholder)
│   └── profile      ⬜ Profile (placeholder)
└── (coach)/         ⬜ Coach screens (placeholder)
```

---

## Database Tables (60+)

### Core (Sprints 2-5)
- ✅ User, UserRole, UserStatus
- ✅ Organization, Facility, FacilityStatus
- ✅ Resource, ResourceAvailability, ResourcePricing
- ✅ OperatingHours, Holiday

### Booking (Sprints 6-7)
- ✅ Booking, BookingStatus, BookingType
- ✅ BookingHold, BookingEvent
- ✅ BookingParticipant, WaitlistEntry
- ✅ RecurringBooking (schema only, no logic)

### Payment (Sprint 8)
- ✅ BillingAccount, Transaction
- ✅ OfflinePayment, BookingPayment
- ✅ FacilityPaymentConfig

### Coach (Sprint 9)
- ✅ CoachProfile, CoachFacilityAffiliation
- ✅ CoachAvailability, CoachBlock
- ✅ CoachStudent

### Admin (Sprint 10)
- ✅ Invitation, Subscription

### Family (schema only)
- ✅ FamilyGroup, FamilyMember
- ✅ ParentChildLink

---

## API Endpoints Summary

### Implemented (Sprints 1-15)
```
Identity:     GET/PATCH /users/me, GET /users/me/roles
Facilities:   CRUD /facilities, /facilities/:id/resources
Bookings:     POST /hold, /confirm, GET/DELETE /bookings
Payments:     POST/GET/PATCH /payments/offline
Coaches:      GET/PATCH /coaches/me, CRUD /coaches/me/blocks
Admin:        POST /invitations, PATCH /facilities/:id/suspend
```

### To Implement (Sprint 16)
```
GET  /facilities/:id/availability   ✅ Done (Day 2)
GET  /facilities/:id/coaches        ⬜
GET  /coaches/:id                   ⬜
GET  /coaches/:id/availability      ⬜
```

---

## Notes

- **Stripe Connect**: Deferred to Sprint 18. Offline payments work for pilot.
- **Mobile Features**: Deferred to Sprint 17+. Web-first for launch.
- **Recurring Bookings**: Schema exists, logic deferred to Sprint 19.
- **Analytics**: Deferred to Sprint 20. Manual SQL queries for now.

---

*Last Updated: January 15, 2026*
