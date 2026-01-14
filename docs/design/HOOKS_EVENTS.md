# SlotBase Hooks & Events Specification

## Overview

SlotBase uses an event-driven architecture with hooks to enable extensibility, real-time updates, and integration with external systems. This document defines all system events, hook points, and integration patterns.

---

## Event Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           EVENT FLOW ARCHITECTURE                                │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │   Service    │──── Emit Event ────┐
  │   (Source)   │                    │
  └──────────────┘                    │
                                      ▼
                           ┌──────────────────┐
                           │   Event Bus      │
                           │   (Redis)        │
                           └──────────────────┘
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         │                            │                            │
         ▼                            ▼                            ▼
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│ Internal Handler │       │  Webhook Queue   │       │  Agent Handler   │
│                  │       │                  │       │                  │
│ - Notifications  │       │ - Facility hooks │       │ - AI Processing  │
│ - Analytics      │       │ - Integration    │       │ - Enforcement    │
│ - Cache update   │       │ - External API   │       │ - Monitoring     │
└──────────────────┘       └──────────────────┘       └──────────────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │  Webhook Worker  │
                           │                  │
                           │ - Retry logic    │
                           │ - Signature      │
                           │ - Delivery       │
                           └──────────────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │  External        │
                           │  Endpoint        │
                           └──────────────────┘
```

---

## Domain Events

### Event Structure

```typescript
interface DomainEvent {
  // Identification
  id: string;                    // UUID
  type: string;                  // e.g., 'booking.confirmed'
  version: string;               // e.g., '1.0'

  // Timing
  timestamp: string;             // ISO 8601
  occurredAt: string;            // When the event actually occurred

  // Source
  source: string;                // e.g., 'booking-service'
  correlationId: string;         // For tracing across services
  causationId?: string;          // ID of event that caused this

  // Context
  facilityId?: string;
  userId?: string;
  organizationId?: string;

  // Payload
  data: Record<string, unknown>;

  // Metadata
  metadata: {
    environment: 'production' | 'staging' | 'development';
    clientIp?: string;
    userAgent?: string;
  };
}
```

### Booking Events

```typescript
// booking.hold.created
interface BookingHoldCreatedEvent extends DomainEvent {
  type: 'booking.hold.created';
  data: {
    holdId: string;
    bookingId: string;
    facilityId: string;
    resourceId: string;
    userId: string;
    startAt: string;
    endAt: string;
    expiresAt: string;
    priceAmount: number;
    priceCurrency: string;
  };
}

// booking.confirmed
interface BookingConfirmedEvent extends DomainEvent {
  type: 'booking.confirmed';
  data: {
    bookingId: string;
    facilityId: string;
    resourceId: string;
    resourceName: string;
    bookerId: string;
    bookerName: string;
    bookerEmail: string;
    participants: Array<{
      userId: string;
      name: string;
      role: string;
    }>;
    startAt: string;
    endAt: string;
    timezone: string;
    bookingType: string;
    priceAmount: number;
    priceCurrency: string;
    paymentMethod: 'online' | 'offline' | 'waived';
    isRecurring: boolean;
    recurringBookingId?: string;
  };
}

// booking.cancelled
interface BookingCancelledEvent extends DomainEvent {
  type: 'booking.cancelled';
  data: {
    bookingId: string;
    facilityId: string;
    resourceId: string;
    bookerId: string;
    startAt: string;
    endAt: string;
    cancelledBy: string;
    cancellationReason: string;
    refundAmount?: number;
    refundStatus?: string;
    wasNoShow: boolean;
  };
}

// booking.checked_in
interface BookingCheckedInEvent extends DomainEvent {
  type: 'booking.checked_in';
  data: {
    bookingId: string;
    facilityId: string;
    userId: string;
    checkedInAt: string;
    checkedInBy: 'self' | 'staff';
    method: 'qr_code' | 'manual' | 'access_code';
  };
}

// booking.no_show
interface BookingNoShowEvent extends DomainEvent {
  type: 'booking.no_show';
  data: {
    bookingId: string;
    facilityId: string;
    userId: string;
    scheduledAt: string;
    markedAt: string;
    feeCharged: number;
    strikeCount: number;
    restrictionApplied: boolean;
  };
}

