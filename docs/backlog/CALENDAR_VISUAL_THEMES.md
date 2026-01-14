# Backlog: Calendar Visual Themes & Resource Identity

**Priority:** High (Product Differentiator)
**Sprint Target:** Sprint 17
**Estimated Effort:** Large (2-3 weeks)
**Dependencies:** Resource management, Facility settings

---

## Executive Summary

This feature transforms the booking calendar from a functional grid into an intuitive, branded visual experience. Facility admins can customize colors, icons, and themes at multiple levels (facility → sport → resource → slot), making it instantly clear what type of resource a user is booking without reading text labels.

**Why this matters:**
- Reduces booking mistakes (wrong court, wrong lane)
- Essential for multi-sport facilities (cricket + tennis + swimming)
- Makes the product accessible to non-technical users (parents, kids, staff)
- Creates visual brand identity for each facility

---

## Feature Overview (Simple Terms)

Imagine walking into a sports facility:
- Cricket nets have green turf backgrounds
- Tennis courts are blue
- Swimming lanes show water patterns
- Each lane has a small icon showing its features (bowling machine, shallow end)

This feature brings that real-world visual recognition to the digital calendar.

**For Facility Admins:**
- Choose colors for different sports/resource types
- Add icons to show resource features
- Create a branded booking experience

**For Players/Parents:**
- Instantly see what type of resource they're booking
- Identify special features (coaching, machine, beginner-friendly)
- Fewer mistakes, better experience

---

## Data Model Additions

### 1. Sport Type Configuration (New Table)

```prisma
model SportType {
  id          String   @id @default(uuid())
  facilityId  String
  name        String   // "Cricket", "Tennis", "Swimming"
  slug        String   // "cricket", "tennis", "swimming"

  // Visual Theme
  colorPrimary    String?  // "#22C55E" (green for cricket)
  colorSecondary  String?  // "#16A34A"
  colorBackground String?  // "#F0FDF4" (light green bg)
  iconName        String?  // "cricket-bat", "tennis-racket"

  // Metadata
  isActive    Boolean  @default(true)
  sortOrder   Int      @default(0)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  facility    Facility   @relation(fields: [facilityId], references: [id])
  resources   Resource[]

  @@unique([facilityId, slug])
  @@index([facilityId])
}
```

### 2. Resource Visual Metadata (Extend Existing)

```prisma
model Resource {
  // ... existing fields ...

  // Visual Identity (NEW)
  sportTypeId     String?
  colorOverride   String?       // Override sport type color
  iconOverride    String?       // Override sport type icon
  badgeText       String?       // "Lane 1", "Court A"
  badgeColor      String?       // Badge background color
  thumbnailUrl    String?       // Small image (optional)

  // Feature Flags (NEW)
  features        ResourceFeature[]

  sportType       SportType?    @relation(fields: [sportTypeId], references: [id])
}
```

### 3. Resource Features (New Table)

```prisma
model ResourceFeature {
  id          String   @id @default(uuid())
  resourceId  String
  featureKey  String   // "bowling_machine", "lighting", "shallow_end"
  label       String   // "Bowling Machine", "Night Lighting"
  iconName    String?  // "machine", "lightbulb"
  isActive    Boolean  @default(true)

  resource    Resource @relation(fields: [resourceId], references: [id])

  @@unique([resourceId, featureKey])
  @@index([resourceId])
}
```

### 4. Facility Theme Defaults (Extend Existing)

```prisma
model Facility {
  // ... existing fields ...

  // Theme Defaults (NEW)
  themeConfig     Json?    // Default colors, fonts, etc.

  sportTypes      SportType[]
}
```

### 5. Slot Feature Overrides (New Table)

```prisma
model SlotFeatureOverride {
  id          String   @id @default(uuid())
  resourceId  String
  dayOfWeek   Int?     // 0-6 (null = all days)
  startTime   String   // "09:00"
  endTime     String   // "12:00"

  // Feature overrides for this time range
  featureKey  String   // "coaching_only", "beginner"
  label       String   // "Coaching Session"
  iconName    String?
  colorOverride String?

  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())

  resource    Resource @relation(fields: [resourceId], references: [id])

  @@index([resourceId, dayOfWeek])
}
```

---

## Theme Inheritance Model

Themes cascade with specificity (most specific wins):

```
Facility Defaults
    ↓
Sport Type Theme (cricket, tennis, swimming)
    ↓
Resource Override (Court 1 has special color)
    ↓
Slot Override (9-12 AM is coaching-only)
```

