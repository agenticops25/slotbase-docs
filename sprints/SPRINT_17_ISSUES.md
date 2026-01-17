# Sprint 17: Stabilization & Integration Testing

**Sprint Goal**: Fix all bugs found, ensure end-to-end flows work without errors. No new features - only fixing what's broken.

**Start Date**: January 16, 2026
**Duration**: 1 week

---

## Issue Summary

| Issue | Title | Priority | Estimate | Status |
|-------|-------|----------|----------|--------|
| S17-1 | Fix Booking Price Display | High | 3 hrs | Done |
| S17-2 | Wire Admin Dashboard to Real API | High | 4 hrs | Done |
| S17-3 | Wire Admin Invitations Page | High | 6 hrs | Done |
| S17-4 | Build Admin Facilities List Page | Medium | 5 hrs | Pending |
| S17-5 | E2E Test: Facility Onboarding | Critical | 2 hrs | Pending |
| S17-6 | E2E Test: Player Booking | Critical | 2 hrs | Pending |
| S17-7 | E2E Test: Payment Recording | High | 1 hr | Pending |
| S17-8 | Fix Coach Dashboard Mock Data | Medium | 4 hrs | Pending |

**Total Estimate**: 27 hours

---

## S17-1: Fix Booking Price Display

**Type**: Bug
**Priority**: High
**Estimate**: 3 hours

### Problem Statement
The booking confirmation page (/book/confirm) displays a hardcoded "$30" price instead of calculating the actual price based on resource pricing rules.

### Scope

**In Scope**:
- Display real calculated price on booking confirmation
- Show price breakdown (base price, duration, any peak surcharges)
- Handle case where no pricing rules exist (show "Contact facility")

**Out of Scope**:
- Resource pricing UI (Sprint 18)
- Stripe payment integration (Sprint 23)

### Acceptance Criteria
- [ ] Price shown matches API calculation from GET /resources/:id/calculate-price
- [ ] Price updates when time slot selection changes
- [ ] If no pricing configured, shows "Contact facility for pricing"
- [ ] Price breakdown shown (e.g., "1 hour × $30/hr = $30")
- [ ] Currency symbol matches facility settings

### Technical Details

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/resources/:id/calculate-price?startTime=X&endTime=Y | Calculate booking price |
| GET | /api/v1/resources/:id/pricing | Get pricing rules |

**Files to Modify**:
- `web/src/app/book/confirm/page.tsx` - Line ~233 has hardcoded "$30"

### Labels
bug, priority-high, sprint-17

---

## S17-2: Wire Admin Dashboard to Real API

**Type**: Bug
**Priority**: High
**Estimate**: 4 hours

### Problem Statement
The platform admin dashboard (/admin) displays hardcoded statistics instead of real data from the API.

### Scope

**In Scope**:
- Fetch real facility count from API
- Fetch real active/suspended facility counts
- Fetch real pending invitation count
- Display loading state while fetching
- Handle error state if API fails

**Out of Scope**:
- Facility list page (separate issue)
- Invitation management page (separate issue)
- Analytics/charts (Sprint 22)

### Acceptance Criteria
- [ ] Total facilities count from GET /admin/facilities
- [ ] Active facilities count (status=ACTIVE filter)
- [ ] Suspended facilities count (status=SUSPENDED filter)
- [ ] Pending invitations count from GET /admin/invitations?status=PENDING
- [ ] Loading spinner while fetching
- [ ] Error message if API fails
- [ ] Numbers update on page refresh

### Technical Details

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/facilities | List all facilities |
| GET | /api/v1/admin/invitations?status=PENDING | Get pending invitations |

**Files to Modify**:
- `web/src/app/admin/page.tsx` - Lines 12-54 have hardcoded stats

**New Hooks Needed**:
- `useAdminStats()` - Fetch dashboard statistics

### Labels
bug, priority-high, sprint-17

---

## S17-3: Wire Admin Invitations Page to Real API

**Type**: Bug
**Priority**: High
**Estimate**: 6 hours

### Problem Statement
The admin invitations management page (/admin/invitations) exists but is not wired to the real API endpoints.

### Scope

**In Scope**:
- List all invitations with status filters
- Create new invitation (FACILITY_OWNER type)
- Resend pending invitation
- Revoke pending invitation
- Delete expired/revoked invitations
- Show invitation status badges

**Out of Scope**:
- Bulk operations
- Email preview
- Invitation analytics