// booking.completed
interface BookingCompletedEvent extends DomainEvent {
  type: 'booking.completed';
  data: {
    bookingId: string;
    facilityId: string;
    resourceId: string;
    bookerId: string;
    participants: string[];
    startAt: string;
    endAt: string;
    actualDuration: number; // minutes
    wasCoachingSession: boolean;
    coachingSessionId?: string;
  };
}

// booking.reminder.sent
interface BookingReminderSentEvent extends DomainEvent {
  type: 'booking.reminder.sent';
  data: {
    bookingId: string;
    userId: string;
    reminderType: '24h' | '1h' | 'custom';
    channel: 'email' | 'sms' | 'push';
    sentAt: string;
  };
}
```

### Payment Events

```typescript
// payment.succeeded
interface PaymentSucceededEvent extends DomainEvent {
  type: 'payment.succeeded';
  data: {
    transactionId: string;
    facilityId: string;
    billingAccountId: string;
    payerId: string;
    amount: number;
    currency: string;
    paymentMethod: string;
    referenceType: 'booking' | 'subscription' | 'invoice';
    referenceId: string;
    stripePaymentIntentId?: string;
  };
}

// payment.failed
interface PaymentFailedEvent extends DomainEvent {
  type: 'payment.failed';
  data: {
    transactionId?: string;
    facilityId: string;
    billingAccountId: string;
    payerId: string;
    amount: number;
    currency: string;
    failureReason: string;
    failureCode?: string;
    referenceType: string;
    referenceId: string;
    retryCount: number;
    nextRetryAt?: string;
  };
}

// payment.refunded
interface PaymentRefundedEvent extends DomainEvent {
  type: 'payment.refunded';
  data: {
    refundId: string;
    transactionId: string;
    facilityId: string;
    amount: number;
    currency: string;
    reason: string;
    refundedBy: string;
    referenceType: string;
    referenceId: string;
  };
}

// offline_payment.recorded
interface OfflinePaymentRecordedEvent extends DomainEvent {
  type: 'offline_payment.recorded';
  data: {
    offlinePaymentId: string;
    facilityId: string;
    billingAccountId: string;
    amount: number;
    currency: string;
    paymentMethod: 'cash' | 'check' | 'bank_transfer' | 'venmo' | 'zelle' | 'other';
    referenceNumber?: string;
    payerName: string;
    recordedBy: string;
    paymentDate: string;
    bookingId?: string;
  };
}
```

### Subscription Events

```typescript
// subscription.created
interface SubscriptionCreatedEvent extends DomainEvent {
  type: 'subscription.created';
  data: {
    subscriptionId: string;
    facilityId: string;
    planId: string;
    planName: string;
    billingCycle: 'monthly' | 'annual';
    status: string;
    trialEnd?: string;
    currentPeriodStart: string;
    currentPeriodEnd: string;
    couponApplied?: string;
    discountAmount?: number;
  };
}

// subscription.renewed
interface SubscriptionRenewedEvent extends DomainEvent {
  type: 'subscription.renewed';
  data: {
    subscriptionId: string;
    facilityId: string;
    planId: string;
    previousPeriodEnd: string;
    newPeriodStart: string;
    newPeriodEnd: string;
    amount: number;
    currency: string;
  };
}

// subscription.cancelled
interface SubscriptionCancelledEvent extends DomainEvent {
  type: 'subscription.cancelled';
  data: {
    subscriptionId: string;
    facilityId: string;
    cancelledBy: string;
    cancelReason?: string;
    cancelAtPeriodEnd: boolean;
    effectiveDate: string;
  };
}

// subscription.plan_changed
interface SubscriptionPlanChangedEvent extends DomainEvent {
  type: 'subscription.plan_changed';
  data: {
    subscriptionId: string;
    facilityId: string;
    previousPlanId: string;
    previousPlanName: string;
    newPlanId: string;
    newPlanName: string;
    changeType: 'upgrade' | 'downgrade';
    effectiveDate: string;
    proratedAmount?: number;
  };
}

