# SlotBase API Design Specification

## Overview

SlotBase uses a RESTful API with WebSocket support for real-time features. This document defines the API structure, conventions, and endpoint specifications.

---

## API Conventions

### Base URL

```
Production: https://api.slotbase.io/v1
Staging:    https://api.staging.slotbase.io/v1
Local:      http://localhost:3000/v1
```

### Authentication

```http
Authorization: Bearer <jwt_token>
```

All authenticated endpoints require a valid JWT token from Clerk.

### Request/Response Format

- **Content-Type**: `application/json`
- **Date format**: ISO 8601 (`2024-01-15T10:30:00Z`)
- **Currency**: Amount in smallest unit (cents for USD)
- **Pagination**: Cursor-based
- **Rate Limiting**: Per-user and per-IP

### Response Envelope

```typescript
// Success response
interface ApiResponse<T> {
  success: true;
  data: T;
  meta?: {
    pagination?: PaginationMeta;
    requestId: string;
    timestamp: string;
  };
}

// Error response
interface ApiErrorResponse {
  success: false;
  error: {
    code: string;           // Machine-readable code
    message: string;        // Human-readable message
    details?: object;       // Additional error details
    field?: string;         // For validation errors
  };
  meta: {
    requestId: string;
    timestamp: string;
  };
}

// Pagination
interface PaginationMeta {
  cursor?: string;
  nextCursor?: string;
  prevCursor?: string;
  hasMore: boolean;
  total?: number;
}
```

### HTTP Status Codes

| Code | Usage |
|------|-------|
| 200 | Success |
| 201 | Created |
| 204 | No Content (delete success) |
| 400 | Bad Request (validation error) |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (not authorized) |
| 404 | Not Found |
| 409 | Conflict (e.g., double booking) |
| 422 | Unprocessable Entity |
| 429 | Too Many Requests |
| 500 | Internal Server Error |

---

## API Endpoints

### Identity & Users

```yaml
# Get current user
GET /users/me
Response:
  - id: string
  - email: string
  - firstName: string
  - lastName: string
  - phone: string?
  - avatarUrl: string?
  - roles: Role[]
  - createdAt: string

# Update current user
PATCH /users/me
Body:
  - firstName?: string
  - lastName?: string
  - phone?: string
  - avatarUrl?: string

# Get user by ID (admin only)
GET /users/:userId

# Get user's roles
GET /users/:userId/roles
Query:
  - scopeType?: 'platform' | 'organization' | 'facility'
  - scopeId?: string

# Assign role to user
POST /users/:userId/roles
Body:
  - role: string
  - scopeType: string
  - scopeId: string

# Remove role from user
DELETE /users/:userId/roles/:roleId
```

### Family & Parent-Child

```yaml
# Get family groups for current user
GET /families
Response:
  - families: FamilyGroup[]

# Create family group
POST /families
Body:
  - name: string

# Add member to family
POST /families/:familyId/members
Body:
  - userId: string
  - role: 'parent' | 'child' | 'guardian'
  - permissions: object

# Get children linked to current user
GET /users/me/children
Response:
  - children: ChildWithLink[]

# Link parent to child
POST /parent-child-links
Body:
  - parentUserId?: string (defaults to current user)
  - childUserId: string
  - relationship: 'parent' | 'guardian' | 'caregiver'
  - permissions: object

# Update link permissions
PATCH /parent-child-links/:linkId
Body:
  - canBook?: boolean
  - canCancel?: boolean
  - canPay?: boolean
  - canReceiveUpdates?: boolean
```

### Organizations & Facilities

