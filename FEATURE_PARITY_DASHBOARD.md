# SlotBase Feature Parity Dashboard

> Single source of truth for feature completion across all platforms.
> **Update this after every sprint.**

---

## Last Updated: January 15, 2026 (Post-Sprint 16 Audit)

---

## Legend

| Symbol | Meaning |
|--------|---------|
| ✅ | Done - Fully implemented and tested |
| ⚠️ | Partial - Implemented but incomplete or has bugs |
| ❌ | Missing - Not implemented |
| 🚫 | N/A - Not applicable for this platform |
| 📋 | Planned - Scheduled for future sprint |

---

## 1. Identity & Authentication

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| User sign-up (Clerk) | ✅ | ✅ | ✅ | ❌ | ❌ | |
| User sign-in (Clerk) | ✅ | ✅ | ✅ | ❌ | ❌ | |
| User profile view | ✅ | ✅ | ⚠️ | ❌ | ❌ | Mobile: Clerk data only |
| User profile update | ✅ | ⚠️ | ❌ | ❌ | ❌ | Web: partial fields |
| Role management | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Complete profile flow | ✅ | ❌ | ❌ | ❌ | ❌ | Page missing |

---

## 2. Facility Onboarding

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Create invitation | ✅ | ⚠️ | 🚫 | ❌ | ❌ | Web: UI not wired |
| Accept invitation | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Onboarding wizard | ✅ | ✅ | 🚫 | ❌ | ❌ | Fixed Jan 15 |
| Facility info step | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Resources step | ✅ | ✅ | 🚫 | ❌ | ❌ | Fixed Jan 15 |
| Operating hours step | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Tier selection step | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Publish facility | ✅ | ✅ | 🚫 | ❌ | ❌ | |

---

## 3. Resource Management

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Create resource | ✅ | ✅ | ❌ | ❌ | ❌ | Fixed Jan 15 |
| List resources | ✅ | ✅ | ❌ | ❌ | ❌ | Mobile: mock data |
| Update resource | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Delete resource | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Resource pricing UI | ✅ | ❌ | ❌ | ❌ | ❌ | Sprint 18 |
| Resource availability UI | ✅ | ❌ | ❌ | ❌ | ❌ | Sprint 18 |
| Resource reordering | ✅ | ❌ | ❌ | ❌ | ❌ | Sprint 18 |
| Resource themes/colors | ❌ | ❌ | ❌ | ❌ | ❌ | Not planned |

---

## 4. Staff & Roles

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Facility admin role | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Coach role | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Front desk role | ✅ | ❌ | ❌ | ❌ | ❌ | Schema only |
| Invite staff | ✅ | ⚠️ | 🚫 | ❌ | ❌ | Web: not wired |
| Remove staff | ✅ | ❌ | 🚫 | ❌ | ❌ | Sprint 20 |
| Staff list view | ✅ | ❌ | 🚫 | ❌ | ❌ | Sprint 20 |

---

## 5. Players & Parents

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Player registration | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Player invitation | ✅ | ⚠️ | ❌ | ❌ | ❌ | |
| Self-service join | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Join request approval | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Parent-child linking | ✅ | ❌ | ❌ | ❌ | ❌ | Sprint 19 |
| Family groups | ✅ | ❌ | ❌ | ❌ | ❌ | Sprint 19 |
| Book for child | ⚠️ | ❌ | ❌ | ❌ | ❌ | Sprint 19 |

---

## 6. Bookings

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| View availability | ✅ | ✅ | ❌ | ❌ | ❌ | Mobile: mock |
| Create hold (5-min) | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Confirm booking | ✅ | ✅ | ❌ | ❌ | ❌ | |
| View booking details | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Cancel booking | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Check-in | ✅ | ✅ | ❌ | ❌ | ❌ | |
| No-show marking | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Booking history | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Price display | ✅ | ⚠️ | ❌ | ❌ | ❌ | Sprint 17: fix hardcoded |
| Recurring bookings | ⚠️ | ❌ | ❌ | ❌ | ❌ | Sprint 21 |
| Waitlist | ✅ | ❌ | ❌ | ❌ | ❌ | Sprint 21 |

---

