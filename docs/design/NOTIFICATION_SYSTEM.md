# SlotBase Notification System Design

## Overview

SlotBase's notification system provides multi-channel communications (email, SMS, push) with role-based routing, intelligent timing, and template management.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      NOTIFICATION SYSTEM ARCHITECTURE                            │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│                           NOTIFICATION SOURCES                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │   System    │  │   Coach     │  │  Facility   │  │   Parent    │           │
│  │  (Events)   │  │  (Manual)   │  │   Admin     │  │  (Request)  │           │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │
│         │                │                │                │                    │
│         └────────────────┴────────────────┴────────────────┘                    │
│                                    │                                             │
└────────────────────────────────────┼─────────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        NOTIFICATION SERVICE                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                      Notification Orchestrator                           │   │
│  │                                                                          │   │
│  │  1. Resolve Recipients     4. Apply Templates                           │   │
│  │  2. Check Preferences      5. Schedule/Send                             │   │
│  │  3. Select Channels        6. Track Delivery                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                      │
│  │   Template    │  │  Preference   │  │    Smart      │                      │
│  │   Engine      │  │   Service     │  │   Timing      │                      │
│  └───────────────┘  └───────────────┘  └───────────────┘                      │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         DELIVERY CHANNELS                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │      EMAIL      │  │       SMS       │  │      PUSH       │                 │
│  │                 │  │                 │  │                 │                 │
│  │    Resend       │  │    Twilio       │  │      FCM        │                 │
│  │   (Provider)    │  │   (Provider)    │  │   (Provider)    │                 │
│  │                 │  │                 │  │                 │                 │
│  │  - Transactional│  │  - Reminders    │  │  - Real-time    │                 │
│  │  - Summaries    │  │  - Urgent       │  │  - Updates      │                 │
│  │  - Marketing*   │  │  - Codes        │  │  - Actions      │                 │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                 │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       DELIVERY TRACKING                                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  Notification Log:                                                       │   │
│  │                                                                          │   │
│  │  - Sent timestamp                - Open/click tracking                  │   │
│  │  - Delivery status               - Bounce handling                      │   │
│  │  - Channel used                  - Unsubscribe tracking                 │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Notification Types

### System Notifications (Automated)

