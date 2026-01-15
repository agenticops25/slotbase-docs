# In-App Help System Design

> Feature for user documentation, onboarding, and contextual guidance

**GitHub Issue**: https://github.com/agenticops25/slotbase-api/issues/12

---

## Overview

A floating help widget providing:
- Role-based guides
- Changelog / What's New
- Keyboard shortcuts
- Contextual tooltips
- Video tutorials
- Search across docs

---

## UI Design

### Help Widget (Floating Icon)

```
┌─────────────────────────────────────┐
│                                     │
│          App Content                │
│                                     │
│                                     │
│                                     │
│                         ┌───┐       │
│                         │ ? │       │
│                         └───┘       │
└─────────────────────────────────────┘
```

### Help Panel (Expanded)

```
┌─────────────────────────────────────┐
│                           ┌────────┐│
│                           │ Help   ││
│                           ├────────┤│
│                           │🔍Search││
│                           ├────────┤│
│                           │📚Guides││
│                           │  Admin ││
│                           │  Coach ││
│                           │  Player││
│                           │  Parent││
│                           ├────────┤│
│                           │🆕New   ││
│                           │  v1.2  ││
│                           │  v1.1  ││
│                           ├────────┤│
│                           │⌨️Keys  ││
│                           ├────────┤│
│                           │🎥Videos││
│                           ├────────┤│
│                           │💬Help  ││
│                           └────────┘│
└─────────────────────────────────────┘
```

---

## Role-Based Guide Content

### Facility Admin
| Article | Description | Priority |
|---------|-------------|----------|
| Getting Started | First-time setup checklist | P0 |
| Managing Resources | Add/edit courts, nets, machines | P0 |
| Setting Hours | Operating hours and holidays | P0 |
| Recording Payments | Cash, check, Venmo, Zelle | P0 |
| Inviting Team | Add staff, coaches | P1 |
| Viewing Reports | Utilization, revenue (future) | P2 |

### Coach
| Article | Description | Priority |
|---------|-------------|----------|
| Getting Started | Profile setup | P0 |
| Managing Schedule | View and manage bookings | P0 |
| Blocking Time | Reserve slots for lessons | P0 |
| Student Management | View students, notes | P1 |

### Player
| Article | Description | Priority |
|---------|-------------|----------|
| Getting Started | Account setup | P0 |
| Booking a Session | Find facility, select slot, confirm | P0 |
| Managing Bookings | View, cancel, reschedule | P0 |
| Payment Options | Offline payment methods | P1 |

### Parent/Guardian
| Article | Description | Priority |
|---------|-------------|----------|
| Getting Started | Link to children | P0 |
| Booking for Children | Book on behalf of child | P0 |
| Family Billing | Manage payments (future) | P2 |

---

## Changelog Format

```json
{
  "releases": [
    {
      "version": "1.2.0",
      "date": "2026-01-20",
      "title": "Player Booking Flow",
      "highlights": [
        "Players can now book sessions directly",
        "Email confirmations for bookings",
        "Improved availability grid"
      ],
      "details": "Sprint 16 release notes..."
    }
  ]
}
```

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `?` | Open help panel |
| `Cmd/Ctrl + K` | Open search |
| `Cmd/Ctrl + B` | New booking |
| `Esc` | Close panel/modal |
| `←` / `→` | Navigate dates (in calendar) |

---

## Contextual Tooltips

Add `data-help` attribute to complex UI elements:

```tsx
<Button data-help="booking-confirm">
  Confirm Booking
</Button>
```

Tooltip content stored in JSON:
```json
{
  "booking-confirm": {
    "title": "Confirm Booking",
    "content": "This will reserve your slot and send a confirmation email.",
    "link": "/help/player/booking-session"
  }
}
```

---

## Video Tutorials

| Topic | Duration | Role |
|-------|----------|------|
| Platform Overview | 2 min | All |
| Admin: Setting Up Your Facility | 5 min | Admin |
| Admin: Recording Payments | 3 min | Admin |
| Coach: Managing Your Schedule | 3 min | Coach |
| Player: Booking Your First Session | 2 min | Player |

---

## Technical Implementation

### Component Structure

```
web/src/components/help/
├── help-provider.tsx       # Context for help state
├── help-widget.tsx         # Floating (?) button
├── help-panel.tsx          # Slide-out panel
├── help-search.tsx         # Search input + results
├── help-guides.tsx         # Guide list by role
├── help-guide-viewer.tsx   # Single guide article
├── help-changelog.tsx      # Release feed
├── help-shortcuts.tsx      # Keyboard shortcuts modal
├── help-tooltip.tsx        # Contextual tooltip wrapper
└── help-video.tsx          # Video player embed
```

### Data Sources

```
web/src/content/help/
├── guides/
│   ├── admin/
│   │   ├── getting-started.mdx
│   │   └── ...
│   ├── coach/
│   ├── player/
│   └── parent/
├── changelog.json
├── shortcuts.json
└── tooltips.json
```

### Build-Time Generation

```bash
# scripts/generate-help.ts
# Runs at build time to:
# 1. Convert MDX guides to JSON
# 2. Fetch GitHub releases for changelog
# 3. Extract JSDoc tooltips from components
```

---

## Implementation Phases

### Phase 1: Foundation (Sprint 17)
- [ ] HelpProvider context
- [ ] HelpWidget floating button
- [ ] HelpPanel slide-out
- [ ] Keyboard shortcut listener
- [ ] Basic shortcuts display

### Phase 2: Content (Sprint 18)
- [ ] MDX guide processing
- [ ] Admin guides (6 articles)
- [ ] Coach guides (4 articles)
- [ ] Player guides (3 articles)
- [ ] Parent guides (2 articles)
- [ ] Guide viewer component

### Phase 3: Changelog & Tooltips (Sprint 19)
- [ ] Changelog component
- [ ] Release fetch script
- [ ] Tooltip component
- [ ] Add tooltips to key UI elements
- [ ] Tooltip data extraction

### Phase 4: Video & Polish (Sprint 20)
- [ ] Record video tutorials
- [ ] Video embed component
- [ ] Search implementation
- [ ] Animations polish
- [ ] Mobile responsive

---

## Sprint 16 Documentation Tasks

After completing Sprint 16, document:

| Feature | Guide | Role |
|---------|-------|------|
| Public facility page | "Finding a Facility" | Player |
| Availability grid | "Viewing Available Slots" | Player |
| Booking flow | "Booking Your First Session" | Player |
| Email confirmations | "Booking Confirmations" | Player |
| Player dashboard | "Managing Your Bookings" | Player |
| Tier selection | "Choosing Your Plan" | Admin |

---

## Questions to Resolve

1. **Video hosting**: YouTube, Loom, or self-hosted?
2. **Search**: Client-side (Fuse.js) or server-side?
3. **Analytics**: Track which help articles are most viewed?
4. **Feedback**: Allow users to rate articles helpful/not helpful?
5. **Support chat**: Integrate Intercom/Crisp or email-only?

---

*Created: January 15, 2026*
*Status: Planning*