## 7. Coach Features

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Coach profile | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Coach dashboard | ✅ | ⚠️ | ❌ | ❌ | ❌ | Sprint 17: fix mock data |
| Schedule view | ✅ | ✅ | ❌ | ❌ | ❌ | Mobile: mock |
| Block requests | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Student management | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Session feedback | ✅ | ⚠️ | ❌ | ❌ | ❌ | |
| Coach directory | ✅ | ✅ | ❌ | ❌ | ❌ | |

---

## 8. Payments

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Record offline payment | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Verify payment | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Void payment | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Payment history | ✅ | ✅ | ❌ | ❌ | ❌ | |
| Payment methods config | ✅ | ❌ | ❌ | ❌ | ❌ | |
| Stripe Connect setup | ❌ | ❌ | ❌ | ❌ | ❌ | Sprint 23-24 |
| Online card payments | ❌ | ❌ | ❌ | ❌ | ❌ | Sprint 23-24 |
| Refunds | ⚠️ | ❌ | ❌ | ❌ | ❌ | Sprint 23-24 |

---

## 9. Subscriptions & Limits

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Tier features | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Subscription management | ✅ | ⚠️ | 🚫 | ❌ | ❌ | Admin UI incomplete |
| Active player counting | ✅ | ❌ | 🚫 | ❌ | ❌ | |
| Over-limit enforcement | ✅ | ⚠️ | 🚫 | ❌ | ❌ | |
| Usage dashboard | ❌ | ❌ | 🚫 | ❌ | ❌ | Sprint 22 |

---

## 10. Notifications

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Booking confirmation email | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Invitation email | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Welcome email | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Join request emails | ✅ | ✅ | 🚫 | ❌ | ❌ | |
| Push notifications | ❌ | 🚫 | ❌ | ❌ | ❌ | Sprint 25+ |
| SMS notifications | ❌ | 🚫 | ❌ | ❌ | ❌ | Not planned |

---

## 11. Platform Admin

| Feature | API | Web | Mobile | Tests | Docs | Notes |
|---------|-----|-----|--------|-------|------|-------|
| Admin dashboard | ✅ | ⚠️ | 🚫 | ❌ | ❌ | Sprint 17: fix mock |
| Facility list | ✅ | ⚠️ | 🚫 | ❌ | ❌ | Sprint 17: build |
| Facility detail | ✅ | ❌ | 🚫 | ❌ | ❌ | Sprint 22 |
| Suspend facility | ✅ | ❌ | 🚫 | ❌ | ❌ | Sprint 22 |
| Invitation management | ✅ | ⚠️ | 🚫 | ❌ | ❌ | Sprint 17: wire |
| Subscription override | ✅ | ❌ | 🚫 | ❌ | ❌ | Sprint 22 |

---

## Summary Statistics

### By Platform

| Platform | Done | Partial | Missing | Total | Completion |
|----------|------|---------|---------|-------|------------|
| API | 62 | 4 | 5 | 71 | 87% |
| Web | 38 | 12 | 21 | 71 | 54% |
| Mobile | 2 | 1 | 52 | 55 | 4% |
| Tests | 0 | 0 | 71 | 71 | 0% |
| Docs | 0 | 0 | 71 | 71 | 0% |

### By Priority

| Priority | Count | Status |
|----------|-------|--------|
| Critical (Booking) | 12 | ⚠️ 1 bug (price display) |
| High (Payments) | 8 | ✅ Offline complete |
| High (Admin) | 6 | ⚠️ 3 bugs (mock data) |
| Medium (Coach) | 8 | ⚠️ 1 bug (mock data) |
| Low (Mobile) | 55 | ❌ Deferred |

---

## Sprint Roadmap

| Sprint | Focus | Key Deliverables |
|--------|-------|------------------|
| **17** | Stabilization | Fix bugs, wire admin pages, E2E tests |
| **18** | Resource UI | Pricing UI, availability schedule UI |
| **19** | Parent/Family | Parent-child linking, book for child |
| **20** | Staff Management | Invite flow, staff list, permissions |
| **21** | Booking Advanced | Recurring bookings, waitlist |
| **22** | Platform Admin | Analytics, facility management |
| **23-24** | Stripe Connect | Online payments |
| **25+** | Mobile | Connect mobile to real APIs |

---

## How to Update This Document

After each sprint:
1. Review all features touched in the sprint
2. Update status symbols (✅/⚠️/❌)
3. Add notes for partial implementations
4. Update summary statistics
5. Commit with message: `docs: update feature parity dashboard post-sprint X`

---

*Created: January 15, 2026*
*Next Update: After Sprint 17*