### Acceptance Criteria
- [ ] List shows all invitations from GET /admin/invitations
- [ ] Filter by status (PENDING, ACCEPTED, EXPIRED, REVOKED)
- [ ] Create invitation form sends POST /admin/invitations
- [ ] Resend button calls POST /admin/invitations/:id/resend
- [ ] Revoke button calls POST /admin/invitations/:id/revoke
- [ ] Delete button calls DELETE /admin/invitations/:id
- [ ] Success/error toasts shown for actions
- [ ] Table refreshes after actions

### Technical Details

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/invitations | List invitations |
| POST | /api/v1/admin/invitations | Create invitation |
| POST | /api/v1/admin/invitations/:id/resend | Resend invitation |
| POST | /api/v1/admin/invitations/:id/revoke | Revoke invitation |
| DELETE | /api/v1/admin/invitations/:id | Delete invitation |

**New Hooks Needed**:
- `useInvitations()` - List invitations
- `useCreateInvitation()` - Create mutation
- `useResendInvitation()` - Resend mutation
- `useRevokeInvitation()` - Revoke mutation
- `useDeleteInvitation()` - Delete mutation

### Labels
bug, priority-high, sprint-17

---

## S17-4: Build Admin Facilities List Page

**Type**: Feature
**Priority**: Medium
**Estimate**: 5 hours

### Problem Statement
Platform admins need to view all facilities on the platform, see their status, subscription tier, and take actions.

### Scope

**In Scope**:
- List all facilities with pagination
- Show: name, status, tier, owner email, created date
- Filter by status (ACTIVE, DRAFT, SUSPENDED)
- Search by facility name
- Link to facility detail page
- Quick actions: suspend/activate

**Out of Scope**:
- Bulk operations
- Export to CSV
- Facility editing

### Acceptance Criteria
- [ ] Table shows all facilities from GET /admin/facilities
- [ ] Pagination works (10 per page)
- [ ] Status filter works
- [ ] Search by name works
- [ ] Click row → goes to /admin/facilities/[id]
- [ ] Suspend button → confirmation → POST /admin/facilities/suspend
- [ ] Activate button → POST /admin/facilities/activate
- [ ] Status badge colors: ACTIVE=green, DRAFT=yellow, SUSPENDED=red

### Technical Details

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/facilities?skip=0&take=10&status=X | List facilities |
| POST | /api/v1/admin/facilities/suspend | Suspend facility |
| POST | /api/v1/admin/facilities/activate | Activate facility |

**Web Screens**:
| Route | Component | Change Required |
|-------|-----------|-----------------|
| /admin/facilities | page.tsx | Create new page |

### Labels
feature, priority-medium, sprint-17

---

## S17-5: E2E Test - Facility Onboarding Flow

**Type**: Test
**Priority**: Critical
**Estimate**: 2 hours

### Problem Statement
Verify the complete facility onboarding flow works without errors.

### Test Scenario
1. Platform admin creates invitation for new facility owner
2. New user receives invitation email (verify in Resend dashboard)
3. User clicks link, signs up via Clerk
4. User completes onboarding wizard:
   - Facility info (name, address, sport types)
   - Resources (add 2-3 resources)
   - Operating hours
   - Tier selection
   - Review & publish
5. Facility appears in facility admin dashboard
6. Facility appears on public listing

### Acceptance Criteria
- [ ] Invitation email received within 1 minute
- [ ] Sign-up flow completes without errors
- [ ] All wizard steps complete without errors
- [ ] Resources are saved (not empty)
- [ ] Operating hours are saved
- [ ] Subscription is created with correct tier
- [ ] Facility status changes from DRAFT to ACTIVE
- [ ] Facility visible at /facility/[slug]
- [ ] Facility admin can access /facility dashboard

### Test Data
```
Facility: Test Cricket Academy
Email: test-owner-[timestamp]@example.com
Sport: Cricket
Resources: Net 1 (CRICKET_LANE), Net 2 (CRICKET_LANE)
Hours: Mon-Fri 8am-9pm, Sat 9am-6pm, Sun closed
Tier: STARTER
```

### Pass/Fail Criteria
- **PASS**: All acceptance criteria met, no console errors
- **FAIL**: Any step fails, any 4xx/5xx errors, any console errors

### Labels
test, priority-critical, sprint-17

---

## S17-6: E2E Test - Player Booking Flow

**Type**: Test
**Priority**: Critical
**Estimate**: 2 hours

### Problem Statement
Verify the complete player booking flow works without errors.