// subscription.limit_warning
interface SubscriptionLimitWarningEvent extends DomainEvent {
  type: 'subscription.limit_warning';
  data: {
    subscriptionId: string;
    facilityId: string;
    entitlementType: string;
    currentUsage: number;
    limit: number;
    percentUsed: number;
    warningLevel: 'approaching' | 'at_limit' | 'exceeded';
    recommendedAction: string;
  };
}
```

### Coach Events

```typescript
// coach.block.requested
interface CoachBlockRequestedEvent extends DomainEvent {
  type: 'coach.block.requested';
  data: {
    blockId: string;
    coachId: string;
    coachName: string;
    facilityId: string;
    resourceId: string;
    resourceName: string;
    startAt: string;
    endAt: string;
    blockType: string;
    reason?: string;
    requiresApproval: boolean;
  };
}

// coach.block.approved
interface CoachBlockApprovedEvent extends DomainEvent {
  type: 'coach.block.approved';
  data: {
    blockId: string;
    coachId: string;
    facilityId: string;
    approvedBy: string;
    approvedAt: string;
  };
}

// coach.block.rejected
interface CoachBlockRejectedEvent extends DomainEvent {
  type: 'coach.block.rejected';
  data: {
    blockId: string;
    coachId: string;
    facilityId: string;
    rejectedBy: string;
    rejectedAt: string;
    rejectionReason: string;
  };
}

// coach.session.completed
interface CoachSessionCompletedEvent extends DomainEvent {
  type: 'coach.session.completed';
  data: {
    sessionId: string;
    bookingId: string;
    coachId: string;
    coachName: string;
    facilityId: string;
    sessionType: string;
    students: Array<{
      userId: string;
      name: string;
      parentIds: string[];
    }>;
    startAt: string;
    endAt: string;
    attendedStudents: string[];
  };
}
```

### Feedback Events

```typescript
// feedback.created
interface FeedbackCreatedEvent extends DomainEvent {
  type: 'feedback.created';
  data: {
    feedbackId: string;
    sessionId: string;
    coachId: string;
    facilityId: string;
    hasVoiceNote: boolean;
    hasMedia: boolean;
  };
}

// feedback.enhanced
interface FeedbackEnhancedEvent extends DomainEvent {
  type: 'feedback.enhanced';
  data: {
    feedbackId: string;
    sessionId: string;
    enhancementType: 'transcription' | 'text_enhancement' | 'skill_extraction' | 'all';
    skillsExtracted: string[];
    processingTime: number;
  };
}

// feedback.published
interface FeedbackPublishedEvent extends DomainEvent {
  type: 'feedback.published';
  data: {
    feedbackId: string;
    sessionId: string;
    coachId: string;
    coachName: string;
    facilityId: string;
    studentId: string;
    studentName: string;
    parentIds: string[];
    hasVideo: boolean;
    hasPhotos: boolean;
    skillsWorked: string[];
  };
}

// media.uploaded
interface MediaUploadedEvent extends DomainEvent {
  type: 'media.uploaded';
  data: {
    mediaId: string;
    feedbackId: string;
    mediaType: 'video' | 'photo' | 'audio';
    fileName: string;
    fileSizeBytes: number;
    durationSeconds?: number;
    uploadedBy: string;
  };
}

// media.processed
interface MediaProcessedEvent extends DomainEvent {
  type: 'media.processed';
  data: {
    mediaId: string;
    feedbackId: string;
    processingType: 'transcoding' | 'thumbnail' | 'ai_analysis';
    success: boolean;
    outputUrl?: string;
    aiInsights?: object;
    processingTime: number;
  };
}

// progress.updated
interface ProgressUpdatedEvent extends DomainEvent {
  type: 'progress.updated';
  data: {
    playerId: string;
    skillId: string;
    skillName: string;
    sport: string;
    previousLevel: number;
    newLevel: number;
    assessedBy: string;
    improvementPercent: number;
  };
}
```

### User Events

```typescript
// user.created
interface UserCreatedEvent extends DomainEvent {
  type: 'user.created';
  data: {
    userId: string;
    email: string;
    firstName: string;
    lastName: string;
    registrationSource: 'web' | 'mobile' | 'admin' | 'invitation';
    facilityId?: string;
  };
}

// user.role_assigned
interface UserRoleAssignedEvent extends DomainEvent {
  type: 'user.role_assigned';
  data: {
    userId: string;
    role: string;
    scopeType: string;
    scopeId: string;
    grantedBy: string;
  };
}