```typescript
const SYSTEM_NOTIFICATIONS = {
  // Booking notifications
  booking: {
    confirmed: {
      id: 'booking.confirmed',
      name: 'Booking Confirmed',
      description: 'Sent when a booking is confirmed',
      channels: ['email', 'push'],
      priority: 'high',
      recipients: ['booker', 'participants'],
    },
    reminder_24h: {
      id: 'booking.reminder.24h',
      name: 'Booking Reminder (24h)',
      description: 'Reminder 24 hours before booking',
      channels: ['email', 'push'],
      priority: 'normal',
      recipients: ['booker', 'participants', 'parents'],
    },
    reminder_1h: {
      id: 'booking.reminder.1h',
      name: 'Booking Reminder (1h)',
      description: 'Reminder 1 hour before booking',
      channels: ['push', 'sms'],
      priority: 'high',
      recipients: ['booker', 'participants', 'parents'],
    },
    cancelled: {
      id: 'booking.cancelled',
      name: 'Booking Cancelled',
      description: 'Sent when a booking is cancelled',
      channels: ['email', 'push'],
      priority: 'high',
      recipients: ['booker', 'participants', 'parents'],
    },
    modified: {
      id: 'booking.modified',
      name: 'Booking Modified',
      description: 'Sent when booking time or details change',
      channels: ['email', 'push'],
      priority: 'high',
      recipients: ['booker', 'participants', 'parents'],
    },
  },

  // Payment notifications
  payment: {
    succeeded: {
      id: 'payment.succeeded',
      name: 'Payment Successful',
      description: 'Confirmation of payment',
      channels: ['email'],
      priority: 'normal',
      recipients: ['payer'],
    },
    failed: {
      id: 'payment.failed',
      name: 'Payment Failed',
      description: 'Payment could not be processed',
      channels: ['email', 'push'],
      priority: 'high',
      recipients: ['payer'],
    },
    refunded: {
      id: 'payment.refunded',
      name: 'Refund Processed',
      description: 'Confirmation of refund',
      channels: ['email'],
      priority: 'normal',
      recipients: ['payer'],
    },
    reminder: {
      id: 'payment.reminder',
      name: 'Payment Reminder',
      description: 'Reminder for unpaid booking',
      channels: ['email'],
      priority: 'normal',
      recipients: ['payer'],
    },
  },

  // Coach notifications
  coach: {
    block_approved: {
      id: 'coach.block.approved',
      name: 'Block Approved',
      description: 'Lane block request approved',
      channels: ['email', 'push'],
      priority: 'normal',
      recipients: ['coach'],
    },
    block_rejected: {
      id: 'coach.block.rejected',
      name: 'Block Rejected',
      description: 'Lane block request rejected',
      channels: ['email', 'push'],
      priority: 'normal',
      recipients: ['coach'],
    },
    session_reminder: {
      id: 'coach.session.reminder',
      name: 'Upcoming Session',
      description: 'Reminder of upcoming coaching session',
      channels: ['push'],
      priority: 'normal',
      recipients: ['coach'],
    },
    feedback_reminder: {
      id: 'coach.feedback.reminder',
      name: 'Feedback Reminder',
      description: 'Reminder to submit session feedback',
      channels: ['push', 'email'],
      priority: 'normal',
      recipients: ['coach'],
    },
  },

  // Feedback notifications
  feedback: {
    published: {
      id: 'feedback.published',
      name: 'Session Feedback Available',
      description: 'New session feedback from coach',
      channels: ['email', 'push'],
      priority: 'normal',
      recipients: ['parents', 'student'],
    },
    media_ready: {
      id: 'feedback.media.ready',
      name: 'Video/Photo Ready',
      description: 'Media from session is ready to view',
      channels: ['push'],
      priority: 'low',
      recipients: ['parents', 'student'],
    },
  },

  // Progress notifications
  progress: {
    skill_improved: {
      id: 'progress.skill.improved',
      name: 'Skill Improvement',
      description: 'Student improved in a skill',
      channels: ['push'],
      priority: 'low',
      recipients: ['parents', 'student'],
    },
    achievement: {
      id: 'progress.achievement',
      name: 'Achievement Earned',
      description: 'Student earned an achievement',
      channels: ['push'],
      priority: 'low',
      recipients: ['parents', 'student'],
    },
    weekly_summary: {
      id: 'progress.weekly',
      name: 'Weekly Progress Summary',
      description: 'Weekly progress report',
      channels: ['email'],
      priority: 'low',
      recipients: ['parents'],
      scheduled: true,
    },
  },

  // Subscription notifications
  subscription: {
    limit_warning: {
      id: 'subscription.limit.warning',
      name: 'Approaching Limit',
      description: 'Approaching active user limit',
      channels: ['email', 'push'],
      priority: 'high',
      recipients: ['facility_admins'],
    },
    renewal_success: {
      id: 'subscription.renewed',
      name: 'Subscription Renewed',
      description: 'Subscription successfully renewed',
      channels: ['email'],
      priority: 'normal',
      recipients: ['facility_owner'],
    },
    renewal_failed: {
      id: 'subscription.renewal.failed',
      name: 'Renewal Failed',
      description: 'Subscription renewal payment failed',
      channels: ['email', 'push'],
      priority: 'high',
      recipients: ['facility_owner'],
    },
  },

  // Facility notifications
  facility: {
    lost_item: {
      id: 'facility.lost_item',
      name: 'Lost Item Found',
      description: 'New lost item reported',
      channels: ['email'],
      priority: 'low',
      recipients: ['recent_visitors'],
    },
    schedule_change: {
      id: 'facility.schedule.change',
      name: 'Schedule Change',
      description: 'Facility schedule has changed',
      channels: ['email', 'push'],
      priority: 'high',
      recipients: ['affected_users'],
    },
    closure: {
      id: 'facility.closure',
      name: 'Facility Closure',
      description: 'Facility temporarily closed',
      channels: ['email', 'sms', 'push'],
      priority: 'urgent',
      recipients: ['affected_users'],
    },
  },
};
```

### User-Initiated Notifications