```yaml
# List facilities (public)
GET /facilities
Query:
  - sportType?: string
  - lat?: number
  - lng?: number
  - radius?: number (km)
  - cursor?: string
  - limit?: number

# Get facility details
GET /facilities/:facilityId
Response:
  - id: string
  - name: string
  - slug: string
  - address: Address
  - timezone: string
  - sportTypes: string[]
  - operatingHours: OperatingHours[]
  - resources: Resource[]

# Create facility (authenticated)
POST /facilities
Body:
  - name: string
  - address: Address
  - timezone: string
  - sportTypes: string[]

# Update facility (admin only)
PATCH /facilities/:facilityId
Body:
  - name?: string
  - address?: Address
  - operatingHours?: OperatingHours[]

# Get facility settings (admin only)
GET /facilities/:facilityId/settings

# Update facility settings (admin only)
PATCH /facilities/:facilityId/settings
Body:
  - policies?: object
  - notifications?: object
  - payments?: object
```

### Resources (Courts/Lanes)

```yaml
# List resources for facility
GET /facilities/:facilityId/resources
Response:
  - resources: Resource[]

# Get resource details
GET /facilities/:facilityId/resources/:resourceId

# Create resource (admin only)
POST /facilities/:facilityId/resources
Body:
  - name: string
  - resourceType: string
  - attributes?: object

# Update resource (admin only)
PATCH /facilities/:facilityId/resources/:resourceId
Body:
  - name?: string
  - status?: string
  - attributes?: object

# Get resource availability
GET /facilities/:facilityId/resources/:resourceId/availability
Query:
  - date: string (YYYY-MM-DD)
  - days?: number (default 1, max 14)
Response:
  - availability: DayAvailability[]
    - date: string
    - slots: Slot[]
      - startTime: string
      - endTime: string
      - status: 'available' | 'booked' | 'blocked' | 'maintenance'
      - price: number
      - currency: string
      - bookingId?: string

# Get resource pricing
GET /facilities/:facilityId/resources/:resourceId/pricing
Response:
  - pricing: ResourcePricing[]

# Update resource pricing (admin only)
PUT /facilities/:facilityId/resources/:resourceId/pricing
Body:
  - pricing: ResourcePricing[]
```

### Bookings

```yaml
# List bookings
GET /bookings
Query:
  - facilityId?: string
  - resourceId?: string
  - status?: string
  - startDate?: string
  - endDate?: string
  - cursor?: string
  - limit?: number
Response:
  - bookings: Booking[]

# Get booking details
GET /bookings/:bookingId
Response:
  - id: string
  - facilityId: string
  - resourceId: string
  - resource: Resource
  - bookerId: string
  - booker: User
  - participants: Participant[]
  - startAt: string
  - endAt: string
  - status: string
  - bookingType: string
  - payment: BookingPayment
  - createdAt: string

# Create booking hold
POST /bookings/hold
Body:
  - facilityId: string
  - resourceId: string
  - startAt: string
  - endAt: string
  - participants?: string[] (user IDs)
  - bookingType?: string
  - forUserId?: string (booking on behalf of)
Response:
  - holdId: string
  - bookingId: string
  - expiresAt: string
  - price: number
  - currency: string
  - paymentIntent?: object (if online payment)

# Confirm booking (after payment)
POST /bookings/:bookingId/confirm
Body:
  - paymentIntentId?: string (for online)
  - paymentMethod?: 'online' | 'offline'
Response:
  - booking: Booking

# Cancel booking
POST /bookings/:bookingId/cancel
Body:
  - reason: string
Response:
  - booking: Booking
  - refund?: Refund

# Check in to booking
POST /bookings/:bookingId/check-in
Body:
  - userId?: string (defaults to current user)
  - method?: 'qr_code' | 'manual' | 'access_code'

# Add participant to booking
POST /bookings/:bookingId/participants
Body:
  - userId: string
  - role?: 'player' | 'guest'

# Remove participant from booking
DELETE /bookings/:bookingId/participants/:userId

# Get upcoming bookings for current user
GET /users/me/bookings/upcoming
Query:
  - limit?: number
```

### Recurring Bookings