// parent_child.linked
interface ParentChildLinkedEvent extends DomainEvent {
  type: 'parent_child.linked';
  data: {
    parentId: string;
    childId: string;
    relationship: string;
    isPrimary: boolean;
    permissions: string[];
  };
}
```

### Facility Events

```typescript
// facility.created
interface FacilityCreatedEvent extends DomainEvent {
  type: 'facility.created';
  data: {
    facilityId: string;
    name: string;
    timezone: string;
    sportTypes: string[];
    ownerId: string;
  };
}

// facility.settings_updated
interface FacilitySettingsUpdatedEvent extends DomainEvent {
  type: 'facility.settings_updated';
  data: {
    facilityId: string;
    changedSettings: string[];
    changedBy: string;
  };
}

// resource.created
interface ResourceCreatedEvent extends DomainEvent {
  type: 'resource.created';
  data: {
    resourceId: string;
    facilityId: string;
    name: string;
    resourceType: string;
  };
}

// lost_item.reported
interface LostItemReportedEvent extends DomainEvent {
  type: 'lost_item.reported';
  data: {
    itemId: string;
    facilityId: string;
    description: string;
    category: string;
    foundLocation: string;
    reportedBy: string;
  };
}

// lost_item.claimed
interface LostItemClaimedEvent extends DomainEvent {
  type: 'lost_item.claimed';
  data: {
    itemId: string;
    facilityId: string;
    claimedBy: string;
    verifiedBy: string;
  };
}
```

---

## Hooks System

### Hook Configuration

```typescript
interface WebhookConfig {
  id: string;
  facilityId: string;
  name: string;
  description?: string;

  // Endpoint
  url: string;
  method: 'POST' | 'PUT';

  // Authentication
  authType: 'none' | 'header' | 'basic' | 'bearer' | 'signature';
  authConfig?: {
    headerName?: string;
    headerValue?: string;
    username?: string;
    password?: string;
    bearerToken?: string;
    signingSecret?: string;
  };

  // Events
  events: string[];  // e.g., ['booking.confirmed', 'booking.cancelled']

  // Filters
  filters?: {
    resourceIds?: string[];
    bookingTypes?: string[];
    // Custom filters
  };

  // Retry configuration
  retryConfig: {
    maxRetries: number;
    retryDelayMs: number;
    backoffMultiplier: number;
  };

  // Status
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}
```

### Webhook Delivery

```typescript
interface WebhookDelivery {
  id: string;
  webhookId: string;
  eventId: string;
  eventType: string;

  // Request
  url: string;
  method: string;
  headers: Record<string, string>;
  payload: string;

  // Response
  responseStatus?: number;
  responseBody?: string;
  responseHeaders?: Record<string, string>;

  // Timing
  attemptCount: number;
  firstAttemptAt: string;
  lastAttemptAt: string;
  nextRetryAt?: string;

  // Status
  status: 'pending' | 'delivered' | 'failed' | 'retrying';
  errorMessage?: string;

  // Duration
  durationMs?: number;
}
```

### Webhook Signature

```typescript
// Signature generation
function generateWebhookSignature(
  payload: string,
  secret: string,
  timestamp: number
): string {
  const signaturePayload = `${timestamp}.${payload}`;
  return crypto
    .createHmac('sha256', secret)
    .update(signaturePayload)
    .digest('hex');
}

// Webhook headers sent
interface WebhookHeaders {
  'X-SlotBase-Event': string;           // Event type
  'X-SlotBase-Event-ID': string;        // Event ID
  'X-SlotBase-Timestamp': string;       // Unix timestamp
  'X-SlotBase-Signature': string;       // HMAC signature
  'X-SlotBase-Delivery-ID': string;     // Delivery attempt ID
  'Content-Type': 'application/json';
}

// Verification on receiver side
function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string,
  timestamp: number,
  toleranceSeconds: number = 300
): boolean {
  // Check timestamp is recent
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - timestamp) > toleranceSeconds) {
    return false;
  }

  // Verify signature
  const expectedSignature = generateWebhookSignature(payload, secret, timestamp);
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}
```

### Pre-defined Hook Points

```typescript
// System-level hooks (SlotBase internal)
const SYSTEM_HOOKS = {
  // Before hooks (can block/modify)
  'before:booking.create': {
    description: 'Before a booking is created',
    canBlock: true,
    canModify: true,
    timeout: 5000,
  },
  'before:payment.process': {
    description: 'Before payment is processed',
    canBlock: true,
    canModify: false,
    timeout: 5000,
  },

  // After hooks (notification only)
  'after:booking.confirmed': {
    description: 'After booking is confirmed',
    canBlock: false,
    canModify: false,
  },
  'after:payment.succeeded': {
    description: 'After payment succeeds',
    canBlock: false,
    canModify: false,
  },
};