### Test Scenario
1. Player visits public facility page (/facility/[slug])
2. Player views availability grid for a date
3. Player selects available time slot
4. Player clicks "Book" → creates hold
5. Player sees confirmation page with:
   - Booking details
   - Real price (not hardcoded)
   - Payment options
6. Player confirms booking
7. Player receives confirmation email
8. Booking appears in player dashboard (/player)
9. Booking appears in facility admin bookings (/facility/bookings)

### Acceptance Criteria
- [ ] Facility page loads with real data
- [ ] Availability grid shows correct slots
- [ ] Unavailable slots are not clickable
- [ ] Hold created successfully (5-min timer starts)
- [ ] Confirmation page shows real price
- [ ] Booking confirmed without errors
- [ ] Confirmation email received
- [ ] Booking in player dashboard
- [ ] Booking in facility admin view

### Test Data
```
Facility: Use existing test facility
Player: test-player-[timestamp]@example.com
Resource: First available resource
Date: Tomorrow
Time: First available slot
```

### Pass/Fail Criteria
- **PASS**: Booking confirmed, appears in both dashboards
- **FAIL**: Any step fails, hold expires, wrong price shown

### Labels
test, priority-critical, sprint-17

---

## S17-7: E2E Test - Payment Recording Flow

**Type**: Test
**Priority**: High
**Estimate**: 1 hour

### Problem Statement
Verify the complete offline payment flow works.

### Test Scenario
1. Facility admin goes to /facility/payments
2. Admin clicks "Record Payment"
3. Admin fills form:
   - Selects booking (or enters player)
   - Payment method: CASH
   - Amount: matches booking price
   - Reference: "Cash payment test"
4. Admin submits → payment created with RECORDED status
5. Admin sees payment in list
6. Admin clicks "Verify" → payment status = VERIFIED
7. Payment appears linked to booking

### Acceptance Criteria
- [ ] Record payment form works
- [ ] All payment methods available (CASH, CHECK, VENMO, ZELLE, BANK_TRANSFER)
- [ ] Payment saved with RECORDED status
- [ ] Payment appears in list immediately
- [ ] Verify action changes status to VERIFIED
- [ ] Void action changes status to VOID
- [ ] Payment total shown on booking detail

### Test Data
```
Booking: Use booking from S17-6
Payment method: CASH
Amount: Match booking price
Reference: "E2E Test Payment"
```

### Pass/Fail Criteria
- **PASS**: Payment recorded, verified, linked to booking
- **FAIL**: Any step fails, status not updated

### Labels
test, priority-high, sprint-17

---

## S17-8: Fix Coach Dashboard Mock Data

**Type**: Bug
**Priority**: Medium
**Estimate**: 4 hours

### Problem Statement
The coach dashboard (/coach) displays hardcoded statistics and mock schedule data instead of real data from the API.

### Scope

**In Scope**:
- Fetch real session count for current week
- Fetch real student count
- Fetch real upcoming sessions (next 3)
- Fetch real students list (top 5)

**Out of Scope**:
- Revenue tracking (not implemented)
- Session analytics (Sprint 22)

### Acceptance Criteria
- [ ] "Sessions This Week" shows real count from API
- [ ] "Active Students" shows real count from API
- [ ] "Upcoming Sessions" list shows real data
- [ ] "Your Students" list shows real data
- [ ] Loading states shown while fetching
- [ ] Empty states shown if no data

### Technical Details

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/coaches/sessions?startDate=X&endDate=Y | Get sessions in range |
| GET | /api/v1/coaches/students | Get coach's students |

**Files to Modify**:
- `web/src/app/(dashboard)/coach/page.tsx`

### Labels
bug, priority-medium, sprint-17

---

## Sprint 17 Definition of Done

- [ ] All 8 issues completed
- [ ] All 3 E2E tests pass
- [ ] No hardcoded mock data in any dashboard
- [ ] All prices shown are real (not hardcoded)
- [ ] Zero console errors in production
- [ ] SPRINT_TRACKER.md updated

---

## QA Checklist

### Before Sprint Demo:
- [ ] Create new facility via onboarding (end-to-end)
- [ ] Add 3 resources with different types
- [ ] Set operating hours
- [ ] Publish facility
- [ ] As player: browse facility, book a slot
- [ ] As admin: see booking, record payment, verify payment
- [ ] As player: see booking in "my bookings"
- [ ] As coach: see real data in dashboard
- [ ] As platform admin: see real facility/invitation counts

---

*Created: January 15, 2026*