```yaml
# Create recurring booking
POST /bookings/recurring
Body:
  - facilityId: string
  - resourceId: string
  - startTime: string (HH:MM)
  - endTime: string (HH:MM)
  - startDate: string (YYYY-MM-DD)
  - endDate?: string
  - recurrenceRule: string (RRULE)
  - participants?: string[]
Response:
  - recurringBookingId: string
  - bookings: Booking[]

# Get recurring booking details
GET /bookings/recurring/:recurringId

# Cancel recurring booking
POST /bookings/recurring/:recurringId/cancel
Body:
  - cancelFuture: boolean (only future or all)

# Add exception to recurring booking
POST /bookings/recurring/:recurringId/exceptions
Body:
  - date: string
  - action: 'skip' | 'reschedule'
  - newTime?: string
```

### Payments

```yaml
# Create payment intent for booking
POST /payments/intent
Body:
  - bookingId: string
  - savePaymentMethod?: boolean
Response:
  - clientSecret: string
  - paymentIntentId: string
  - amount: number
  - currency: string

# Get payment methods for current user
GET /users/me/payment-methods
Response:
  - paymentMethods: PaymentMethod[]

# Add payment method
POST /users/me/payment-methods
Body:
  - stripePaymentMethodId: string
  - setAsDefault?: boolean

# Remove payment method
DELETE /users/me/payment-methods/:paymentMethodId

# Record offline payment (staff only)
POST /facilities/:facilityId/offline-payments
Body:
  - bookingId?: string
  - billingAccountId: string
  - amount: number
  - paymentMethod: 'cash' | 'check' | 'bank_transfer' | 'venmo' | 'zelle' | 'other'
  - referenceNumber?: string
  - payerName?: string
  - paymentDate: string
  - notes?: string
  - attachments?: string[]

# Get transactions for facility (admin only)
GET /facilities/:facilityId/transactions
Query:
  - startDate?: string
  - endDate?: string
  - type?: string
  - status?: string
  - cursor?: string

# Request refund
POST /transactions/:transactionId/refund
Body:
  - amount?: number (partial refund)
  - reason: string
  - refundMethod?: 'original' | 'credit'
```

### Subscriptions

```yaml
# Get available plans
GET /plans
Response:
  - plans: Plan[]

# Get facility subscription (admin only)
GET /facilities/:facilityId/subscription
Response:
  - subscription: Subscription
  - entitlements: Entitlement[]
  - usage: Usage

# Create subscription
POST /facilities/:facilityId/subscription
Body:
  - planId: string
  - billingCycle: 'monthly' | 'annual'
  - couponCode?: string

# Change subscription plan
POST /facilities/:facilityId/subscription/change-plan
Body:
  - planId: string
  - effective: 'immediately' | 'next_period'

# Cancel subscription
POST /facilities/:facilityId/subscription/cancel
Body:
  - cancelAtPeriodEnd: boolean
  - reason?: string

# Apply coupon
POST /facilities/:facilityId/subscription/apply-coupon
Body:
  - couponCode: string

# Validate coupon
POST /coupons/validate
Body:
  - code: string
  - planId?: string
Response:
  - valid: boolean
  - coupon?: Coupon
  - discount?: Discount
```

### Coach Management

```yaml
# Get coach profile
GET /coaches/:coachId
Response:
  - id: string
  - userId: string
  - user: User
  - bio: string
  - certifications: string[]
  - sports: string[]
  - hourlyRate: number
  - rating?: number
  - facilityAffiliations: Affiliation[]

# Create/update coach profile
PUT /users/me/coach-profile
Body:
  - bio?: string
  - certifications?: string[]
  - sports?: string[]
  - hourlyRate?: number

# Get coach's students
GET /coaches/:coachId/students
Query:
  - facilityId?: string
  - status?: string
Response:
  - students: CoachStudent[]

# Add student to coach
POST /coaches/:coachId/students
Body:
  - studentUserId: string
  - parentUserId?: string
  - facilityId: string
  - sport: string
  - skillLevel?: number

# Get coach's schedule
GET /coaches/:coachId/schedule
Query:
  - facilityId?: string
  - startDate: string
  - endDate: string
Response:
  - schedule: ScheduleItem[]

# Get coach availability
GET /coaches/:coachId/availability
Query:
  - facilityId: string
Response:
  - availability: CoachAvailability[]

# Set coach availability
PUT /coaches/:coachId/availability
Body:
  - facilityId: string
  - availability: CoachAvailability[]
```

