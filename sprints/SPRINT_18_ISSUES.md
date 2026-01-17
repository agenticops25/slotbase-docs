# Sprint 18: Resource Management UI

**Sprint Goal**: Build the web UI for managing resource pricing, availability schedules, and display ordering. Enable facility admins to fully configure their resources.

**Start Date**: January 18, 2026
**Duration**: 1 week

---

## Issue Summary

| Issue | Title | Priority | Estimate | Status |
|-------|-------|----------|----------|--------|
| S18-1 | Resource Pricing UI | High | 6 hrs | Pending |
| S18-2 | Resource Availability Schedule UI | High | 5 hrs | Pending |
| S18-3 | Resource Reordering (Drag & Drop) | Medium | 4 hrs | Pending |
| S18-4 | Resource Detail Page | Medium | 3 hrs | Pending |
| S18-5 | Pricing Display on Booking Grid | Medium | 2 hrs | Pending |
| S18-6 | E2E Test: Resource Management | High | 2 hrs | Pending |

**Total Estimate**: 22 hours

---

## S18-1: Resource Pricing UI

**Type**: Feature
**Priority**: High
**Estimate**: 6 hours

### Problem Statement
Facility admins cannot set pricing for their resources through the web UI. The API supports complex time-based and day-type pricing, but there's no interface to manage it.

### Scope

**In Scope**:
- View existing pricing rules for a resource
- Add new pricing rules (day type, time range, price per hour)
- Edit existing pricing rules
- Delete pricing rules
- Support for day types: WEEKDAY, WEEKEND, HOLIDAY, PEAK, OFF_PEAK
- Time-based pricing (different rates for different hours)
- Currency display (default USD)

**Out of Scope**:
- Multi-currency support (future)
- Bulk pricing import/export
- Pricing templates

### Acceptance Criteria
- [ ] Pricing tab/section visible on resource detail/edit page
- [ ] List shows all pricing rules with day type, time range, rate
- [ ] "Add Pricing Rule" button opens form
- [ ] Form includes: Day Type dropdown, Start Time, End Time, Price per Hour
- [ ] Can leave time empty for "all day" pricing
- [ ] Edit button on each rule opens edit form
- [ ] Delete button with confirmation
- [ ] Changes saved via API (POST /resources/:id/pricing)
- [ ] Success/error feedback shown

### Technical Details

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/resources/:id/pricing | Get pricing rules |
| POST | /api/v1/resources/:id/pricing | Set all pricing rules |
| POST | /api/v1/resources/:id/pricing/rule | Add single rule |
| DELETE | /api/v1/resources/pricing/:pricingId | Delete rule |

**New Hooks Needed**:
- `useResourcePricing(resourceId)` - Fetch pricing rules
- `useSetResourcePricing()` - Set pricing rules mutation
- `useAddPricingRule()` - Add single rule mutation
- `useDeletePricingRule()` - Delete rule mutation

**UI Components**:
- `PricingRuleList` - Display pricing rules table
- `PricingRuleForm` - Add/edit pricing rule form

**Files to Modify/Create**:
- `web/src/lib/api/hooks/use-facilities.ts` - Add pricing hooks
- `web/src/app/(dashboard)/facility/resources/[id]/page.tsx` - Resource detail page
- `web/src/components/resource/pricing-manager.tsx` - New component

### Labels
feature, priority-high, sprint-18

---

## S18-2: Resource Availability Schedule UI

**Type**: Feature
**Priority**: High
**Estimate**: 5 hours

### Problem Statement
Resources inherit facility operating hours by default, but admins may want to set resource-specific availability (e.g., a court available only certain days or hours).

### Scope

**In Scope**:
- View resource-specific availability schedule
- Toggle "Use facility hours" vs "Custom hours"
- Set custom hours per day of week
- Set closed days
- Visual schedule display (week grid)

**Out of Scope**:
- Recurring closures/maintenance schedules
- Seasonal availability
- Integration with booking blocks

### Acceptance Criteria
- [ ] Availability tab/section on resource detail page
- [ ] Toggle: "Use Facility Hours" (default) vs "Custom Schedule"
- [ ] When custom: show day-by-day schedule editor
- [ ] Each day: Open/Closed toggle + Open Time + Close Time
- [ ] Visual preview of weekly schedule
- [ ] Save changes via API
- [ ] Reset to facility hours button

### Technical Details

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/resources/:id/availability-schedule | Get resource schedule |
| POST | /api/v1/resources/:id/availability-schedule | Set custom schedule |
| DELETE | /api/v1/resources/:id/availability-schedule | Reset to facility hours |

**Note**: Check if these endpoints exist. May need to add to API if not.

**UI Components**:
- `AvailabilityScheduleEditor` - Week schedule editor
- `DayScheduleRow` - Single day editor row

**Files to Modify/Create**:
- `web/src/components/resource/availability-schedule.tsx` - New component
- `web/src/app/(dashboard)/facility/resources/[id]/page.tsx` - Add tab

### Labels
feature, priority-high, sprint-18

---

## S18-3: Resource Reordering (Drag & Drop)

**Type**: Feature
**Priority**: Medium
**Estimate**: 4 hours

### Problem Statement
Resources display in creation order. Facility admins want to control the display order (e.g., show most popular court first).

### Scope

**In Scope**:
- Drag-and-drop reordering on resources list page
- Save new order via API
- Display order reflected in public booking grid
- Visual feedback during drag

**Out of Scope**:
- Keyboard-only reordering (accessibility)
- Grouping/categories

