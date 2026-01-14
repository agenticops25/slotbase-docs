# SlotBase System Architecture

## Overview

SlotBase follows a modular monolith architecture, designed for rapid iteration in early phases while maintaining clear boundaries for future service extraction.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENTS                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                     │
│   │   Web App    │    │  Mobile App  │    │  Admin App   │                     │
│   │  (Next.js)   │    │(React Native)│    │  (Next.js)   │                     │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘                     │
│          │                   │                   │                              │
│          └───────────────────┴───────────────────┘                              │
│                              │                                                   │
└──────────────────────────────┼───────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EDGE LAYER                                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐ │
│   │                         Cloudflare (CDN + WAF)                            │ │
│   │                    - DDoS protection                                      │ │
│   │                    - SSL termination                                      │ │
│   │                    - Rate limiting (edge)                                 │ │
│   │                    - Static asset caching                                 │ │
│   └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            API GATEWAY LAYER                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐ │
│   │                         API Gateway (NestJS)                              │ │
│   │                                                                           │ │
│   │   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │ │
│   │   │   Auth      │ │   Rate      │ │  Request    │ │  Response   │       │ │
│   │   │   Guard     │ │   Limiter   │ │  Validator  │ │  Transform  │       │ │
│   │   └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘       │ │
│   │                                                                           │ │
│   │   REST API: /api/v1/*          WebSocket: /ws                            │ │
│   └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          APPLICATION LAYER (Modules)                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │  Identity   │ │Organization │ │  Resource   │ │   Booking   │              │
│  │   Module    │ │   Module    │ │   Module    │ │   Module    │              │
│  │             │ │             │ │             │ │             │              │
│  │ - Auth      │ │ - Facility  │ │ - Courts    │ │ - Schedule  │              │
│  │ - Users     │ │ - Settings  │ │ - Lanes     │ │ - Holds     │              │
│  │ - Roles     │ │ - Staff     │ │ - Pricing   │ │ - Recurring │              │
│  │ - Sessions  │ │ - Policies  │ │ - Maint.    │ │ - Waitlist  │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │  Payment    │ │Subscription │ │   Coach     │ │  Feedback   │              │
│  │   Module    │ │   Module    │ │   Module    │ │   Module    │              │
│  │             │ │             │ │             │ │             │              │
│  │ - Stripe    │ │ - Plans     │ │ - Profiles  │ │ - Sessions  │              │
│  │ - Offline   │ │ - Entitle.  │ │ - Blocks    │ │ - AI Proc.  │              │
│  │ - Refunds   │ │ - Billing   │ │ - Students  │ │ - Media     │              │
│  │ - Invoices  │ │ - Coupons   │ │ - Schedule  │ │ - Progress  │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │   Family    │ │Notification │ │  Analytics  │ │Facility Ops │              │
│  │   Module    │ │   Module    │ │   Module    │ │   Module    │              │
│  │             │ │             │ │             │ │             │              │
│  │ - Parents   │ │ - Email     │ │ - Events    │ │ - Access    │              │
│  │ - Children  │ │ - SMS       │ │ - Reports   │ │ - Devices   │              │
│  │ - Links     │ │ - Push      │ │ - Metrics   │ │ - Lost&Found│              │
│  │ - Consents  │ │ - Templates │ │ - Exports   │ │ - Tourney   │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            AI AGENT LAYER                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                        Agent Orchestrator                                 │  │
│  │                   (Coordinates all AI agents)                             │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │ Scheduling  │ │  Capacity   │ │  Payment    │ │  Feedback   │              │
│  │ Fairness    │ │  & Limits   │ │ Enforcement │ │Enhancement  │              │
│  │   Agent     │ │   Agent     │ │   Agent     │ │   Agent     │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │  Progress   │ │Notification │ │   Video     │ │  No-Show    │              │
│  │  Insights   │ │   Smart     │ │  Analysis   │ │  Detection  │              │
│  │   Agent     │ │   Agent     │ │   Agent     │ │   Agent     │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          INFRASTRUCTURE LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │    PostgreSQL    │  │      Redis       │  │    BullMQ        │             │
│  │    (Primary DB)  │  │  (Cache/Pub-Sub) │  │   (Job Queue)    │             │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘             │
│                                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │  Cloudflare R2   │  │       Mux        │  │     OpenAI       │             │
│  │  (File Storage)  │  │ (Video Process)  │  │   (AI/LLM API)   │             │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘             │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL SERVICES                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │   Stripe    │ │    Clerk    │ │   Resend    │ │   Twilio    │              │
│  │  (Payments) │ │   (Auth)    │ │   (Email)   │ │   (SMS)     │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                               │
│  │   Sentry    │ │   Axiom     │ │  PostHog    │                               │
│  │  (Errors)   │ │   (Logs)    │ │ (Analytics) │                               │
│  └─────────────┘ └─────────────┘ └─────────────┘                               │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Module Architecture

### Module Boundaries & Communication

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          MODULE COMMUNICATION PATTERNS                          │
└────────────────────────────────────────────────────────────────────────────────┘

  SYNCHRONOUS (Direct method calls within monolith)
  ─────────────────────────────────────────────────

  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
  │   Booking    │────────▶│   Resource   │────────▶│Organization  │
  │   Module     │         │   Module     │         │   Module     │
  └──────────────┘         └──────────────┘         └──────────────┘
         │                        │
         │                        │
         ▼                        ▼
  ┌──────────────┐         ┌──────────────┐
  │   Payment    │         │Subscription  │
  │   Module     │         │   Module     │
  └──────────────┘         └──────────────┘


  ASYNCHRONOUS (Event-driven via Redis pub/sub + BullMQ)
  ───────────────────────────────────────────────────────

  ┌──────────────┐                              ┌──────────────┐
  │   Booking    │───── BookingCreated ────────▶│Notification  │
  │   Module     │───── BookingCancelled ──────▶│   Module     │
  └──────────────┘                              └──────────────┘
         │
         │───── SessionCompleted ──────────────▶┌──────────────┐
         │                                      │   Feedback   │
         │                                      │   Module     │
         │                                      └──────────────┘
         │
         │───── BookingCreated ────────────────▶┌──────────────┐
                                                │  Analytics   │
                                                │   Module     │
                                                └──────────────┘


  QUERY PATTERNS
  ──────────────

  Module A needs data from Module B:

  PREFERRED: Module A calls Module B's service interface
  ┌────────────┐                    ┌────────────┐
  │  Booking   │──getAvailability()──▶│  Resource  │
  │  Service   │◀──────────────────────│  Service   │
  └────────────┘                    └────────────┘

  AVOID: Direct database queries across modules
  ┌────────────┐                    ┌────────────┐
  │  Booking   │────── SQL ────────▶│  Resource  │
  │  Service   │                    │   Tables   │
  └────────────┘                    └────────────┘
```

---

## Module Specifications

### Identity Module

```typescript
// Module interface
interface IdentityModule {
  // User management
  createUser(data: CreateUserDto): Promise<User>;
  getUser(id: string): Promise<User>;
  updateUser(id: string, data: UpdateUserDto): Promise<User>;
  deleteUser(id: string): Promise<void>;

  // Role management
  assignRole(userId: string, role: RoleAssignment): Promise<void>;
  removeRole(userId: string, roleId: string): Promise<void>;
  getUserRoles(userId: string, scope?: RoleScope): Promise<Role[]>;
  checkPermission(userId: string, permission: string, scope: RoleScope): Promise<boolean>;

  // Session management
  validateSession(token: string): Promise<SessionInfo>;
  invalidateSession(sessionId: string): Promise<void>;
}

// Events emitted
type IdentityEvents = {
  'user.created': { userId: string; email: string };
  'user.updated': { userId: string; changes: string[] };
  'user.deleted': { userId: string };
  'role.assigned': { userId: string; role: string; scope: RoleScope };
  'role.removed': { userId: string; role: string; scope: RoleScope };
};

// Data owned
const IDENTITY_TABLES = [
  'users',
  'user_roles',
  'sessions',
  'parent_child_links',
  'family_groups',
  'family_members',
];
```

### Booking Module

```typescript
// Module interface
interface BookingModule {
  // Availability
  getAvailability(facilityId: string, resourceId: string, dateRange: DateRange): Promise<Slot[]>;

  // Booking lifecycle
  createHold(data: CreateHoldDto): Promise<BookingHold>;
  confirmBooking(holdId: string, paymentIntentId?: string): Promise<Booking>;
  cancelBooking(bookingId: string, reason: string): Promise<void>;
  rescheduleBooking(bookingId: string, newSlot: SlotDto): Promise<Booking>;

  // Recurring
  createRecurringBooking(data: CreateRecurringDto): Promise<RecurringBooking>;
  modifyRecurringBooking(id: string, data: ModifyRecurringDto): Promise<RecurringBooking>;

  // Check-in
  checkIn(bookingId: string, userId: string): Promise<void>;
  markNoShow(bookingId: string): Promise<void>;

  // Waitlist
  joinWaitlist(data: WaitlistDto): Promise<WaitlistEntry>;
  leaveWaitlist(entryId: string): Promise<void>;
}

// Events emitted
type BookingEvents = {
  'booking.hold.created': { holdId: string; expiresAt: Date };
  'booking.confirmed': { bookingId: string; userId: string; facilityId: string };
  'booking.cancelled': { bookingId: string; reason: string; cancelledBy: string };
  'booking.checked_in': { bookingId: string; userId: string };
  'booking.no_show': { bookingId: string; userId: string };
  'booking.completed': { bookingId: string };
  'waitlist.notified': { entryId: string; slotAvailable: Slot };
};

// Dependencies
const BOOKING_DEPENDENCIES = [
  'ResourceModule',      // Get availability, pricing
  'SubscriptionModule',  // Check entitlements
  'PaymentModule',       // Process payments
  'IdentityModule',      // Get user info
];
```

### Payment Module

```typescript
// Module interface
interface PaymentModule {
  // Online payments (Stripe)
  createPaymentIntent(data: PaymentIntentDto): Promise<{ clientSecret: string }>;
  confirmPayment(paymentIntentId: string): Promise<Transaction>;
  processRefund(transactionId: string, amount?: number): Promise<Refund>;

  // Offline payments
  recordOfflinePayment(data: OfflinePaymentDto): Promise<OfflinePayment>;
  verifyOfflinePayment(paymentId: string): Promise<OfflinePayment>;

  // Billing accounts
  createBillingAccount(data: CreateBillingAccountDto): Promise<BillingAccount>;
  addPaymentMethod(accountId: string, paymentMethodId: string): Promise<PaymentMethod>;

  // Invoicing
  generateInvoice(accountId: string, lineItems: LineItem[]): Promise<Invoice>;

  // Stripe Connect (facility onboarding)
  createConnectAccount(facilityId: string): Promise<{ onboardingUrl: string }>;
  getConnectAccountStatus(facilityId: string): Promise<ConnectAccountStatus>;
}

// Events emitted
type PaymentEvents = {
  'payment.succeeded': { transactionId: string; amount: number; bookingId?: string };
  'payment.failed': { paymentIntentId: string; reason: string };
  'payment.refunded': { refundId: string; amount: number };
  'offline_payment.recorded': { paymentId: string; recordedBy: string };
  'invoice.created': { invoiceId: string; accountId: string };
  'invoice.paid': { invoiceId: string };
};
```

### Feedback Module

```typescript
// Module interface
interface FeedbackModule {
  // Session feedback
  createFeedback(sessionId: string, data: CreateFeedbackDto): Promise<SessionFeedback>;
  updateFeedback(feedbackId: string, data: UpdateFeedbackDto): Promise<SessionFeedback>;
  publishFeedback(feedbackId: string): Promise<void>;

  // Media
  uploadMedia(feedbackId: string, file: File): Promise<FeedbackMedia>;
  deleteMedia(mediaId: string): Promise<void>;

  // AI enhancement
  enhanceFeedback(feedbackId: string): Promise<AIEnhancement>;
  generateProgressSummary(playerId: string, sportType: string): Promise<ProgressSummary>;

  // Progress tracking
  getPlayerProgress(playerId: string): Promise<PlayerProgress>;
  updateSkillLevel(playerId: string, skillId: string, level: number): Promise<void>;

  // Drill library
  suggestDrills(playerId: string, skillGaps: string[]): Promise<Drill[]>;
}

// Events emitted
type FeedbackEvents = {
  'feedback.created': { feedbackId: string; sessionId: string; coachId: string };
  'feedback.published': { feedbackId: string; recipients: string[] };
  'media.uploaded': { mediaId: string; feedbackId: string; type: 'video' | 'photo' };
  'media.processed': { mediaId: string; aiAnalysis: object };
  'progress.updated': { playerId: string; skillId: string; newLevel: number };
};
```

---

## Event-Driven Architecture

### Event Bus Implementation

```typescript
// Event types
interface DomainEvent {
  id: string;
  type: string;
  timestamp: Date;
  source: string;
  correlationId: string;
  payload: Record<string, unknown>;
  metadata: {
    userId?: string;
    facilityId?: string;
    version: number;
  };
}

// Event bus interface
interface EventBus {
  publish(event: DomainEvent): Promise<void>;
  subscribe(eventType: string, handler: EventHandler): void;
  subscribePattern(pattern: string, handler: EventHandler): void;
}

// Implementation using Redis pub/sub
class RedisEventBus implements EventBus {
  constructor(private redis: Redis) {}

  async publish(event: DomainEvent): Promise<void> {
    // Publish to Redis channel
    await this.redis.publish(`events:${event.type}`, JSON.stringify(event));

    // Also store in event log for replay
    await this.redis.xadd('event-log', '*', 'data', JSON.stringify(event));
  }

  subscribe(eventType: string, handler: EventHandler): void {
    // Subscribe to specific event type
    this.redis.subscribe(`events:${eventType}`, (message) => {
      const event = JSON.parse(message);
      handler(event);
    });
  }
}
```

### Event Flow Example: Booking Confirmation

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     EVENT FLOW: BOOKING CONFIRMATION                             │
└─────────────────────────────────────────────────────────────────────────────────┘

  1. User confirms booking
         │
         ▼
  ┌──────────────┐
  │   Booking    │──── booking.confirmed ────┬────────────────────────────────────┐
  │   Module     │                           │                                    │
  └──────────────┘                           │                                    │
                                             │                                    │
         ┌───────────────────────────────────┼────────────────────────────────────┼───┐
         │                                   │                                    │   │
         ▼                                   ▼                                    ▼   │
  ┌──────────────┐                    ┌──────────────┐                    ┌──────────────┐
  │Notification  │                    │  Analytics   │                    │Subscription  │
  │   Module     │                    │   Module     │                    │   Module     │
  │              │                    │              │                    │              │
  │ - Send email │                    │ - Log event  │                    │ - Update     │
  │ - Send SMS   │                    │ - Update     │                    │   active     │
  │ - Schedule   │                    │   metrics    │                    │   user count │
  │   reminder   │                    │              │                    │              │
  └──────────────┘                    └──────────────┘                    └──────────────┘
         │
         ▼
  ┌──────────────┐
  │   BullMQ     │
  │              │
  │ Jobs:        │
  │ - reminder   │
  │   (24h)      │
  │ - reminder   │
  │   (1h)       │
  └──────────────┘
```

---

## Agent Architecture

### Agent Orchestrator

```typescript
// Agent orchestrator manages all AI agents
class AgentOrchestrator {
  private agents: Map<string, Agent> = new Map();

  constructor(
    private eventBus: EventBus,
    private aiService: AIService,
  ) {
    this.registerAgents();
    this.setupEventSubscriptions();
  }

  private registerAgents(): void {
    this.agents.set('scheduling-fairness', new SchedulingFairnessAgent());
    this.agents.set('capacity-limits', new CapacityLimitsAgent());
    this.agents.set('payment-enforcement', new PaymentEnforcementAgent());
    this.agents.set('feedback-enhancement', new FeedbackEnhancementAgent());
    this.agents.set('progress-insights', new ProgressInsightsAgent());
    this.agents.set('notification-smart', new SmartNotificationAgent());
    this.agents.set('video-analysis', new VideoAnalysisAgent());
    this.agents.set('no-show-detection', new NoShowDetectionAgent());
  }

  private setupEventSubscriptions(): void {
    // Feedback Enhancement Agent
    this.eventBus.subscribe('feedback.created', async (event) => {
      await this.agents.get('feedback-enhancement')?.process(event);
    });

    // Progress Insights Agent
    this.eventBus.subscribe('feedback.published', async (event) => {
      await this.agents.get('progress-insights')?.process(event);
    });

    // Video Analysis Agent
    this.eventBus.subscribe('media.uploaded', async (event) => {
      if (event.payload.type === 'video') {
        await this.agents.get('video-analysis')?.process(event);
      }
    });

    // Capacity Agent
    this.eventBus.subscribe('booking.confirmed', async (event) => {
      await this.agents.get('capacity-limits')?.process(event);
    });

    // No-Show Agent
    this.eventBus.subscribePattern('booking.*', async (event) => {
      await this.agents.get('no-show-detection')?.process(event);
    });
  }
}

// Base agent interface
interface Agent {
  name: string;
  process(event: DomainEvent): Promise<AgentResult>;
  getGuardrails(): Guardrail[];
}

interface AgentResult {
  success: boolean;
  actions: AgentAction[];
  reasoning?: string;
}

interface AgentAction {
  type: 'update' | 'notify' | 'flag' | 'block';
  target: string;
  data: Record<string, unknown>;
}

interface Guardrail {
  name: string;
  check: (action: AgentAction) => boolean;
  errorMessage: string;
}
```

### Agent Implementation Example

```typescript
// Feedback Enhancement Agent
class FeedbackEnhancementAgent implements Agent {
  name = 'feedback-enhancement';

  constructor(
    private aiService: AIService,
    private feedbackService: FeedbackService,
  ) {}

  async process(event: DomainEvent): Promise<AgentResult> {
    const feedback = await this.feedbackService.getFeedback(event.payload.feedbackId);

    // Check guardrails before processing
    for (const guardrail of this.getGuardrails()) {
      if (!guardrail.check({ type: 'update', target: 'feedback', data: feedback })) {
        return { success: false, actions: [], reasoning: guardrail.errorMessage };
      }
    }

    const actions: AgentAction[] = [];

    // 1. Transcribe voice if present
    if (feedback.voiceNoteUrl) {
      const transcription = await this.aiService.transcribe(feedback.voiceNoteUrl);
      actions.push({
        type: 'update',
        target: `feedback:${feedback.id}`,
        data: { transcription },
      });
    }

    // 2. Enhance text
    const enhanced = await this.aiService.enhanceFeedback({
      rawNotes: feedback.summary || feedback.transcription,
      studentName: feedback.studentName,
      sport: feedback.sportType,
    });

    actions.push({
      type: 'update',
      target: `feedback:${feedback.id}`,
      data: {
        aiEnhancedSummary: enhanced.summary,
        aiInsights: enhanced.insights,
        aiSuggestedDrills: enhanced.drills,
      },
    });

    // 3. Extract skills
    const skills = await this.aiService.extractSkills(enhanced.summary);
    actions.push({
      type: 'update',
      target: `feedback:${feedback.id}`,
      data: { skillsWorked: skills },
    });

    return {
      success: true,
      actions,
      reasoning: 'Feedback enhanced with transcription, text improvement, and skill extraction',
    };
  }

  getGuardrails(): Guardrail[] {
    return [
      {
        name: 'no-fabrication',
        check: (action) => {
          // AI cannot add information coach didn't provide
          return true; // Implemented in AI prompt
        },
        errorMessage: 'AI must not fabricate information',
      },
      {
        name: 'coach-approval-required',
        check: (action) => {
          // Enhanced content must go through coach approval
          return action.data.status !== 'published';
        },
        errorMessage: 'Enhanced feedback must be approved by coach before publishing',
      },
      {
        name: 'no-medical-advice',
        check: (action) => {
          // Check for medical language
          const text = JSON.stringify(action.data).toLowerCase();
          const medicalTerms = ['injury', 'pain', 'hurt', 'doctor', 'medical'];
          return !medicalTerms.some(term => text.includes(term) && text.includes('recommend'));
        },
        errorMessage: 'AI must not provide medical recommendations',
      },
    ];
  }
}
```

---

## Data Flow Patterns

### Read Path (Query)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            READ PATH (QUERY)                                     │
└─────────────────────────────────────────────────────────────────────────────────┘

  Client Request
         │
         ▼
  ┌──────────────┐
  │   API Layer  │
  │              │
  │ - Validate   │
  │ - Auth check │
  └──────────────┘
         │
         ▼
  ┌──────────────┐     ┌──────────────┐
  │    Cache     │────▶│    Redis     │
  │    Check     │     │              │
  └──────────────┘     └──────────────┘
         │                    │
         │ (cache miss)       │ (cache hit)
         ▼                    │
  ┌──────────────┐            │
  │   Service    │            │
  │    Layer     │            │
  └──────────────┘            │
         │                    │
         ▼                    │
  ┌──────────────┐            │
  │  PostgreSQL  │            │
  │              │            │
  └──────────────┘            │
         │                    │
         ▼                    │
  ┌──────────────┐            │
  │  Update      │            │
  │  Cache       │            │
  └──────────────┘            │
         │                    │
         └────────────────────┘
                    │
                    ▼
              Response to Client
```

### Write Path (Command)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            WRITE PATH (COMMAND)                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

  Client Request
         │
         ▼
  ┌──────────────┐
  │   API Layer  │
  │              │
  │ - Validate   │
  │ - Auth check │
  │ - Rate limit │
  └──────────────┘
         │
         ▼
  ┌──────────────┐
  │   Service    │
  │    Layer     │
  │              │
  │ - Business   │
  │   logic      │
  │ - Validation │
  └──────────────┘
         │
         ▼
  ┌─────────────────────────────────────────────────────┐
  │                     Transaction                      │
  │                                                      │
  │  ┌──────────────┐      ┌──────────────┐            │
  │  │  PostgreSQL  │      │  Event Log   │            │
  │  │  (Write)     │      │  (Append)    │            │
  │  └──────────────┘      └──────────────┘            │
  │                                                      │
  └─────────────────────────────────────────────────────┘
         │
         ├─── Invalidate Cache ──────────▶ Redis
         │
         └─── Publish Event ─────────────▶ Event Bus
                                               │
                            ┌──────────────────┼──────────────────┐
                            ▼                  ▼                  ▼
                     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
                     │Notification  │  │  Analytics   │  │    Agent     │
                     │   Handler    │  │   Handler    │  │   Handler    │
                     └──────────────┘  └──────────────┘  └──────────────┘
```

---

## Security Architecture

### Authentication Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          AUTHENTICATION FLOW                                     │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
  │    Client    │────────▶│    Clerk     │────────▶│  SlotBase    │
  │              │         │   (Auth)     │         │     API      │
  └──────────────┘         └──────────────┘         └──────────────┘
         │                        │                        │
         │ 1. Login request       │                        │
         │────────────────────────▶                        │
         │                        │                        │
         │ 2. JWT Token           │                        │
         │◀────────────────────────                        │
         │                        │                        │
         │ 3. API request + JWT   │                        │
         │─────────────────────────────────────────────────▶
         │                        │                        │
         │                        │ 4. Verify JWT          │
         │                        │◀───────────────────────│
         │                        │                        │
         │                        │ 5. Token valid         │
         │                        │───────────────────────▶│
         │                        │                        │
         │                        │                        │ 6. Check permissions
         │                        │                        │    (local DB)
         │                        │                        │
         │ 7. Response            │                        │
         │◀─────────────────────────────────────────────────
```

### Authorization Matrix

```typescript
// Permission checks are performed at multiple layers

// 1. API Layer - Basic auth check
@UseGuards(AuthGuard)
@Controller('bookings')
class BookingController {

  // 2. Endpoint level - Permission check
  @RequirePermissions('booking:create')
  @Post()
  async createBooking(@Body() dto: CreateBookingDto) {}

  // 3. Service level - Resource-level check
  async createBooking(userId: string, dto: CreateBookingDto) {
    // Check user can book at this facility
    await this.authService.checkFacilityAccess(userId, dto.facilityId);

    // Check entitlements
    await this.subscriptionService.checkEntitlement(dto.facilityId, 'booking:create');
  }
}
```

---

## Deployment Architecture

### Phase 1: Single Region

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        PHASE 1: RAILWAY DEPLOYMENT                               │
└─────────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │   Cloudflare    │
                         │      CDN        │
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
           ┌─────────────────┐        ┌─────────────────┐
           │    Vercel       │        │    Railway      │
           │   (Next.js)     │        │   (NestJS)      │
           │                 │        │                 │
           │   Web App       │        │   API Server    │
           │   Admin App     │        │   Worker        │
           └─────────────────┘        └────────┬────────┘
                                               │
                              ┌────────────────┼────────────────┐
                              │                │                │
                              ▼                ▼                ▼
                    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
                    │   Railway    │  │   Upstash    │  │     R2       │
                    │  PostgreSQL  │  │    Redis     │  │   Storage    │
                    └──────────────┘  └──────────────┘  └──────────────┘
```

### Phase 3: Multi-Region AWS

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         PHASE 3: AWS DEPLOYMENT                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │   CloudFront    │
                         │      CDN        │
                         └────────┬────────┘
                                  │
                         ┌────────┴────────┐
                         │       ALB       │
                         │ (Load Balancer) │
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │   ECS Fargate   │ │   ECS Fargate   │ │   ECS Fargate   │
    │   (API - 1)     │ │   (API - 2)     │ │   (Worker)      │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │      RDS        │ │  ElastiCache    │ │       S3        │
    │   PostgreSQL    │ │     Redis       │ │    Storage      │
    │   (Multi-AZ)    │ │   (Cluster)     │ │                 │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## Scalability Considerations

### Database Scaling Strategy

```
Phase 1: Single PostgreSQL instance
         └── Vertical scaling (upgrade instance)

Phase 2: Read replicas
         ├── Primary: Writes
         └── Replica: Reads (reporting, analytics)

Phase 3: Sharding by facility
         ├── Shard 1: Facilities A-M
         └── Shard 2: Facilities N-Z

Alternative: Managed (e.g., CockroachDB, PlanetScale)
```

### Caching Strategy

```typescript
// Cache layers
const CACHE_CONFIG = {
  // L1: In-memory (per instance)
  memory: {
    maxSize: '100mb',
    ttl: '1m',
    items: ['user-session', 'permissions'],
  },

  // L2: Redis (shared)
  redis: {
    items: {
      'facility-settings': { ttl: '5m' },
      'resource-availability': { ttl: '30s' },
      'user-profile': { ttl: '5m' },
      'subscription-status': { ttl: '1m' },
    },
  },

  // Cache invalidation patterns
  invalidation: {
    'booking.created': ['resource-availability:*'],
    'booking.cancelled': ['resource-availability:*'],
    'facility.updated': ['facility-settings:*'],
    'subscription.changed': ['subscription-status:*'],
  },
};
```

---

*Last Updated: 2024-01-10*
*Version: 1.0*