### Coach Blocks

```yaml
# Request lane block
POST /facilities/:facilityId/coach-blocks
Body:
  - resourceId: string
  - startAt: string
  - endAt: string
  - blockType: 'lesson' | 'recurring_lesson' | 'personal'
  - reason?: string
  - recurrenceRule?: string
Response:
  - block: CoachBlock
  - requiresApproval: boolean

# Get coach blocks
GET /facilities/:facilityId/coach-blocks
Query:
  - coachId?: string
  - resourceId?: string
  - status?: string
  - startDate?: string
  - endDate?: string

# Approve/reject block (admin only)
POST /facilities/:facilityId/coach-blocks/:blockId/approve
Body:
  - approved: boolean
  - reason?: string (if rejected)

# Cancel block
POST /facilities/:facilityId/coach-blocks/:blockId/cancel
Body:
  - reason?: string
```

### Session Feedback

```yaml
# Create session feedback
POST /coaching-sessions/:sessionId/feedback
Body:
  - overallRating?: number
  - effortRating?: number
  - focusRating?: number
  - summary?: string
  - voiceNoteUrl?: string
  - skillsWorked?: SkillWorked[]
  - homework?: string

# Get session feedback
GET /coaching-sessions/:sessionId/feedback
Response:
  - feedback: SessionFeedback
  - aiEnhancements?: AIEnhancement

# Update session feedback
PATCH /session-feedback/:feedbackId
Body:
  - summary?: string
  - aiEnhancedSummary?: string (coach can edit AI version)
  - homework?: string

# Publish session feedback
POST /session-feedback/:feedbackId/publish
Response:
  - feedback: SessionFeedback
  - notificationsSent: number

# Upload media to feedback
POST /session-feedback/:feedbackId/media
Body: multipart/form-data
  - file: File
  - mediaType: 'video' | 'photo' | 'audio'
  - visibility?: string

# Delete media
DELETE /session-feedback/:feedbackId/media/:mediaId

# Trigger AI enhancement
POST /session-feedback/:feedbackId/enhance
Response:
  - jobId: string
  - status: 'queued' | 'processing'

# Get enhancement status
GET /session-feedback/:feedbackId/enhance/:jobId
Response:
  - status: string
  - result?: AIEnhancement
```

### Player Progress

```yaml
# Get player progress
GET /players/:playerId/progress
Query:
  - sport?: string
Response:
  - summary: ProgressSummary
  - skills: SkillProgress[]
  - recentFeedback: SessionFeedback[]
  - achievements: Achievement[]

# Get skill progress history
GET /players/:playerId/skills/:skillId/history
Query:
  - startDate?: string
  - endDate?: string
Response:
  - history: SkillHistoryEntry[]

# Update skill level (coach only)
POST /players/:playerId/skills/:skillId/assess
Body:
  - level: number
  - notes?: string

# Generate progress report
POST /players/:playerId/progress/report
Body:
  - sport: string
  - startDate: string
  - endDate: string
Response:
  - reportUrl: string
  - aiSummary: string
```

### Notifications

```yaml
# Get notifications for current user
GET /notifications
Query:
  - unreadOnly?: boolean
  - cursor?: string
  - limit?: number
Response:
  - notifications: Notification[]
  - unreadCount: number

# Mark notification as read
POST /notifications/:notificationId/read

# Mark all as read
POST /notifications/read-all

# Get notification preferences
GET /users/me/notification-preferences
Response:
  - preferences: NotificationPreferences

# Update notification preferences
PUT /users/me/notification-preferences
Body:
  - email: object
  - sms: object
  - push: object

# Send notification (coach/admin)
POST /facilities/:facilityId/notifications
Body:
  - type: string
  - recipientType: 'user' | 'roster' | 'all_players'
  - recipientIds?: string[]
  - rosterId?: string
  - subject: string
  - body: string
  - channels: string[]
```