### Acceptance Criteria
- [ ] Resources list shows drag handles
- [ ] Can drag resource cards to reorder
- [ ] Visual feedback shows drop target
- [ ] Order saved automatically on drop
- [ ] New order persists after page refresh
- [ ] Public availability grid uses same order

### Technical Details

**Libraries**:
- `@dnd-kit/core` and `@dnd-kit/sortable` for drag-and-drop

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| PATCH | /api/v1/resources/reorder | Update display order |

**Request Body**:
```json
{
  "facilityId": "...",
  "resourceIds": ["id1", "id2", "id3"]
}
```

**Files to Modify**:
- `web/src/app/(dashboard)/facility/resources/page.tsx` - Add DnD
- `web/src/lib/api/hooks/use-facilities.ts` - Add reorder mutation

### Labels
feature, priority-medium, sprint-18

---

## S18-4: Resource Detail Page

**Type**: Feature
**Priority**: Medium
**Estimate**: 3 hours

### Problem Statement
Currently resources are edited via a modal. We need a full detail page with tabs for pricing, availability, and settings.

### Scope

**In Scope**:
- Dedicated resource detail page (/facility/resources/[id])
- Tabs: Overview, Pricing, Availability, Settings
- Overview: Basic info, status, type, bookings count
- Breadcrumb navigation
- Quick actions (edit, delete, change status)

**Out of Scope**:
- Resource analytics/reports
- Booking history view

### Acceptance Criteria
- [ ] Route /facility/resources/[id] works
- [ ] Page loads resource data from API
- [ ] Tab navigation between sections
- [ ] Overview shows: name, type, status, display order
- [ ] Edit button opens inline edit or modal
- [ ] Delete button with confirmation
- [ ] Back button returns to resources list
- [ ] 404 handling for invalid resource ID

### Technical Details

**Files to Create**:
- `web/src/app/(dashboard)/facility/resources/[id]/page.tsx`
- `web/src/app/(dashboard)/facility/resources/[id]/layout.tsx` (optional)

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/resources/:id | Get resource details |

### Labels
feature, priority-medium, sprint-18

---

## S18-5: Pricing Display on Booking Grid

**Type**: Enhancement
**Priority**: Medium
**Estimate**: 2 hours

### Problem Statement
The public booking grid shows prices from availability API, but we should verify prices match the pricing rules and display clearly.

### Scope

**In Scope**:
- Verify pricing calculation matches pricing rules
- Show "from $X/hr" on each resource column header
- Tooltip showing pricing details on hover
- Handle resources with no pricing set

**Out of Scope**:
- Peak pricing indicators
- Dynamic pricing based on demand

### Acceptance Criteria
- [ ] Each resource column shows base rate
- [ ] Slots show actual calculated price
- [ ] If no pricing rules, show "Contact for pricing"
- [ ] Price updates when changing date (weekday vs weekend)
- [ ] Tooltip on price shows breakdown

### Technical Details

**Files to Modify**:
- `web/src/components/booking/availability-grid.tsx`

### Labels
enhancement, priority-medium, sprint-18

---

## S18-6: E2E Test: Resource Management

**Type**: Test
**Priority**: High
**Estimate**: 2 hours

### Problem Statement
Verify the complete resource management flow works end-to-end.

### Test Scenario
1. Facility admin navigates to /facility/resources
2. Admin creates new resource
3. Admin opens resource detail page
4. Admin adds pricing rules (weekday, weekend)
5. Admin sets custom availability schedule
6. Admin reorders resources
7. Player sees updated pricing on booking grid

### Acceptance Criteria
- [ ] Create resource flow works
- [ ] Pricing rules can be added/edited/deleted
- [ ] Availability schedule saves correctly
- [ ] Reordering persists
- [ ] Public grid reflects changes

### Test Data
```
Resource: "Court 3 - Premium"
Type: TENNIS_COURT
Pricing:
  - WEEKDAY: $30/hr
  - WEEKEND: $45/hr
  - PEAK (5pm-9pm): $50/hr
Availability: Mon-Sat 7am-10pm, Sun closed
```

### Labels
test, priority-high, sprint-18

---

## Sprint 18 Definition of Done

- [ ] All 6 issues completed
- [ ] Pricing UI fully functional
- [ ] Availability schedule UI working
- [ ] Drag-and-drop reordering works
- [ ] Resource detail page with tabs
- [ ] E2E tests pass
- [ ] No console errors in production
- [ ] SPRINT_TRACKER.md updated
- [ ] FEATURE_PARITY_DASHBOARD.md updated

---

## Dependencies

### API Endpoints to Verify/Add
Before starting, verify these endpoints exist in the API:

1. **Pricing** (confirmed existing):
   - GET /resources/:id/pricing ✅
   - POST /resources/:id/pricing ✅
   - DELETE /resources/pricing/:id ✅

2. **Availability Schedule** (need to verify):
   - GET /resources/:id/availability-schedule
   - POST /resources/:id/availability-schedule
   - DELETE /resources/:id/availability-schedule

3. **Reordering** (need to verify):
   - PATCH /resources/reorder

### Packages to Install
```bash
pnpm add @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

---

## QA Checklist

### Before Sprint Demo:
- [ ] Create resource with all fields
- [ ] Add pricing rules (multiple day types)
- [ ] Set custom availability schedule
- [ ] Reorder resources via drag-and-drop
- [ ] Verify changes appear on public booking grid
- [ ] Test on mobile viewport
- [ ] No console errors

---

*Created: January 17, 2026*