// Facility-configurable webhooks
const FACILITY_WEBHOOK_EVENTS = [
  // Booking events
  'booking.confirmed',
  'booking.cancelled',
  'booking.checked_in',
  'booking.no_show',
  'booking.completed',

  // Payment events
  'payment.succeeded',
  'payment.failed',
  'payment.refunded',
  'offline_payment.recorded',

  // Coach events
  'coach.block.requested',
  'coach.block.approved',
  'coach.session.completed',

  // Feedback events
  'feedback.published',

  // User events
  'user.created',
  'parent_child.linked',

  // Subscription events (for facility's own sub)
  'subscription.limit_warning',
];
```

---

## Event Handlers

### Internal Event Handlers

```typescript
// Event handler registry
const EVENT_HANDLERS: Record<string, EventHandler[]> = {
  'booking.confirmed': [
    NotificationHandler,      // Send confirmations
    AnalyticsHandler,         // Track booking
    CacheInvalidationHandler, // Update availability cache
    CapacityTrackingHandler,  // Update active user count
    CalendarSyncHandler,      // Sync to external calendar
  ],

  'booking.cancelled': [
    NotificationHandler,
    AnalyticsHandler,
    CacheInvalidationHandler,
    RefundHandler,            // Process refund if needed
    WaitlistHandler,          // Notify waitlist
  ],

  'payment.succeeded': [
    NotificationHandler,
    AnalyticsHandler,
    BookingConfirmationHandler, // Confirm booking if held
    InvoiceHandler,           // Generate invoice/receipt
  ],

  'feedback.created': [
    FeedbackEnhancementAgent, // AI processing
  ],

  'feedback.published': [
    NotificationHandler,      // Notify parents
    ProgressUpdateHandler,    // Update skill progress
  ],

  'media.uploaded': [
    MediaProcessingHandler,   // Transcode video
    AIAnalysisHandler,        // Analyze content
  ],

  'subscription.limit_warning': [
    NotificationHandler,      // Alert facility admin
    UpgradePromptHandler,     // Show upgrade prompt
  ],
};
```

### Handler Implementation Example

```typescript
// handlers/notification.handler.ts
@Injectable()
export class NotificationHandler implements EventHandler {
  constructor(
    private notificationService: NotificationService,
    private templateService: TemplateService,
  ) {}

  async handle(event: DomainEvent): Promise<void> {
    const config = this.getNotificationConfig(event.type);
    if (!config) return;

    for (const recipientConfig of config.recipients) {
      const recipients = await this.resolveRecipients(event, recipientConfig);

      for (const recipient of recipients) {
        const template = await this.templateService.render(
          config.templateId,
          { event, recipient }
        );

        await this.notificationService.send({
          userId: recipient.userId,
          channel: recipientConfig.channel,
          template,
          priority: config.priority,
          metadata: {
            eventId: event.id,
            eventType: event.type,
          },
        });
      }
    }
  }

  private getNotificationConfig(eventType: string): NotificationConfig | null {
    return NOTIFICATION_CONFIGS[eventType] || null;
  }

  private async resolveRecipients(
    event: DomainEvent,
    config: RecipientConfig
  ): Promise<Recipient[]> {
    switch (config.type) {
      case 'booker':
        return [{ userId: event.data.bookerId }];
      case 'participants':
        return event.data.participants.map(p => ({ userId: p.userId }));
      case 'parents':
        return this.getParents(event.data.studentId);
      case 'facility_admins':
        return this.getFacilityAdmins(event.facilityId);
      default:
        return [];
    }
  }
}