### Facility Operations

```yaml
# Lost & Found
# Report lost item
POST /facilities/:facilityId/lost-items
Body:
  - description: string
  - category: string
  - foundLocation: string
  - photoUrls?: string[]

# List lost items
GET /facilities/:facilityId/lost-items
Query:
  - status?: string
  - category?: string

# Claim lost item
POST /facilities/:facilityId/lost-items/:itemId/claim
Body:
  - description: string

# Tournaments
# Create tournament
POST /facilities/:facilityId/tournaments
Body:
  - name: string
  - sportType: string
  - format: string
  - startDate: string
  - endDate: string
  - maxParticipants: number
  - entryFee?: number
  - rules?: string

# Get tournament details
GET /facilities/:facilityId/tournaments/:tournamentId

# Register for tournament
POST /facilities/:facilityId/tournaments/:tournamentId/register
Body:
  - playerId: string
  - partnerId?: string (doubles)

# Rosters
# Create roster
POST /facilities/:facilityId/rosters
Body:
  - name: string
  - rosterType: 'team' | 'class' | 'camp'
  - sportType: string
  - maxMembers?: number
  - coachId?: string

# Add member to roster
POST /facilities/:facilityId/rosters/:rosterId/members
Body:
  - userId: string
  - role?: string

# Access Control
# Generate access code
POST /facilities/:facilityId/access-codes
Body:
  - userId: string
  - codeType: 'pin' | 'card' | 'temporary'
  - zones: string[]
  - validFrom?: string
  - validUntil?: string

# Get access logs (admin only)
GET /facilities/:facilityId/access-logs
Query:
  - startDate?: string
  - endDate?: string
  - userId?: string
  - deviceId?: string
```

### Analytics

```yaml
# Get facility dashboard
GET /facilities/:facilityId/analytics/dashboard
Query:
  - period: 'today' | 'week' | 'month' | 'custom'
  - startDate?: string
  - endDate?: string
Response:
  - bookings: BookingStats
  - revenue: RevenueStats
  - utilization: UtilizationStats
  - topResources: ResourceStats[]

# Get utilization report
GET /facilities/:facilityId/analytics/utilization
Query:
  - resourceId?: string
  - startDate: string
  - endDate: string
  - groupBy: 'day' | 'week' | 'hour'

# Get revenue report
GET /facilities/:facilityId/analytics/revenue
Query:
  - startDate: string
  - endDate: string
  - groupBy: 'day' | 'week' | 'month'

# Get active users report
GET /facilities/:facilityId/analytics/active-users
Query:
  - period: string

# Export report
POST /facilities/:facilityId/analytics/export
Body:
  - reportType: string
  - format: 'csv' | 'xlsx' | 'pdf'
  - dateRange: object
Response:
  - downloadUrl: string
```

### Webhooks (Facility Configuration)

```yaml
# List webhooks
GET /facilities/:facilityId/webhooks
Response:
  - webhooks: Webhook[]

# Create webhook
POST /facilities/:facilityId/webhooks
Body:
  - name: string
  - url: string
  - events: string[]
  - authType?: string
  - authConfig?: object

# Update webhook
PATCH /facilities/:facilityId/webhooks/:webhookId
Body:
  - name?: string
  - url?: string
  - events?: string[]
  - isActive?: boolean

# Delete webhook
DELETE /facilities/:facilityId/webhooks/:webhookId

# Get webhook deliveries
GET /facilities/:facilityId/webhooks/:webhookId/deliveries
Query:
  - status?: string
  - cursor?: string

# Retry webhook delivery
POST /facilities/:facilityId/webhooks/:webhookId/deliveries/:deliveryId/retry
```

---

## WebSocket API

### Connection

```javascript
// Connect to WebSocket
const socket = io('wss://api.slotbase.io', {
  auth: {
    token: '<jwt_token>'
  }
});

socket.on('connect', () => {
  console.log('Connected');
});

socket.on('error', (error) => {
  console.error('Socket error:', error);
});
```