```typescript
const USER_NOTIFICATIONS = {
  // Coach → Parents/Students
  coach_to_students: {
    session_update: {
      id: 'coach.message.session_update',
      allowedSenders: ['coach'],
      allowedRecipients: ['students', 'parents'],
      channels: ['email', 'push'],
      requiresContent: true,
    },
    schedule_reminder: {
      id: 'coach.message.schedule_reminder',
      allowedSenders: ['coach'],
      allowedRecipients: ['students', 'parents'],
      channels: ['push'],
      template: 'coach_schedule_reminder',
    },
    homework_assigned: {
      id: 'coach.message.homework',
      allowedSenders: ['coach'],
      allowedRecipients: ['students', 'parents'],
      channels: ['email', 'push'],
    },
    session_cancelled: {
      id: 'coach.message.session_cancelled',
      allowedSenders: ['coach'],
      allowedRecipients: ['students', 'parents'],
      channels: ['email', 'push', 'sms'],
      priority: 'high',
    },
  },

  // Parent → Coach/Facility
  parent_requests: {
    booking_request: {
      id: 'parent.request.booking',
      allowedSenders: ['parent'],
      allowedRecipients: ['coach', 'facility_staff'],
      channels: ['email', 'push'],
    },
    cancellation_request: {
      id: 'parent.request.cancellation',
      allowedSenders: ['parent'],
      allowedRecipients: ['coach', 'facility_staff'],
      channels: ['email'],
    },
    question: {
      id: 'parent.request.question',
      allowedSenders: ['parent'],
      allowedRecipients: ['coach'],
      channels: ['email'],
      rateLimit: { maxPerDay: 5 },
    },
  },

  // Facility → Users
  facility_announcements: {
    announcement: {
      id: 'facility.announcement',
      allowedSenders: ['facility_admin'],
      allowedRecipients: ['all_users', 'roster', 'players'],
      channels: ['email', 'push'],
    },
    tournament: {
      id: 'facility.tournament',
      allowedSenders: ['facility_admin'],
      allowedRecipients: ['all_users', 'roster'],
      channels: ['email'],
    },
    promotion: {
      id: 'facility.promotion',
      allowedSenders: ['facility_admin'],
      allowedRecipients: ['all_users'],
      channels: ['email'],
      requiresConsent: true,  // Marketing opt-in required
    },
  },
};
```

---

## User Preferences

### Preference Structure

```typescript
interface NotificationPreferences {
  userId: string;

  // Global settings
  global: {
    enabled: boolean;
    quietHoursEnabled: boolean;
    quietHoursStart: string;  // HH:MM
    quietHoursEnd: string;    // HH:MM
    timezone: string;
  };

  // Per-channel settings
  channels: {
    email: {
      enabled: boolean;
      address: string;
      verified: boolean;
    };
    sms: {
      enabled: boolean;
      phoneNumber: string;
      verified: boolean;
    };
    push: {
      enabled: boolean;
      deviceTokens: string[];
    };
  };

  // Per-notification-type settings
  notifications: {
    [notificationTypeId: string]: {
      enabled: boolean;
      channels: ('email' | 'sms' | 'push')[];
    };
  };

  // Digest preferences
  digest: {
    enabled: boolean;
    frequency: 'daily' | 'weekly';
    dayOfWeek?: number;  // For weekly
    timeOfDay: string;   // HH:MM
  };

  // Marketing preferences
  marketing: {
    enabled: boolean;
    facilityUpdates: boolean;
    productNews: boolean;
  };
}

// Default preferences
const DEFAULT_PREFERENCES: NotificationPreferences = {
  global: {
    enabled: true,
    quietHoursEnabled: true,
    quietHoursStart: '22:00',
    quietHoursEnd: '07:00',
    timezone: 'America/Los_Angeles',
  },
  channels: {
    email: { enabled: true },
    sms: { enabled: true },
    push: { enabled: true },
  },
  notifications: {
    // High priority - always enabled by default
    'booking.confirmed': { enabled: true, channels: ['email', 'push'] },
    'booking.cancelled': { enabled: true, channels: ['email', 'push'] },
    'booking.reminder.24h': { enabled: true, channels: ['email'] },
    'booking.reminder.1h': { enabled: true, channels: ['push'] },
    'payment.failed': { enabled: true, channels: ['email', 'push'] },

    // Normal priority
    'feedback.published': { enabled: true, channels: ['email', 'push'] },
    'progress.weekly': { enabled: true, channels: ['email'] },

    // Low priority - user can disable
    'progress.skill.improved': { enabled: true, channels: ['push'] },
    'progress.achievement': { enabled: true, channels: ['push'] },
  },
  digest: {
    enabled: false,
    frequency: 'weekly',
    dayOfWeek: 1, // Monday
    timeOfDay: '09:00',
  },
  marketing: {
    enabled: false,
    facilityUpdates: true,
    productNews: false,
  },
};
```