// Notification configuration
const NOTIFICATION_CONFIGS: Record<string, NotificationConfig> = {
  'booking.confirmed': {
    templateId: 'booking-confirmation',
    priority: 'high',
    recipients: [
      { type: 'booker', channel: 'email' },
      { type: 'booker', channel: 'push' },
      { type: 'participants', channel: 'email' },
    ],
  },
  'feedback.published': {
    templateId: 'session-feedback',
    priority: 'normal',
    recipients: [
      { type: 'parents', channel: 'email' },
      { type: 'parents', channel: 'push' },
    ],
  },
};
```

---

## Scheduled Events (Cron Jobs)

```typescript
// Scheduled job definitions
const SCHEDULED_JOBS = {
  // Every minute
  'booking-reminder-check': {
    schedule: '* * * * *',
    handler: BookingReminderJob,
    description: 'Check for upcoming bookings needing reminders',
  },
  'no-show-detection': {
    schedule: '* * * * *',
    handler: NoShowDetectionJob,
    description: 'Mark no-shows after grace period',
  },
  'hold-expiration': {
    schedule: '* * * * *',
    handler: HoldExpirationJob,
    description: 'Release expired booking holds',
  },

  // Every 5 minutes
  'webhook-retry': {
    schedule: '*/5 * * * *',
    handler: WebhookRetryJob,
    description: 'Retry failed webhook deliveries',
  },

  // Every hour
  'subscription-renewal-check': {
    schedule: '0 * * * *',
    handler: SubscriptionRenewalJob,
    description: 'Process subscription renewals',
  },
  'capacity-check': {
    schedule: '0 * * * *',
    handler: CapacityCheckJob,
    description: 'Check facility capacity limits',
  },

  // Daily at 6 AM
  'daily-reports': {
    schedule: '0 6 * * *',
    handler: DailyReportJob,
    description: 'Generate daily reports for facilities',
  },
  'offline-payment-aging': {
    schedule: '0 6 * * *',
    handler: OfflinePaymentAgingJob,
    description: 'Check for aging unpaid bookings',
  },

  // Weekly Monday 9 AM
  'weekly-progress-summary': {
    schedule: '0 9 * * 1',
    handler: WeeklyProgressSummaryJob,
    description: 'Generate player progress summaries',
  },

  // Monthly 1st at midnight
  'monthly-billing': {
    schedule: '0 0 1 * *',
    handler: MonthlyBillingJob,
    description: 'Process monthly invoices',
  },
  'active-user-reset': {
    schedule: '0 0 1 * *',
    handler: ActiveUserResetJob,
    description: 'Reset monthly active user counts',
  },
};
```

---

## Real-time Events (WebSocket)

```typescript
// WebSocket event types
const WEBSOCKET_EVENTS = {
  // Client -> Server
  'subscribe': {
    description: 'Subscribe to a channel',
    payload: { channel: string },
  },
  'unsubscribe': {
    description: 'Unsubscribe from a channel',
    payload: { channel: string },
  },

  // Server -> Client
  'availability.changed': {
    description: 'Resource availability changed',
    channels: ['facility:{facilityId}:availability'],
    payload: {
      resourceId: string,
      date: string,
      slots: Slot[],
    },
  },
  'booking.update': {
    description: 'Booking status changed',
    channels: ['user:{userId}:bookings', 'facility:{facilityId}:bookings'],
    payload: {
      bookingId: string,
      status: string,
      updatedAt: string,
    },
  },
  'notification': {
    description: 'New notification for user',
    channels: ['user:{userId}:notifications'],
    payload: {
      notificationId: string,
      type: string,
      title: string,
      body: string,
      data: object,
    },
  },
  'feedback.new': {
    description: 'New feedback available',
    channels: ['user:{userId}:feedback'],
    payload: {
      feedbackId: string,
      sessionId: string,
      coachName: string,
    },
  },
};

// Channel patterns
const WEBSOCKET_CHANNELS = {
  // User-specific
  'user:{userId}:bookings': 'Booking updates for user',
  'user:{userId}:notifications': 'Notifications for user',
  'user:{userId}:feedback': 'Feedback updates for user',

  // Facility-wide
  'facility:{facilityId}:availability': 'Availability changes',
  'facility:{facilityId}:bookings': 'All bookings (admin)',
  'facility:{facilityId}:check-ins': 'Check-in events (staff)',

  // Resource-specific
  'resource:{resourceId}:availability': 'Single resource availability',
};
```

---

## Integration Examples

### Zapier/Make Integration

```typescript
// Webhook payload format for Zapier
interface ZapierWebhookPayload {
  event_type: string;
  timestamp: string;
  facility: {
    id: string;
    name: string;
  };
  data: Record<string, unknown>;
}