### Channels

```javascript
// Subscribe to channels
socket.emit('subscribe', {
  channels: [
    'user:bookings',           // My booking updates
    'user:notifications',      // My notifications
    'facility:123:availability' // Facility availability
  ]
});

// Unsubscribe
socket.emit('unsubscribe', {
  channels: ['facility:123:availability']
});
```

### Events

```javascript
// Availability changed
socket.on('availability.changed', (data) => {
  // data: { resourceId, date, slots }
});

// Booking updated
socket.on('booking.update', (data) => {
  // data: { bookingId, status, updatedAt }
});

// New notification
socket.on('notification', (data) => {
  // data: { notificationId, type, title, body }
});

// Feedback published
socket.on('feedback.new', (data) => {
  // data: { feedbackId, sessionId, coachName }
});
```

---

## Error Codes

```typescript
const ERROR_CODES = {
  // Authentication
  AUTH_TOKEN_EXPIRED: 'Token has expired',
  AUTH_TOKEN_INVALID: 'Invalid token',
  AUTH_INSUFFICIENT_PERMISSIONS: 'Insufficient permissions',

  // Validation
  VALIDATION_FAILED: 'Validation failed',
  INVALID_INPUT: 'Invalid input',
  MISSING_REQUIRED_FIELD: 'Missing required field',

  // Booking
  BOOKING_SLOT_UNAVAILABLE: 'Slot is no longer available',
  BOOKING_HOLD_EXPIRED: 'Booking hold has expired',
  BOOKING_ALREADY_CANCELLED: 'Booking is already cancelled',
  BOOKING_PAST_CANCELLATION_WINDOW: 'Past cancellation window',
  BOOKING_CONFLICT: 'Booking conflicts with existing booking',

  // Payment
  PAYMENT_FAILED: 'Payment failed',
  PAYMENT_METHOD_INVALID: 'Invalid payment method',
  REFUND_EXCEEDS_AMOUNT: 'Refund exceeds original amount',
  INSUFFICIENT_BALANCE: 'Insufficient account balance',

  // Subscription
  SUBSCRIPTION_LIMIT_REACHED: 'Subscription limit reached',
  SUBSCRIPTION_INACTIVE: 'Subscription is not active',
  COUPON_INVALID: 'Invalid coupon code',
  COUPON_EXPIRED: 'Coupon has expired',

  // Coach
  COACH_NOT_AFFILIATED: 'Coach is not affiliated with this facility',
  BLOCK_APPROVAL_REQUIRED: 'Block requires admin approval',
  BLOCK_CONFLICT: 'Block conflicts with existing booking',

  // Resource
  RESOURCE_NOT_FOUND: 'Resource not found',
  RESOURCE_UNAVAILABLE: 'Resource is unavailable',

  // General
  NOT_FOUND: 'Resource not found',
  RATE_LIMIT_EXCEEDED: 'Rate limit exceeded',
  INTERNAL_ERROR: 'Internal server error',
};
```

---

## Rate Limiting

```typescript
const RATE_LIMITS = {
  // Global
  global: {
    windowMs: 60 * 1000,      // 1 minute
    maxRequests: 100,
  },

  // Per endpoint
  endpoints: {
    'POST /bookings/hold': {
      windowMs: 60 * 1000,
      maxRequests: 10,
    },
    'POST /payments/intent': {
      windowMs: 60 * 1000,
      maxRequests: 5,
    },
    'POST /session-feedback/*/enhance': {
      windowMs: 60 * 1000,
      maxRequests: 3,
    },
  },

  // Headers returned
  headers: {
    'X-RateLimit-Limit': 'Total requests allowed',
    'X-RateLimit-Remaining': 'Requests remaining',
    'X-RateLimit-Reset': 'Time when limit resets (Unix timestamp)',
  },
};
```

---

*Last Updated: 2024-01-10*
*Version: 1.0*