### Resolution Logic

```typescript
function resolveSlotTheme(
  facility: Facility,
  resource: Resource,
  slot: TimeSlot
): SlotTheme {
  // Start with facility defaults
  let theme = facility.themeConfig?.defaults || DEFAULT_THEME;

  // Apply sport type theme
  if (resource.sportType) {
    theme = {
      ...theme,
      colorPrimary: resource.sportType.colorPrimary || theme.colorPrimary,
      colorSecondary: resource.sportType.colorSecondary || theme.colorSecondary,
      colorBackground: resource.sportType.colorBackground || theme.colorBackground,
      iconName: resource.sportType.iconName || theme.iconName,
    };
  }

  // Apply resource overrides
  if (resource.colorOverride) {
    theme.colorPrimary = resource.colorOverride;
  }
  if (resource.iconOverride) {
    theme.iconName = resource.iconOverride;
  }

  // Apply slot-specific overrides
  const slotOverride = findSlotOverride(resource.id, slot);
  if (slotOverride) {
    theme = {
      ...theme,
      colorPrimary: slotOverride.colorOverride || theme.colorPrimary,
      iconName: slotOverride.iconName || theme.iconName,
      badge: slotOverride.label,
    };
  }

  return theme;
}
```

---

## Calendar Rendering Logic

### Slot Component Structure

```tsx
interface SlotTheme {
  colorPrimary: string;      // Border, text
  colorSecondary: string;    // Hover states
  colorBackground: string;   // Slot background
  iconName?: string;         // Sport/feature icon
  badge?: string;            // "Coaching", "Machine"
  features: FeatureBadge[];  // Small feature indicators
}

function CalendarSlot({ slot, theme, booking }: SlotProps) {
  return (
    <div
      className="calendar-slot"
      style={{
        backgroundColor: theme.colorBackground,
        borderColor: booking ? theme.colorPrimary : 'transparent',
      }}
    >
      {/* Sport Icon (top-left) */}
      {theme.iconName && (
        <Icon name={theme.iconName} className="slot-sport-icon" />
      )}

      {/* Feature Badges (top-right) */}
      <div className="slot-features">
        {theme.features.map(f => (
          <FeatureBadge key={f.key} icon={f.icon} tooltip={f.label} />
        ))}
      </div>

      {/* Booking Info or Empty Slot */}
      {booking ? (
        <BookingCard booking={booking} theme={theme} />
      ) : (
        <EmptySlot theme={theme} />
      )}

      {/* Slot Badge (bottom) */}
      {theme.badge && (
        <span className="slot-badge" style={{ backgroundColor: theme.colorSecondary }}>
          {theme.badge}
        </span>
      )}
    </div>
  );
}
```

### Performance Optimizations

```typescript
// 1. Pre-compute themes for visible resources
const resourceThemes = useMemo(() => {
  return resources.reduce((acc, resource) => {
    acc[resource.id] = computeResourceTheme(facility, resource);
    return acc;
  }, {} as Record<string, ResourceTheme>);
}, [facility, resources]);

// 2. Use CSS custom properties for colors (no inline styles per slot)
const calendarStyles = useMemo(() => ({
  '--slot-color-cricket': sportTypes.cricket?.colorPrimary || '#22C55E',
  '--slot-color-tennis': sportTypes.tennis?.colorPrimary || '#3B82F6',
  '--slot-color-swimming': sportTypes.swimming?.colorPrimary || '#06B6D4',
}), [sportTypes]);

// 3. Use icon sprites or icon fonts (not individual SVG requests)
// 4. Lazy load thumbnails only when visible
// 5. Cache theme computations in React Query
```

---

## Admin UX Flow

### 1. Sport Types Setup (Settings → Sports)