// Example: booking.confirmed -> Google Sheets
const zapierExample = {
  trigger: 'booking.confirmed',
  webhook_url: 'https://hooks.zapier.com/hooks/catch/xxx/yyy/',
  payload_mapping: {
    'Booking ID': '{{data.bookingId}}',
    'Customer Name': '{{data.bookerName}}',
    'Customer Email': '{{data.bookerEmail}}',
    'Court': '{{data.resourceName}}',
    'Date': '{{data.startAt | date: "%Y-%m-%d"}}',
    'Time': '{{data.startAt | date: "%H:%M"}}',
    'Amount': '{{data.priceAmount}}',
  },
};
```

### Calendar Sync (Google Calendar)

```typescript
// Calendar sync configuration
interface CalendarSyncConfig {
  id: string;
  facilityId: string;
  userId: string;
  provider: 'google' | 'outlook' | 'apple';
  credentials: EncryptedCredentials;
  calendarId: string;

  // Sync settings
  syncDirection: 'push' | 'pull' | 'both';
  events: string[]; // ['booking.confirmed', 'booking.cancelled']

  // Mapping
  titleTemplate: string;
  descriptionTemplate: string;
  locationTemplate: string;

  // Status
  lastSyncAt: string;
  syncStatus: 'active' | 'error' | 'paused';
}

// Calendar event creation on booking.confirmed
async function syncToGoogleCalendar(
  event: BookingConfirmedEvent,
  config: CalendarSyncConfig
): Promise<void> {
  const calendarEvent = {
    summary: renderTemplate(config.titleTemplate, event.data),
    description: renderTemplate(config.descriptionTemplate, event.data),
    location: renderTemplate(config.locationTemplate, event.data),
    start: {
      dateTime: event.data.startAt,
      timeZone: event.data.timezone,
    },
    end: {
      dateTime: event.data.endAt,
      timeZone: event.data.timezone,
    },
    extendedProperties: {
      private: {
        slotbaseBookingId: event.data.bookingId,
      },
    },
  };

  await googleCalendarClient.events.insert({
    calendarId: config.calendarId,
    resource: calendarEvent,
  });
}
```

---

## Error Handling

```typescript
// Event processing error handling
interface EventProcessingError {
  eventId: string;
  eventType: string;
  handlerName: string;
  error: {
    message: string;
    code: string;
    stack?: string;
  };
  attemptCount: number;
  maxRetries: number;
  nextRetryAt?: string;
  status: 'retrying' | 'failed' | 'dead_letter';
}

// Dead letter queue for failed events
interface DeadLetterEntry {
  id: string;
  eventId: string;
  event: DomainEvent;
  errors: EventProcessingError[];
  createdAt: string;
  lastAttemptAt: string;
  resolvedAt?: string;
  resolvedBy?: string;
  resolution?: 'reprocessed' | 'ignored' | 'manual';
}

// Retry configuration
const RETRY_CONFIG = {
  defaultMaxRetries: 3,
  defaultDelayMs: 1000,
  maxDelayMs: 60000,
  backoffMultiplier: 2,

  // Per-handler overrides
  handlers: {
    NotificationHandler: { maxRetries: 5 },
    PaymentHandler: { maxRetries: 0 }, // No retry for payments
    WebhookHandler: { maxRetries: 5, maxDelayMs: 3600000 }, // 1 hour max
  },
};
```

---

## Monitoring & Observability

```typescript
// Event metrics
const EVENT_METRICS = {
  // Counters
  'events.published.total': 'Total events published',
  'events.processed.total': 'Total events processed',
  'events.failed.total': 'Total events failed',

  // Histograms
  'events.processing.duration': 'Event processing duration',
  'webhooks.delivery.duration': 'Webhook delivery duration',

  // Gauges
  'events.queue.size': 'Event queue size',
  'webhooks.pending.count': 'Pending webhook deliveries',
  'dead_letter.size': 'Dead letter queue size',
};

// Event logging
interface EventLog {
  eventId: string;
  eventType: string;
  timestamp: string;
  facilityId?: string;
  userId?: string;
  handlers: Array<{
    name: string;
    status: 'success' | 'failed' | 'skipped';
    duration: number;
    error?: string;
  }>;
  webhooks: Array<{
    webhookId: string;
    deliveryId: string;
    status: string;
    duration?: number;
  }>;
}
```

---

*Last Updated: 2024-01-10*
*Version: 1.0*