### Preferences UI

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       NOTIFICATION PREFERENCES                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  CHANNELS                                                                        │
│  ─────────────────────────────────────────────────────────────────────────      │
│                                                                                  │
│  Email    john@example.com           [✓]  Verified                              │
│  SMS      +1 (555) 123-4567          [✓]  Verified                              │
│  Push     2 devices connected         [✓]                                        │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────      │
│                                                                                  │
│  QUIET HOURS                                     [✓] Enabled                    │
│  Don't disturb me between 10:00 PM - 7:00 AM                                    │
│  (Urgent notifications will still come through)                                 │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────      │
│                                                                                  │
│  NOTIFICATION TYPES                                                             │
│                                                                                  │
│  BOOKINGS                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐     │
│  │  Booking confirmations        Email [✓]  SMS [ ]  Push [✓]          │     │
│  │  Booking reminders            Email [✓]  SMS [ ]  Push [✓]          │     │
│  │  Cancellations                Email [✓]  SMS [✓]  Push [✓]          │     │
│  └──────────────────────────────────────────────────────────────────────┘     │
│                                                                                  │
│  PAYMENTS                                                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐     │
│  │  Payment confirmations        Email [✓]  SMS [ ]  Push [ ]          │     │
│  │  Payment failures             Email [✓]  SMS [ ]  Push [✓]          │     │
│  │  Payment reminders            Email [✓]  SMS [ ]  Push [ ]          │     │
│  └──────────────────────────────────────────────────────────────────────┘     │
│                                                                                  │
│  COACHING                                                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐     │
│  │  Session feedback             Email [✓]  SMS [ ]  Push [✓]          │     │
│  │  Progress updates             Email [ ]  SMS [ ]  Push [✓]          │     │
│  │  Weekly summary               Email [✓]  SMS [ ]  Push [ ]          │     │
│  └──────────────────────────────────────────────────────────────────────┘     │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────      │
│                                                                                  │
│  EMAIL DIGEST                                    [ ] Enabled                    │
│  Combine non-urgent notifications into a daily summary                          │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────      │
│                                                                                  │
│  MARKETING                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐     │
│  │  [ ] Receive facility updates and promotions                         │     │
│  │  [ ] Receive SlotBase product news                                    │     │
│  └──────────────────────────────────────────────────────────────────────┘     │
│                                                                                  │
│                              [ Save Preferences ]                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Template System

### Template Structure

```typescript
interface NotificationTemplate {
  id: string;
  notificationType: string;
  channel: 'email' | 'sms' | 'push';

  // Content
  subject?: string;      // Email only
  title?: string;        // Push only
  body: string;
  htmlBody?: string;     // Email only

  // Variables
  variables: string[];   // Available template variables

  // Customization
  facilityId?: string;   // Null = default template
  locale: string;        // 'en', 'es', etc.

  // Status
  isActive: boolean;
  version: number;
}

// Example template
const BOOKING_CONFIRMED_EMAIL: NotificationTemplate = {
  id: 'booking.confirmed.email.en',
  notificationType: 'booking.confirmed',
  channel: 'email',
  subject: 'Booking Confirmed: {{resource_name}} on {{date}}',
  body: `
Hi {{booker_name}},

Your booking has been confirmed!

📍 {{facility_name}}
🎾 {{resource_name}}
📅 {{date}} at {{time}}
⏱️ {{duration}}