```
┌─────────────────────────────────────────────────────────┐
│ Sports & Activities                                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ 🏏       │  │ 🎾       │  │ 🏊       │  [+ Add]     │
│  │ Cricket  │  │ Tennis   │  │ Swimming │              │
│  │ 3 lanes  │  │ 2 courts │  │ 4 lanes  │              │
│  │ ●●●      │  │ ●●       │  │ ●●●●     │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│                                                          │
│  [Edit Sport Theme]                                      │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Sport: Cricket                                   │    │
│  │                                                  │    │
│  │ Primary Color:    [████████] #22C55E            │    │
│  │ Background:       [████████] #F0FDF4            │    │
│  │ Icon:             [🏏 Cricket Bat ▼]            │    │
│  │                                                  │    │
│  │ Preview:                                         │    │
│  │ ┌─────────────────────────┐                     │    │
│  │ │ 🏏 Cricket Lane 1       │                     │    │
│  │ │ ░░░░░░░░░░░░░░░░░░░░░░░ │                     │    │
│  │ └─────────────────────────┘                     │    │
│  │                                                  │    │
│  │ [Cancel]                    [Save Theme]        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 2. Resource Visual Customization (Resources → Edit)

```
┌─────────────────────────────────────────────────────────┐
│ Edit Resource: Cricket Lane 1                            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│ Basic Info                                               │
│ ┌─────────────────────────────────────────────────┐     │
│ │ Name: [Cricket Lane 1        ]                   │     │
│ │ Sport: [Cricket ▼]  ← inherits cricket theme    │     │
│ └─────────────────────────────────────────────────┘     │
│                                                          │
│ Visual Customization (Optional)                          │
│ ┌─────────────────────────────────────────────────┐     │
│ │ □ Override sport color                           │     │
│ │   Color: [████████]                              │     │
│ │                                                  │     │
│ │ Badge Label: [Lane 1          ]                  │     │
│ │ Badge Color: [████████] #1E40AF                  │     │
│ └─────────────────────────────────────────────────┘     │
│                                                          │
│ Features                                                 │
│ ┌─────────────────────────────────────────────────┐     │
│ │ ☑ Bowling Machine Available                      │     │
│ │ ☐ Batting Only                                   │     │
│ │ ☑ Night Lighting                                 │     │
│ │ ☐ Coach Required                                 │     │
│ │                                                  │     │
│ │ [+ Add Custom Feature]                           │     │
│ └─────────────────────────────────────────────────┘     │
│                                                          │
│ Preview                                                  │
│ ┌─────────────────────────────────────────────────┐     │
│ │ 🏏 Cricket Lane 1              ⚡ 💡            │     │
│ │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │     │
│ │                           [Lane 1] [Machine]    │     │
│ └─────────────────────────────────────────────────┘     │
│                                                          │
│ [Cancel]                              [Save Resource]   │
└─────────────────────────────────────────────────────────┘
```

### 3. Slot Schedule Features (Resources → Schedule)

```
┌─────────────────────────────────────────────────────────┐
│ Cricket Lane 1 - Schedule Features                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│ Define time-specific features for this resource          │
│                                                          │
│ ┌─────────────────────────────────────────────────┐     │
│ │ Time Range      Days           Feature          │     │
│ ├─────────────────────────────────────────────────┤     │
│ │ 6:00 - 9:00 AM  Mon-Fri        Morning Practice │     │
│ │ 9:00 - 12:00    Sat-Sun        Coaching Only    │     │
│ │ 6:00 - 10:00 PM All Days       Night Session    │     │
│ │                                                  │     │
│ │ [+ Add Time Feature]                            │     │
│ └─────────────────────────────────────────────────┘     │
│                                                          │
│ Preview: Monday                                          │
│ ┌─────────────────────────────────────────────────┐     │
│ │ 6AM  [Morning Practice ░░░░░░░░░░░]             │     │
│ │ 9AM  [░░░░░░░░░░░░░░░░░░░░░░░░░░░░]             │     │
│ │ 6PM  [Night Session ░░░░░░░░░░░░░░] 💡          │     │
│ └─────────────────────────────────────────────────┘     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Player UX Behavior

### Calendar View

```
┌─────────────────────────────────────────────────────────┐
│ Book a Session                          Jan 15, 2026    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│ Filter: [All Sports ▼] [All Features ▼] [Available ▼]   │
│                                                          │
│         Cricket Lane 1    Cricket Lane 2    Tennis A    │
│         🏏 ⚡ 💡          🏏               🎾           │
│ ┌───────┬───────────────┬───────────────┬─────────────┐ │
│ │ 9 AM  │ ░░░░░░░░░░░░░ │ ░░░░░░░░░░░░░ │ ▓▓▓▓▓▓▓▓▓▓▓ │ │
│ │       │ [Available]   │ [Available]   │ [Booked]    │ │
│ ├───────┼───────────────┼───────────────┼─────────────┤ │
│ │ 10 AM │ ▓▓▓▓▓▓▓▓▓▓▓▓▓ │ ░░░░░░░░░░░░░ │ ░░░░░░░░░░░ │ │
│ │       │ [Coaching]    │ [Available]   │ [Available] │ │
│ ├───────┼───────────────┼───────────────┼─────────────┤ │
│ │ 11 AM │ ░░░░░░░░░░░░░ │ ░░░░░░░░░░░░░ │ ░░░░░░░░░░░ │ │
│ │       │ [Machine] ⚡  │ [Available]   │ [Available] │ │
│ └───────┴───────────────┴───────────────┴─────────────┘ │
│                                                          │
│ Legend: 🏏 Cricket  🎾 Tennis  ⚡ Machine  💡 Lights    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Feature Filtering

Players can filter by features:
- "Show only lanes with bowling machine"
- "Show only lanes with night lighting"
- "Show only coaching sessions"

```typescript
// API endpoint
GET /api/v1/resources?facilityId=xxx&features=bowling_machine,lighting