{{#if has_participants}}
Playing with: {{participants}}
{{/if}}

{{#if prepaid}}
Amount paid: {{amount}}
{{else}}
Amount due: {{amount}}
Payment: {{payment_instructions}}
{{/if}}

Need to make changes? Manage your booking:
{{booking_url}}

See you on the court!
{{facility_name}}
  `,
  htmlBody: `<!-- HTML version -->`,
  variables: [
    'booker_name',
    'facility_name',
    'resource_name',
    'date',
    'time',
    'duration',
    'amount',
    'booking_url',
    'participants',
    'has_participants',
    'prepaid',
    'payment_instructions',
  ],
  locale: 'en',
  isActive: true,
  version: 1,
};
```

### Template Variables by Type

```typescript
const TEMPLATE_VARIABLES = {
  // User variables (available in all templates)
  user: {
    user_name: 'Full name',
    user_first_name: 'First name',
    user_email: 'Email address',
  },

  // Booking variables
  booking: {
    booking_id: 'Booking ID',
    facility_name: 'Facility name',
    resource_name: 'Court/lane name',
    date: 'Formatted date',
    time: 'Formatted time',
    duration: 'Duration string',
    amount: 'Formatted price',
    booking_url: 'Link to booking',
    cancel_url: 'Link to cancel',
    participants: 'List of participants',
    booker_name: 'Person who booked',
  },

  // Payment variables
  payment: {
    amount: 'Payment amount',
    currency: 'Currency code',
    payment_method: 'Card ending, etc.',
    receipt_url: 'Link to receipt',
    refund_amount: 'Refund amount',
  },

  // Feedback variables
  feedback: {
    coach_name: 'Coach name',
    session_date: 'Session date',
    summary: 'Feedback summary',
    skills_worked: 'Skills list',
    homework: 'Homework assigned',
    feedback_url: 'Link to full feedback',
    has_video: 'Boolean',
    has_photos: 'Boolean',
  },

  // Progress variables
  progress: {
    skill_name: 'Skill that improved',
    old_level: 'Previous level',
    new_level: 'New level',
    achievement_name: 'Achievement name',
    streak_days: 'Current streak',
  },

  // Coach variables
  coach: {
    block_date: 'Block date',
    block_time: 'Block time range',
    resource_name: 'Court/lane blocked',
    rejection_reason: 'Why rejected',
    student_name: 'Student name',
    parent_name: 'Parent name',
  },
};
```

---

## Channel Providers

### Email Provider (Resend)

```typescript
// email/resend.provider.ts
import { Resend } from 'resend';

interface EmailConfig {
  apiKey: string;
  fromEmail: string;
  fromName: string;
  replyTo?: string;
}

class ResendEmailProvider implements EmailProvider {
  private client: Resend;

  constructor(private config: EmailConfig) {
    this.client = new Resend(config.apiKey);
  }

  async send(email: EmailMessage): Promise<SendResult> {
    try {
      const result = await this.client.emails.send({
        from: `${this.config.fromName} <${this.config.fromEmail}>`,
        to: email.to,
        subject: email.subject,
        html: email.htmlBody,
        text: email.textBody,
        reply_to: email.replyTo || this.config.replyTo,
        tags: [
          { name: 'notification_type', value: email.notificationType },
          { name: 'facility_id', value: email.facilityId || 'platform' },
        ],
      });

      return {
        success: true,
        messageId: result.id,
        provider: 'resend',
      };
    } catch (error) {
      return {
        success: false,
        error: error.message,
        provider: 'resend',
      };
    }
  }
}
```

### SMS Provider (Twilio)

```typescript
// sms/twilio.provider.ts
import twilio from 'twilio';

interface SmsConfig {
  accountSid: string;
  authToken: string;
  fromNumber: string;
}

class TwilioSmsProvider implements SmsProvider {
  private client: twilio.Twilio;

  constructor(private config: SmsConfig) {
    this.client = twilio(config.accountSid, config.authToken);
  }

  async send(sms: SmsMessage): Promise<SendResult> {
    try {
      const result = await this.client.messages.create({
        body: sms.body,
        from: this.config.fromNumber,
        to: sms.to,
      });

      return {
        success: true,
        messageId: result.sid,
        provider: 'twilio',
      };
    } catch (error) {
      return {
        success: false,
        error: error.message,
        provider: 'twilio',
      };
    }
  }
}
```

### Push Provider (Firebase Cloud Messaging)

```typescript
// push/fcm.provider.ts
import * as admin from 'firebase-admin';

class FcmPushProvider implements PushProvider {
  constructor() {
    admin.initializeApp({
      credential: admin.credential.cert(serviceAccount),
    });
  }

  async send(push: PushMessage): Promise<SendResult> {
    try {
      const message: admin.messaging.Message = {
        notification: {
          title: push.title,
          body: push.body,
        },
        data: push.data,
        token: push.deviceToken,
        android: {
          priority: push.priority === 'high' ? 'high' : 'normal',
        },
        apns: {
          payload: {
            aps: {
              badge: push.badge,
              sound: push.sound || 'default',
            },
          },
        },
      };

      const result = await admin.messaging().send(message);

      return {
        success: true,
        messageId: result,
        provider: 'fcm',
      };
    } catch (error) {
      return {
        success: false,
        error: error.message,
        provider: 'fcm',
      };
    }
  }

  async sendToMultiple(
    push: PushMessage,
    tokens: string[]
  ): Promise<BatchSendResult> {
    const message: admin.messaging.MulticastMessage = {
      notification: {
        title: push.title,
        body: push.body,
      },
      data: push.data,
      tokens,
    };

    const result = await admin.messaging().sendMulticast(message);

    return {
      successCount: result.successCount,
      failureCount: result.failureCount,
      responses: result.responses,
    };
  }
}
```

---

## Notification Orchestration

### Send Flow

```typescript
// notification.service.ts
@Injectable()
class NotificationService {
  constructor(
    private templateService: TemplateService,
    private preferenceService: PreferenceService,
    private emailProvider: EmailProvider,
    private smsProvider: SmsProvider,
    private pushProvider: PushProvider,
    private notificationLog: NotificationLogService,
  ) {}

  async send(notification: NotificationRequest): Promise<NotificationResult> {
    // 1. Resolve recipients
    const recipients = await this.resolveRecipients(notification);

    const results: RecipientResult[] = [];

    for (const recipient of recipients) {
      // 2. Check preferences
      const prefs = await this.preferenceService.get(recipient.userId);

      if (!this.shouldSend(notification, prefs)) {
        results.push({ userId: recipient.userId, skipped: true, reason: 'preferences' });
        continue;
      }

      // 3. Determine channels
      const channels = this.selectChannels(notification, prefs);

      // 4. Check quiet hours
      const effectiveChannels = this.applyQuietHours(channels, prefs);

      // 5. Render template for each channel
      for (const channel of effectiveChannels) {
        const template = await this.templateService.get(
          notification.type,
          channel,
          recipient.locale
        );

        const rendered = this.templateService.render(template, {
          ...notification.data,
          user_name: recipient.name,
          user_first_name: recipient.firstName,
        });

        // 6. Send via appropriate provider
        const sendResult = await this.sendViaChannel(channel, recipient, rendered);

        // 7. Log result
        await this.notificationLog.log({
          notificationId: notification.id,
          userId: recipient.userId,
          channel,
          status: sendResult.success ? 'sent' : 'failed',
          messageId: sendResult.messageId,
          error: sendResult.error,
        });

        results.push({
          userId: recipient.userId,
          channel,
          success: sendResult.success,
          messageId: sendResult.messageId,
        });
      }
    }

    return { notificationId: notification.id, results };
  }

  private async sendViaChannel(
    channel: string,
    recipient: Recipient,
    content: RenderedContent
  ): Promise<SendResult> {
    switch (channel) {
      case 'email':
        return this.emailProvider.send({
          to: recipient.email,
          subject: content.subject,
          htmlBody: content.htmlBody,
          textBody: content.textBody,
        });

      case 'sms':
        return this.smsProvider.send({
          to: recipient.phone,
          body: content.body,
        });

      case 'push':
        return this.pushProvider.send({
          deviceToken: recipient.deviceToken,
          title: content.title,
          body: content.body,
          data: content.data,
        });

      default:
        throw new Error(`Unknown channel: ${channel}`);
    }
  }

  private applyQuietHours(
    channels: string[],
    prefs: NotificationPreferences
  ): string[] {
    if (!prefs.global.quietHoursEnabled) {
      return channels;
    }

    const now = new Date();
    const userTime = convertToTimezone(now, prefs.global.timezone);

    if (isInQuietHours(userTime, prefs.global.quietHoursStart, prefs.global.quietHoursEnd)) {
      // During quiet hours, only allow email (will be seen later)
      // Remove push and SMS unless urgent
      return channels.filter(c => c === 'email');
    }

    return channels;
  }
}
```

### Recipient Resolution

```typescript
// Recipient resolution for different notification types
async function resolveRecipients(
  notification: NotificationRequest
): Promise<Recipient[]> {
  const recipients: Recipient[] = [];

  switch (notification.recipientType) {
    case 'booker':
      const booking = await getBooking(notification.referenceId);
      recipients.push(await getUser(booking.bookerId));
      break;

    case 'participants':
      const bookingParticipants = await getBookingParticipants(notification.referenceId);
      for (const p of bookingParticipants) {
        recipients.push(await getUser(p.userId));
      }
      break;

    case 'parents':
      // Get parents of a child player
      const parentLinks = await getParentLinks(notification.userId);
      for (const link of parentLinks) {
        if (link.canReceiveUpdates) {
          recipients.push(await getUser(link.parentUserId));
        }
      }
      break;

    case 'students':
      // Get coach's students
      const students = await getCoachStudents(notification.coachId);
      for (const student of students) {
        // Add student
        if (student.studentAge >= 13) {
          recipients.push(await getUser(student.studentUserId));
        }
        // Add parents
        const parents = await getParentLinks(student.studentUserId);
        for (const parent of parents) {
          recipients.push(await getUser(parent.parentUserId));
        }
      }
      break;

    case 'facility_admins':
      const admins = await getFacilityAdmins(notification.facilityId);
      recipients.push(...admins);
      break;

    case 'roster':
      const rosterMembers = await getRosterMembers(notification.rosterId);
      for (const member of rosterMembers) {
        recipients.push(await getUser(member.userId));
      }
      break;

    case 'affected_users':
      // Users with bookings in affected time range
      const affectedBookings = await getBookingsInRange(
        notification.facilityId,
        notification.startTime,
        notification.endTime
      );
      const uniqueUserIds = new Set(affectedBookings.map(b => b.bookerId));
      for (const userId of uniqueUserIds) {
        recipients.push(await getUser(userId));
      }
      break;
  }

  // De-duplicate
  return deduplicateRecipients(recipients);
}
```

---

## Scheduled Notifications

### Booking Reminders

```typescript
// Scheduled job for booking reminders
@Cron('* * * * *') // Every minute
async processBookingReminders() {
  const now = new Date();

  // 24-hour reminders
  const bookings24h = await this.bookingService.findUpcoming({
    startAfter: addHours(now, 23),
    startBefore: addHours(now, 25),
    reminderSent: { '24h': false },
    status: 'confirmed',
  });

  for (const booking of bookings24h) {
    await this.notificationService.send({
      type: 'booking.reminder.24h',
      referenceId: booking.id,
      recipientType: 'participants_and_parents',
    });

    await this.bookingService.markReminderSent(booking.id, '24h');
  }

  // 1-hour reminders
  const bookings1h = await this.bookingService.findUpcoming({
    startAfter: addMinutes(now, 55),
    startBefore: addMinutes(now, 65),
    reminderSent: { '1h': false },
    status: 'confirmed',
  });

  for (const booking of bookings1h) {
    await this.notificationService.send({
      type: 'booking.reminder.1h',
      referenceId: booking.id,
      recipientType: 'participants_and_parents',
    });

    await this.bookingService.markReminderSent(booking.id, '1h');
  }
}
```

### Weekly Progress Summary

```typescript
// Weekly progress summary - runs Monday mornings
@Cron('0 9 * * 1') // Monday 9 AM
async sendWeeklyProgressSummaries() {
  // Get all players with progress tracking enabled
  const players = await this.playerService.findWithProgressTracking();

  for (const player of players) {
    // Check if parent has weekly summary enabled
    const parents = await this.familyService.getParents(player.id);

    for (const parent of parents) {
      const prefs = await this.preferenceService.get(parent.id);

      if (prefs.notifications['progress.weekly']?.enabled) {
        // Generate summary
        const summary = await this.progressService.generateWeeklySummary(player.id);

        await this.notificationService.send({
          type: 'progress.weekly',
          userId: parent.id,
          data: {
            player_name: player.firstName,
            summary: summary.text,
            sessions_count: summary.sessionsCount,
            hours_practiced: summary.hoursPracticed,
            skills_improved: summary.skillsImproved,
            dashboard_url: `${APP_URL}/progress/${player.id}`,
          },
        });
      }
    }
  }
}
```

---

## Rate Limiting & Throttling

```typescript
const NOTIFICATION_RATE_LIMITS = {
  // Global limits per user
  perUser: {
    email: { maxPerHour: 20, maxPerDay: 50 },
    sms: { maxPerHour: 5, maxPerDay: 15 },
    push: { maxPerHour: 30, maxPerDay: 100 },
  },

  // Per notification type
  perType: {
    'payment.reminder': { maxPerDay: 1 },
    'booking.reminder.24h': { maxPerBooking: 1 },
    'feedback.reminder': { maxPerSession: 3 },
  },

  // Facility limits (for announcements)
  perFacility: {
    'facility.announcement': { maxPerDay: 3 },
    'facility.promotion': { maxPerWeek: 2 },
  },
};

async function checkRateLimit(
  userId: string,
  channel: string,
  notificationType: string
): Promise<{ allowed: boolean; retryAfter?: number }> {
  const key = `rate_limit:${userId}:${channel}`;
  const count = await redis.incr(key);

  if (count === 1) {
    await redis.expire(key, 3600); // 1 hour TTL
  }

  const limit = NOTIFICATION_RATE_LIMITS.perUser[channel].maxPerHour;

  if (count > limit) {
    const ttl = await redis.ttl(key);
    return { allowed: false, retryAfter: ttl };
  }

  return { allowed: true };
}
```

---

## Delivery Tracking

```typescript
interface NotificationDelivery {
  id: string;
  notificationId: string;
  userId: string;
  channel: 'email' | 'sms' | 'push';

  // Status
  status: 'pending' | 'sent' | 'delivered' | 'failed' | 'bounced';

  // Provider details
  provider: string;
  providerMessageId?: string;

  // Timestamps
  createdAt: Date;
  sentAt?: Date;
  deliveredAt?: Date;
  failedAt?: Date;

  // Engagement
  openedAt?: Date;
  clickedAt?: Date;
  clickedLinks?: string[];

  // Error
  errorCode?: string;
  errorMessage?: string;

  // Retry
  retryCount: number;
  nextRetryAt?: Date;
}

// Webhook handlers for delivery tracking
async function handleResendWebhook(event: ResendWebhookEvent) {
  switch (event.type) {
    case 'email.delivered':
      await updateDeliveryStatus(event.data.email_id, 'delivered', {
        deliveredAt: new Date(event.created_at),
      });
      break;

    case 'email.opened':
      await updateDeliveryEngagement(event.data.email_id, {
        openedAt: new Date(event.created_at),
      });
      break;

    case 'email.clicked':
      await updateDeliveryEngagement(event.data.email_id, {
        clickedAt: new Date(event.created_at),
        clickedLinks: [event.data.click.link],
      });
      break;

    case 'email.bounced':
      await updateDeliveryStatus(event.data.email_id, 'bounced', {
        errorCode: event.data.bounce.type,
        errorMessage: event.data.bounce.message,
      });
      // Mark email as invalid
      await markEmailInvalid(event.data.to);
      break;
  }
}
```

---

## Unsubscribe Handling

```typescript
// One-click unsubscribe support
function generateUnsubscribeUrl(userId: string, notificationType?: string): string {
  const token = signUnsubscribeToken({ userId, notificationType });
  return `${APP_URL}/unsubscribe?token=${token}`;
}

// Add List-Unsubscribe header to emails
function addUnsubscribeHeaders(email: EmailMessage, userId: string): EmailMessage {
  const unsubscribeUrl = generateUnsubscribeUrl(userId);

  return {
    ...email,
    headers: {
      ...email.headers,
      'List-Unsubscribe': `<${unsubscribeUrl}>`,
      'List-Unsubscribe-Post': 'List-Unsubscribe=One-Click',
    },
  };
}

// Handle unsubscribe request
async function handleUnsubscribe(token: string): Promise<void> {
  const { userId, notificationType } = verifyUnsubscribeToken(token);

  if (notificationType) {
    // Unsubscribe from specific type
    await preferenceService.updateNotificationPreference(userId, notificationType, {
      enabled: false,
    });
  } else {
    // Unsubscribe from all marketing
    await preferenceService.updateMarketingPreference(userId, {
      enabled: false,
    });
  }

  // Log for compliance
  await auditLog.log({
    action: 'unsubscribe',
    userId,
    notificationType,
    timestamp: new Date(),
  });
}
```

---

*Last Updated: 2024-01-10*
*Version: 1.0*