// Returns resources that have ALL specified features
```

---

## API Endpoints

### Sport Types

```
GET    /api/v1/facilities/:id/sport-types
POST   /api/v1/facilities/:id/sport-types
PATCH  /api/v1/sport-types/:id
DELETE /api/v1/sport-types/:id
```

### Resource Features

```
GET    /api/v1/resources/:id/features
POST   /api/v1/resources/:id/features
PATCH  /api/v1/resource-features/:id
DELETE /api/v1/resource-features/:id
```

### Slot Feature Overrides

```
GET    /api/v1/resources/:id/slot-overrides
POST   /api/v1/resources/:id/slot-overrides
PATCH  /api/v1/slot-overrides/:id
DELETE /api/v1/slot-overrides/:id
```

### Theme Resolution (Read-only)

```
GET /api/v1/facilities/:id/calendar-themes
// Returns pre-computed themes for all resources (cached)
```

---

## What Belongs Where

### Database (Persistent, User-Configurable)
- Sport type definitions per facility
- Resource-to-sport assignments
- Resource feature flags
- Slot feature overrides
- Custom colors and icon selections

### Configuration (Code Constants)
- Default color palette options
- Available icon set (icon names)
- Default theme values
- Feature key definitions (bowling_machine, lighting, etc.)

### Never Hardcode
- Sport names (facilities define their own)
- Color values (always configurable)
- Feature requirements (all optional)
- Visual styling tied to booking logic

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Performance: Too many colors/icons slow rendering | High | Use CSS variables, icon sprites, theme caching |
| Complexity: Admin overwhelmed by options | Medium | Progressive disclosure, sensible defaults |
| Inconsistency: Mobile vs web look different | Medium | Shared theme tokens, design system |
| Data migration: Existing resources need themes | Low | Default to facility theme, gradual migration |

---

## Future Extensions

1. **Theme Templates**: Pre-built themes for common sports (cricket, tennis, swimming)
2. **Seasonal Themes**: Different colors for summer/winter
3. **Player Preferences**: Remember player's preferred resource colors
4. **Accessibility Modes**: High contrast, color-blind friendly palettes
5. **White-label Theming**: Full facility branding (logo, fonts)
6. **Calendar Export**: Themed PDF/image export for schedules

---

## Implementation Phases

### Phase 1: Foundation (Sprint 17)
- [ ] Sport type model and CRUD
- [ ] Resource visual metadata fields
- [ ] Basic theme inheritance logic
- [ ] Admin sport type configuration UI

### Phase 2: Resource Customization (Sprint 18)
- [ ] Resource feature model
- [ ] Feature badges in calendar
- [ ] Resource edit UI with visual preview
- [ ] Feature filtering API

### Phase 3: Slot Features (Sprint 19)
- [ ] Slot override model
- [ ] Time-based feature display
- [ ] Schedule features admin UI
- [ ] Player calendar with full theming

### Phase 4: Polish (Sprint 20)
- [ ] Performance optimization
- [ ] Mobile calendar theming
- [ ] Theme templates
- [ ] Documentation and admin guide

---

## Success Metrics

- **Booking Error Rate**: Reduce wrong-resource bookings by 50%
- **Time to Book**: No increase in booking time (visual should help, not hinder)
- **Admin Adoption**: 80% of facilities configure at least sport types
- **Player Satisfaction**: Positive feedback on calendar usability

---

## References

- [Current Booking Calendar](/web/src/components/bookings/booking-calendar.tsx)
- [Resource Model](/api/prisma/schema.prisma)
- [Tailwind Color Palette](https://tailwindcss.com/docs/customizing-colors)
