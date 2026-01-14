# SlotBase AI Agents Specification

## Overview

SlotBase uses specialized AI agents to automate tasks, enforce business rules, and enhance user experience. Each agent has clearly defined responsibilities, inputs, outputs, and guardrails.

---

## Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          AGENT ORCHESTRATION LAYER                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                        Agent Orchestrator                                 │  │
│  │                                                                           │  │
│  │  - Routes events to appropriate agents                                   │  │
│  │  - Manages agent lifecycle                                               │  │
│  │  - Enforces guardrails                                                   │  │
│  │  - Logs all agent actions                                                │  │
│  │  - Handles agent failures gracefully                                     │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │                          Event Bus (Redis)                              │    │
│  │                                                                          │    │
│  │  booking.* │ payment.* │ feedback.* │ session.* │ media.* │ user.*     │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│  BUSINESS RULE  │          │   AI-POWERED    │          │   MONITORING    │
│     AGENTS      │          │     AGENTS      │          │     AGENTS      │
│                 │          │                 │          │                 │
│ - Scheduling    │          │ - Feedback      │          │ - No-Show       │
│   Fairness      │          │   Enhancement   │          │   Detection     │
│ - Capacity      │          │ - Progress      │          │ - Payment       │
│   Limits        │          │   Insights      │          │   Enforcement   │
│ - Payment       │          │ - Video         │          │ - Usage         │
│   Enforcement   │          │   Analysis      │          │   Tracking      │
│                 │          │ - Smart         │          │                 │
│                 │          │   Notification  │          │                 │
└─────────────────┘          └─────────────────┘          └─────────────────┘
```

---

## Agent Specifications

### 1. Scheduling Fairness Agent

**Type:** Business Rule Agent
**Priority:** High
**Execution:** Synchronous (blocks booking flow)

#### Purpose
Ensure fair access to popular time slots and prevent gaming of the booking system.

#### Triggers
| Event | Action |
|-------|--------|
| `booking.hold.requested` | Validate fairness before allowing hold |
| `booking.recurring.requested` | Check recurring booking limits |
| `waitlist.join.requested` | Validate waitlist eligibility |

#### Inputs
```typescript
interface SchedulingFairnessInput {
  userId: string;
  facilityId: string;
  resourceId: string;
  requestedSlot: {
    startAt: Date;
    endAt: Date;
  };
  bookingType: 'one_time' | 'recurring';
  userBookingHistory: BookingHistory;
  facilityPolicy: FairnessPolicy;
}

interface FairnessPolicy {
  maxActiveBookingsPerUser: number;
  maxPrimeTimeBookingsPerWeek: number;
  primeTimeDefinition: { start: string; end: string; days: number[] };
  advanceBookingDays: number;
  minTimeBetweenBookings: number; // hours
  antiHoardingEnabled: boolean;
}
```

#### Outputs
```typescript
interface SchedulingFairnessResult {
  allowed: boolean;
  reason?: string;
  waitlistPosition?: number;
  suggestedAlternatives?: Slot[];
  warnings?: string[];
}
```

#### Rules Engine
```typescript
const FAIRNESS_RULES = [
  {
    name: 'max-active-bookings',
    check: (input) => input.userBookingHistory.activeCount < input.facilityPolicy.maxActiveBookingsPerUser,
    message: 'You have reached the maximum number of active bookings',
  },
  {
    name: 'prime-time-limit',
    check: (input) => {
      if (!isPrimeTime(input.requestedSlot, input.facilityPolicy.primeTimeDefinition)) return true;
      return input.userBookingHistory.primeTimeThisWeek < input.facilityPolicy.maxPrimeTimeBookingsPerWeek;
    },
    message: 'You have reached your weekly prime time booking limit',
  },
  {
    name: 'advance-booking-window',
    check: (input) => {
      const daysInAdvance = daysBetween(new Date(), input.requestedSlot.startAt);
      return daysInAdvance <= input.facilityPolicy.advanceBookingDays;
    },
    message: 'This slot is not yet available for booking',
  },
  {
    name: 'anti-hoarding',
    check: (input) => {
      if (!input.facilityPolicy.antiHoardingEnabled) return true;
      // Check for suspicious patterns
      return !detectHoardingPattern(input.userBookingHistory);
    },
    message: 'Booking pattern detected as potentially unfair to other users',
  },
];
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `no-discrimination` | Rules must apply equally to all users at same entitlement level |
| `staff-bypass-audit` | Staff can bypass rules but all bypasses are logged |
| `no-silent-rejection` | Always provide clear reason when booking is denied |
| `preserve-confirmed` | Never cancel confirmed bookings without user notification |
| `consistent-rules` | Same rules apply regardless of request channel (web/mobile/staff) |

#### DO / DO NOT

| DO ✅ | DO NOT ❌ |
|-------|----------|
| Apply consistent rules across all users | Discriminate based on non-entitlement factors |
| Provide clear rejection reasons | Silently block requests |
| Suggest alternatives when denying | Cancel confirmed bookings automatically |
| Log all fairness decisions | Allow unlimited staff bypasses without audit |
| Respect entitlement-based priorities | Expose other users' booking patterns |

---

### 2. Capacity & Limits Agent

**Type:** Business Rule Agent
**Priority:** High
**Execution:** Asynchronous (background monitoring)

#### Purpose
Track active users per facility and enforce subscription limits with appropriate warnings and grace periods.

#### Triggers
| Event | Action |
|-------|--------|
| `booking.confirmed` | Update active user count |
| `subscription.renewed` | Reset billing period counts |
| `cron.daily` | Check all facilities for limit status |

#### Inputs
```typescript
interface CapacityInput {
  facilityId: string;
  subscriptionId: string;
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  entitlements: {
    activePlayerLimit: number;
    gracePercentage: number; // e.g., 10 for 10%
    gracePeriodDays: number;
  };
}
```

#### Outputs
```typescript
interface CapacityResult {
  activeUsers: number;
  limit: number;
  percentUsed: number;
  status: 'under_limit' | 'warning' | 'at_limit' | 'grace_period' | 'over_limit';
  daysInGrace?: number;
  actions: CapacityAction[];
}

interface CapacityAction {
  type: 'notify_admin' | 'notify_user' | 'block_new_users' | 'suggest_upgrade';
  target: string;
  message: string;
  data?: Record<string, unknown>;
}
```

#### Active User Counting Logic
```typescript
function countActiveUsers(facilityId: string, periodStart: Date, periodEnd: Date): number {
  // Active user = made OR participated in at least one booking this period

  const activeUserIds = new Set<string>();

  const bookings = await getBookings(facilityId, periodStart, periodEnd, {
    statuses: ['confirmed', 'checked_in', 'completed'],
  });

  for (const booking of bookings) {
    // Add booker
    activeUserIds.add(booking.bookerId);

    // Add all participants
    for (const participant of booking.participants) {
      activeUserIds.add(participant.userId);
    }
  }

  // Exclude cancelled-only users
  // A user who only has cancelled bookings is NOT active

  return activeUserIds.size;
}
```

#### Status Thresholds
```typescript
const CAPACITY_THRESHOLDS = {
  warning: 0.80,      // 80% - Send warning notification
  at_limit: 1.00,     // 100% - Block new users
  grace: 1.10,        // 110% - Grace period (allow existing + 10%)
  hard_limit: 1.10,   // After grace period expires
};
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `never-block-existing` | Never block existing active users from booking |
| `warning-before-action` | Always send warning at 80% before any blocking |
| `grace-period-required` | Provide minimum 7-day grace period before hard limit |
| `accurate-counting` | Only count truly active users (with actual bookings) |
| `no-auto-upgrade` | Never automatically upgrade subscription tier |

#### DO / DO NOT

| DO ✅ | DO NOT ❌ |
|-------|----------|
| Send warnings at 80%, 90%, 100% | Block existing active users |
| Provide clear upgrade paths | Count registered-only users |
| Allow grace period before hard limit | Auto-upgrade without consent |
| Track daily active user counts | Share usage across facilities |
| Suggest right-sized plans | Penalize sudden legitimate growth |

---

### 3. Feedback Enhancement Agent

**Type:** AI-Powered Agent
**Priority:** Medium
**Execution:** Asynchronous (background processing)

#### Purpose
Assist coaches in creating high-quality session feedback with AI-powered transcription, enhancement, and skill extraction.

#### Triggers
| Event | Action |
|-------|--------|
| `feedback.created` | Start enhancement pipeline |
| `feedback.voice_uploaded` | Transcribe voice note |
| `feedback.updated` | Re-enhance if significant changes |

#### Inputs
```typescript
interface FeedbackEnhancementInput {
  feedbackId: string;
  sessionId: string;
  coachId: string;
  studentId: string;
  sportType: string;

  // Raw content
  rawNotes?: string;
  voiceNoteUrl?: string;

  // Ratings
  ratings: {
    overall?: number;
    effort?: number;
    focus?: number;
  };

  // Context
  previousSessions: SessionSummary[];
  studentSkillLevels: SkillLevel[];
  drillLibrary: Drill[];
}
```

#### Outputs
```typescript
interface FeedbackEnhancementResult {
  // Transcription
  transcription?: string;

  // Enhanced content
  enhancedSummary: string;
  parentFriendlySummary: string;

  // Extracted data
  skillsWorked: ExtractedSkill[];
  keyInsights: string[];
  suggestedHomework: string[];
  suggestedDrills: Drill[];

  // Progress updates
  skillProgressUpdates: SkillProgressUpdate[];

  // Metadata
  processingTime: number;
  modelUsed: string;
  confidence: number;
}

interface ExtractedSkill {
  skillId: string;
  skillName: string;
  status: 'worked_on' | 'improved' | 'needs_work' | 'mastered';
  notes: string;
}
```

#### AI Pipeline
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        FEEDBACK ENHANCEMENT PIPELINE                             │
└─────────────────────────────────────────────────────────────────────────────────┘

Input (voice/text)
       │
       ▼
┌──────────────────┐
│ 1. TRANSCRIPTION │  Model: Whisper
│    (if voice)    │  Latency: ~5s per minute
│                  │
│  Voice → Text    │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ 2. ENHANCEMENT   │  Model: GPT-4 Turbo
│                  │
│  - Fix grammar   │  Prompt: feedback_enhancement_v1
│  - Add structure │
│  - Keep voice    │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ 3. EXTRACTION    │  Model: GPT-4 Turbo
│                  │
│  - Skills worked │  Prompt: skill_extraction_v1
│  - Key points    │
│  - Progress      │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ 4. SKILL MAPPING │  Method: Embedding similarity
│                  │
│  Map extracted   │  Model: text-embedding-3-small
│  skills to       │
│  skill taxonomy  │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ 5. DRILL RECS    │  Method: RAG
│                  │
│  Find relevant   │  Vector DB: pgvector
│  drills for      │
│  skill gaps      │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ 6. PARENT SUMMARY│  Model: GPT-3.5 Turbo
│                  │
│  Create friendly │  Prompt: parent_summary_v1
│  summary for     │
│  parents         │
└──────────────────┘
       │
       ▼
Output (enhanced feedback)
```

#### Prompts

```typescript
const PROMPTS = {
  feedback_enhancement: `
You are an assistant helping sports coaches write better session feedback for parents.

CONTEXT:
- Sport: {{sport_type}}
- Student: {{student_name}}, age {{student_age}}
- Session type: {{session_type}}
- Previous sessions summary: {{previous_summary}}

COACH'S RAW NOTES:
{{raw_notes}}

TASK:
1. Clean up grammar and structure while keeping the coach's natural voice
2. Organize into clear sections: What We Worked On, Progress Notes, Areas for Improvement
3. Keep it concise but informative
4. Ensure feedback is constructive and encouraging
5. NEVER add information the coach didn't provide
6. NEVER make medical or injury assessments

OUTPUT FORMAT:
{
  "enhanced_summary": "...",
  "key_points": ["...", "..."],
  "areas_for_improvement": ["...", "..."],
  "encouragement": "..."
}
`,

  skill_extraction: `
Extract skills mentioned in this coaching session feedback.

SPORT: {{sport_type}}
SKILL TAXONOMY: {{skill_taxonomy}}

FEEDBACK TEXT:
{{feedback_text}}

For each skill mentioned, identify:
1. The skill name (must match taxonomy or be a reasonable sub-skill)
2. Whether it was: worked_on, improved, needs_work, or mastered
3. Any specific notes about this skill

OUTPUT FORMAT:
{
  "skills": [
    {
      "name": "Serve Toss",
      "status": "improved",
      "notes": "Getting more consistent, needs more height"
    }
  ]
}
`,

  parent_summary: `
Create a brief, parent-friendly summary of this coaching session feedback.

STUDENT NAME: {{student_name}}
FULL FEEDBACK: {{enhanced_summary}}

Guidelines:
- Keep it to 2-3 sentences
- Highlight one positive and one area for practice
- Be encouraging and clear
- Avoid technical jargon

OUTPUT: A single paragraph summary.
`,
};
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `no-fabrication` | Never add information coach didn't provide |
| `no-medical` | Never make medical or injury assessments |
| `no-comparison` | Never compare students to each other |
| `coach-approval` | Always require coach approval before publishing |
| `preserve-voice` | Maintain coach's natural tone and style |
| `student-privacy` | Never expose student data in logs |

#### DO / DO NOT

| DO ✅ | DO NOT ❌ |
|-------|----------|
| Transcribe voice accurately | Fabricate information |
| Enhance while preserving voice | Change meaning of feedback |
| Extract skills objectively | Make medical assessments |
| Suggest relevant drills | Compare students by name |
| Generate parent-friendly summaries | Publish without coach approval |
| Log processing for debugging | Log PII or student details |

---

### 4. Progress Insights Agent

**Type:** AI-Powered Agent
**Priority:** Medium
**Execution:** Asynchronous (scheduled + on-demand)

#### Purpose
Generate meaningful progress analytics and natural language summaries for players and parents.

#### Triggers
| Event | Action |
|-------|--------|
| `feedback.published` | Update progress metrics |
| `cron.weekly` | Generate weekly progress summaries |
| `progress.view.requested` | Generate on-demand insights |

#### Inputs
```typescript
interface ProgressInsightsInput {
  playerId: string;
  sportType: string;
  timeRange: {
    start: Date;
    end: Date;
  };

  // Historical data
  sessions: SessionRecord[];
  skillProgress: SkillProgressRecord[];
  achievements: Achievement[];

  // Benchmarks (anonymized)
  benchmarks: {
    avgImprovementRate: number;
    typicalTimeToLevel: Record<string, number>;
  };
}
```

#### Outputs
```typescript
interface ProgressInsightsResult {
  // Summary
  naturalLanguageSummary: string;

  // Metrics
  metrics: {
    totalSessions: number;
    totalHours: number;
    currentStreak: number;
    longestStreak: number;
  };

  // Skill analysis
  skillAnalysis: {
    improving: SkillTrend[];
    stable: SkillTrend[];
    needsAttention: SkillTrend[];
  };

  // AI insights
  insights: {
    highlights: string[];
    concerns: string[];
    predictions: Prediction[];
    recommendations: string[];
  };

  // Achievements
  newAchievements: Achievement[];
  upcomingMilestones: Milestone[];
}

interface SkillTrend {
  skillId: string;
  skillName: string;
  currentLevel: number;
  previousLevel: number;
  changePercent: number;
  trend: 'improving' | 'stable' | 'declining';
  predictedTimeToTarget?: number; // days
}
```

#### Insight Generation Prompt
```typescript
const PROGRESS_INSIGHTS_PROMPT = `
Generate progress insights for a sports student.

STUDENT: {{student_name}}
SPORT: {{sport_type}}
TIME PERIOD: {{time_range}}

SESSION DATA:
{{session_summary}}

SKILL PROGRESS:
{{skill_progress}}

BENCHMARKS (anonymized averages):
{{benchmarks}}

Generate:
1. A 2-3 sentence natural language summary suitable for a parent
2. Key highlights (positive observations)
3. Areas of concern (if any, phrase constructively)
4. Predictions (realistic, don't create pressure)
5. Recommendations for focus areas

Guidelines:
- Be encouraging but realistic
- Compare to anonymized benchmarks, NOT other students
- Never suggest injury or medical concerns
- Focus on effort and consistency, not just results
- Predictions should be ranges, not exact dates

OUTPUT FORMAT:
{
  "summary": "...",
  "highlights": ["...", "..."],
  "concerns": ["..."],
  "predictions": [{"metric": "...", "prediction": "...", "confidence": "..."}],
  "recommendations": ["...", "..."]
}
`;
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `no-named-comparison` | Never compare to named individuals |
| `constructive-language` | All concerns phrased constructively |
| `no-pressure-predictions` | Predictions are ranges, not deadlines |
| `no-medical` | Never suggest injury or training intensity changes |
| `anonymized-benchmarks` | Benchmarks are aggregate, never individual |

---

### 5. Video Analysis Agent

**Type:** AI-Powered Agent
**Priority:** Low (Phase 3)
**Execution:** Asynchronous (background processing)

#### Purpose
Extract insights from coaching videos to enhance feedback with visual analysis.

#### Triggers
| Event | Action |
|-------|--------|
| `media.uploaded` (type: video) | Queue for analysis |
| `media.processing.complete` | Start AI analysis |

#### Inputs
```typescript
interface VideoAnalysisInput {
  mediaId: string;
  feedbackId: string;
  videoUrl: string;
  sportType: string;
  focusAreas?: string[]; // What to look for

  // Video metadata
  duration: number;
  resolution: string;
}
```

#### Outputs
```typescript
interface VideoAnalysisResult {
  // Key moments
  keyMoments: KeyMoment[];

  // Technique observations
  observations: TechniqueObservation[];

  // Transcription (if coach speaking)
  audioTranscription?: string;

  // Highlight reel
  highlightClipUrl?: string;
  highlightTimestamps: number[];

  // Processing metadata
  processingTime: number;
  confidence: number;
}

interface KeyMoment {
  timestamp: number;
  type: 'serve' | 'shot' | 'drill' | 'instruction' | 'celebration';
  description: string;
  thumbnailUrl: string;
}

interface TechniqueObservation {
  timestamp: number;
  area: string; // 'footwork', 'grip', 'stance', etc.
  observation: string;
  suggestion?: string;
  confidence: number;
}
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `no-medical` | Never diagnose injuries or pain |
| `observations-not-judgments` | Provide observations, not definitive judgments |
| `no-cross-comparison` | Never compare to other students' videos |
| `retention-policy` | Delete raw video after processing per policy |
| `consent-required` | Only process videos with proper consent |
| `coach-override` | Coach can reject/modify any AI observation |

---

### 6. Smart Notification Agent

**Type:** AI-Powered Agent
**Priority:** Medium
**Execution:** Asynchronous

#### Purpose
Optimize notification timing and personalization for better engagement.

#### Triggers
| Event | Action |
|-------|--------|
| `notification.scheduled` | Optimize send time |
| `notification.template.render` | Personalize content |
| `cron.hourly` | Process scheduled notifications |

#### Capabilities
```typescript
interface SmartNotificationCapabilities {
  // Timing optimization
  optimizeSendTime(userId: string, notificationType: string): Promise<Date>;

  // Content personalization
  personalizeContent(template: string, context: NotificationContext): Promise<string>;

  // Channel selection
  selectChannel(userId: string, priority: string): Promise<'email' | 'sms' | 'push'>;

  // Batching
  shouldBatch(userId: string, notifications: Notification[]): Promise<boolean>;
}
```

#### Send Time Optimization
```typescript
interface SendTimeOptimization {
  userId: string;

  // User engagement history
  openRates: {
    byHour: Record<number, number>;
    byDay: Record<number, number>;
  };

  // User preferences
  quietHours?: { start: string; end: string };
  timezone: string;

  // Notification type patterns
  typePatterns: {
    reminder: { bestHour: number; avgOpenRate: number };
    update: { bestHour: number; avgOpenRate: number };
  };
}

function calculateOptimalSendTime(data: SendTimeOptimization): Date {
  // Consider:
  // 1. Historical open rates by hour
  // 2. Quiet hours (never send during)
  // 3. Notification type patterns
  // 4. Timezone
  // 5. Don't send too early or too late
}
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `respect-preferences` | Always honor user notification preferences |
| `quiet-hours` | Never send during user's quiet hours (except critical) |
| `no-spam` | Maximum N notifications per day per user |
| `unsubscribe-honored` | Immediately honor unsubscribe requests |
| `critical-immediate` | Critical notifications (cancellations) sent immediately |

---

### 7. Payment Enforcement Agent

**Type:** Business Rule Agent
**Priority:** High
**Execution:** Synchronous + Asynchronous

#### Purpose
Ensure revenue collection while handling edge cases gracefully.

#### Triggers
| Event | Action |
|-------|--------|
| `booking.confirmed` | Verify payment captured |
| `subscription.renewal.due` | Attempt renewal charge |
| `payment.failed` | Initiate retry sequence |
| `cron.daily` | Check aging receivables |

#### Payment Retry Logic
```typescript
const PAYMENT_RETRY_SCHEDULE = [
  { attempt: 1, delayHours: 0, action: 'charge' },
  { attempt: 2, delayHours: 24, action: 'retry' },
  { attempt: 3, delayHours: 72, action: 'retry' },
  { attempt: 4, delayHours: 168, action: 'final_retry' }, // 7 days
  { attempt: 5, delayHours: 336, action: 'suspend' }, // 14 days
];

interface PaymentRetryResult {
  success: boolean;
  attempt: number;
  nextAction: 'retry' | 'notify' | 'suspend' | 'cancel';
  nextAttemptAt?: Date;
}
```

#### Offline Payment Aging
```typescript
const OFFLINE_AGING_ACTIONS = [
  { daysUnpaid: 7, action: 'reminder_email' },
  { daysUnpaid: 14, action: 'reminder_email_urgent' },
  { daysUnpaid: 30, action: 'flag_for_review' },
  { daysUnpaid: 60, action: 'restrict_new_bookings' },
];
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `no-surprise-charges` | Never charge without clear authorization |
| `retry-limits` | Maximum 5 retry attempts |
| `notification-before-action` | Always notify before restricting access |
| `grace-periods` | Provide reasonable grace periods |
| `audit-trail` | Log all payment actions |
| `pci-compliance` | Never store full card numbers |

---

### 8. No-Show Detection Agent

**Type:** Monitoring Agent
**Priority:** Medium
**Execution:** Scheduled (every minute during operating hours)

#### Purpose
Detect no-shows, apply appropriate policies, and manage strike system.

#### Triggers
| Event | Action |
|-------|--------|
| `cron.minute` | Check for overdue check-ins |
| `booking.checked_in` | Clear no-show timer |
| `booking.time.passed` | Final no-show determination |

#### No-Show Logic
```typescript
interface NoShowDetection {
  bookingId: string;
  scheduledStart: Date;
  gracePeriodMinutes: number;
  currentTime: Date;

  // Check-in status
  checkedIn: boolean;
  checkedInAt?: Date;

  // User history
  userNoShowHistory: {
    count: number;
    lastNoShow?: Date;
    strikes: number;
  };

  // Facility policy
  policy: {
    gracePeriodMinutes: number;
    noShowFeePercent: number;
    strikeThreshold: number;
    strikeResetDays: number;
    autoRestrict: boolean;
  };
}

function determineNoShowStatus(data: NoShowDetection): NoShowResult {
  const gracePeriodEnd = addMinutes(data.scheduledStart, data.gracePeriodMinutes);

  if (data.checkedIn) {
    return { status: 'checked_in', action: 'none' };
  }

  if (data.currentTime < data.scheduledStart) {
    return { status: 'upcoming', action: 'none' };
  }

  if (data.currentTime < gracePeriodEnd) {
    return { status: 'grace_period', action: 'none' };
  }

  // Past grace period, not checked in
  return {
    status: 'no_show',
    action: 'apply_policy',
    fee: calculateNoShowFee(data),
    newStrikeCount: data.userNoShowHistory.strikes + 1,
    restrictBooking: data.userNoShowHistory.strikes + 1 >= data.policy.strikeThreshold,
  };
}
```

#### Guardrails

| Guardrail | Description |
|-----------|-------------|
| `grace-period-required` | Always allow grace period before marking no-show |
| `notification-before-marking` | Send warning during grace period |
| `forgiveness-option` | Facility can forgive no-shows |
| `no-retroactive-policy` | Don't apply new policy to past no-shows |
| `appeal-process` | Users can dispute no-shows |

---

## Agent Orchestrator Implementation

```typescript
// agents/orchestrator.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { EventBus } from '../events/event-bus';

@Injectable()
export class AgentOrchestrator implements OnModuleInit {
  private agents: Map<string, Agent> = new Map();
  private eventSubscriptions: Map<string, string[]> = new Map();

  constructor(
    private eventBus: EventBus,
    private schedulingFairnessAgent: SchedulingFairnessAgent,
    private capacityLimitsAgent: CapacityLimitsAgent,
    private feedbackEnhancementAgent: FeedbackEnhancementAgent,
    private progressInsightsAgent: ProgressInsightsAgent,
    private smartNotificationAgent: SmartNotificationAgent,
    private paymentEnforcementAgent: PaymentEnforcementAgent,
    private noShowDetectionAgent: NoShowDetectionAgent,
  ) {}

  async onModuleInit() {
    this.registerAgents();
    this.setupEventSubscriptions();
  }

  private registerAgents() {
    this.agents.set('scheduling-fairness', this.schedulingFairnessAgent);
    this.agents.set('capacity-limits', this.capacityLimitsAgent);
    this.agents.set('feedback-enhancement', this.feedbackEnhancementAgent);
    this.agents.set('progress-insights', this.progressInsightsAgent);
    this.agents.set('smart-notification', this.smartNotificationAgent);
    this.agents.set('payment-enforcement', this.paymentEnforcementAgent);
    this.agents.set('no-show-detection', this.noShowDetectionAgent);
  }

  private setupEventSubscriptions() {
    // Define event -> agent mappings
    const subscriptions = {
      'booking.hold.requested': ['scheduling-fairness'],
      'booking.confirmed': ['capacity-limits', 'smart-notification'],
      'booking.cancelled': ['smart-notification'],
      'feedback.created': ['feedback-enhancement'],
      'feedback.published': ['progress-insights', 'smart-notification'],
      'media.uploaded': ['feedback-enhancement'],
      'payment.failed': ['payment-enforcement'],
      'subscription.renewal.due': ['payment-enforcement'],
    };

    for (const [eventType, agentNames] of Object.entries(subscriptions)) {
      this.eventBus.subscribe(eventType, async (event) => {
        await this.routeToAgents(event, agentNames);
      });
    }
  }

  private async routeToAgents(event: DomainEvent, agentNames: string[]) {
    for (const agentName of agentNames) {
      const agent = this.agents.get(agentName);
      if (!agent) continue;

      try {
        // Log agent invocation
        await this.logAgentInvocation(agentName, event);

        // Check guardrails before processing
        const guardrailResult = await this.checkGuardrails(agent, event);
        if (!guardrailResult.passed) {
          await this.logGuardrailViolation(agentName, event, guardrailResult);
          continue;
        }

        // Process event
        const result = await agent.process(event);

        // Execute actions
        await this.executeAgentActions(agentName, result.actions);

        // Log result
        await this.logAgentResult(agentName, event, result);
      } catch (error) {
        await this.handleAgentError(agentName, event, error);
      }
    }
  }

  private async checkGuardrails(agent: Agent, event: DomainEvent): Promise<GuardrailResult> {
    const guardrails = agent.getGuardrails();

    for (const guardrail of guardrails) {
      if (!guardrail.check(event)) {
        return {
          passed: false,
          failedGuardrail: guardrail.name,
          message: guardrail.errorMessage,
        };
      }
    }

    return { passed: true };
  }

  private async executeAgentActions(agentName: string, actions: AgentAction[]) {
    for (const action of actions) {
      switch (action.type) {
        case 'update':
          await this.executeUpdate(action);
          break;
        case 'notify':
          await this.executeNotify(action);
          break;
        case 'flag':
          await this.executeFlag(action);
          break;
        case 'block':
          await this.executeBlock(action);
          break;
      }
    }
  }
}
```

---

## Agent Configuration

```typescript
// config/agents.config.ts
export const AGENT_CONFIG = {
  'scheduling-fairness': {
    enabled: true,
    priority: 'high',
    timeout: 5000, // 5 seconds
    retryOnFailure: false, // Synchronous, don't retry
  },
  'capacity-limits': {
    enabled: true,
    priority: 'high',
    timeout: 10000,
    retryOnFailure: true,
    maxRetries: 3,
  },
  'feedback-enhancement': {
    enabled: true,
    priority: 'medium',
    timeout: 60000, // 60 seconds for AI processing
    retryOnFailure: true,
    maxRetries: 2,
    rateLimit: {
      maxPerMinute: 10,
      maxPerHour: 100,
    },
  },
  'progress-insights': {
    enabled: true,
    priority: 'low',
    timeout: 30000,
    retryOnFailure: true,
    maxRetries: 3,
    scheduling: {
      weekly: { day: 'monday', hour: 9 },
    },
  },
  'video-analysis': {
    enabled: false, // Phase 3
    priority: 'low',
    timeout: 300000, // 5 minutes
    retryOnFailure: true,
    maxRetries: 1,
  },
};
```

---

## Monitoring & Observability

```typescript
// Agent metrics to track
const AGENT_METRICS = {
  // Performance
  'agent.invocation.count': 'Counter',
  'agent.processing.duration': 'Histogram',
  'agent.success.rate': 'Gauge',

  // Errors
  'agent.error.count': 'Counter',
  'agent.guardrail.violation.count': 'Counter',

  // AI-specific
  'agent.ai.token.usage': 'Counter',
  'agent.ai.cost': 'Counter',
  'agent.ai.latency': 'Histogram',

  // Business
  'agent.action.count': 'Counter',
  'agent.feedback.enhanced.count': 'Counter',
  'agent.noshow.detected.count': 'Counter',
};
```

---

## Agent Priority & Conflict Resolution

When multiple agents respond to the same event or make conflicting decisions, the system must resolve conflicts predictably.

### Priority Levels

| Agent | Priority | Type | Blocking |
|-------|----------|------|----------|
| Scheduling Fairness | 100 | Sync | Yes - blocks booking |
| Capacity Limits | 90 | Sync | Yes - can block users |
| Payment Enforcement | 80 | Sync | Yes - blocks confirmed bookings |
| No-Show Detection | 70 | Async | No - flags only |
| Feedback Enhancement | 50 | Async | No - background |
| Progress Insights | 40 | Async | No - scheduled |
| Smart Notification | 30 | Async | No - best-effort |
| Video Analysis | 20 | Async | No - Phase 3 |

### Conflict Resolution Rules

```typescript
interface AgentDecision {
  agentId: string;
  priority: number;
  action: 'allow' | 'block' | 'warn' | 'modify';
  reason: string;
  data?: any;
}

function resolveConflicts(decisions: AgentDecision[]): AgentDecision {
  // Sort by priority (highest first)
  const sorted = decisions.sort((a, b) => b.priority - a.priority);

  // Blocking decisions take precedence
  const blockers = sorted.filter(d => d.action === 'block');
  if (blockers.length > 0) {
    // Return highest priority blocker
    return blockers[0];
  }

  // Then check for modifications
  const modifiers = sorted.filter(d => d.action === 'modify');
  if (modifiers.length > 0) {
    // Merge modifications in priority order
    return mergeModifications(modifiers);
  }

  // Collect all warnings
  const warnings = sorted.filter(d => d.action === 'warn');

  // Return allow with warnings
  return {
    agentId: 'orchestrator',
    priority: 100,
    action: 'allow',
    reason: 'All agents approved',
    data: { warnings: warnings.map(w => w.reason) }
  };
}
```

### Conflict Scenarios

#### Scenario 1: Capacity vs Fairness

```
Event: booking.hold.requested
├── Scheduling Fairness Agent (Priority 100)
│   └── Decision: ALLOW (user within limits)
└── Capacity Limits Agent (Priority 90)
    └── Decision: BLOCK (facility at player limit)

Result: BLOCK (Capacity wins because it's a blocking decision)
```

#### Scenario 2: Multiple Warnings

```
Event: booking.hold.requested
├── Scheduling Fairness Agent (Priority 100)
│   └── Decision: WARN (approaching prime time limit)
├── Payment Enforcement Agent (Priority 80)
│   └── Decision: WARN (outstanding balance)
└── No-Show Detection Agent (Priority 70)
    └── Decision: WARN (high no-show rate)

Result: ALLOW with all warnings displayed to user
```

#### Scenario 3: Blocking Agent Timeout

```typescript
async function executeWithFallback(
  agent: Agent,
  input: any,
  timeout: number
): Promise<AgentDecision> {
  try {
    const result = await Promise.race([
      agent.execute(input),
      timeoutPromise(timeout)
    ]);
    return result;
  } catch (error) {
    if (agent.blocking) {
      // Blocking agent timeout = ALLOW (fail-open for UX)
      // But log for investigation
      logger.error(`Blocking agent ${agent.id} timed out`, { error, input });

      return {
        agentId: agent.id,
        priority: agent.priority,
        action: 'allow',
        reason: 'Agent timeout - fail-open',
        data: { timedOut: true }
      };
    }

    // Non-blocking agents just skip
    return null;
  }
}
```

### Orchestrator Implementation

```typescript
@Injectable()
export class AgentOrchestrator {
  private readonly agents: Map<string, Agent>;
  private readonly config: AgentConfig;

  async handleEvent(event: DomainEvent): Promise<OrchestratorResult> {
    const startTime = Date.now();

    // 1. Find relevant agents
    const relevantAgents = this.findAgentsForEvent(event);

    // 2. Separate sync and async agents
    const syncAgents = relevantAgents.filter(a => a.isBlocking);
    const asyncAgents = relevantAgents.filter(a => !a.isBlocking);

    // 3. Execute sync agents in priority order (must complete)
    const syncDecisions: AgentDecision[] = [];
    for (const agent of syncAgents.sort((a, b) => b.priority - a.priority)) {
      const decision = await this.executeWithFallback(
        agent,
        event,
        agent.timeout
      );
      syncDecisions.push(decision);

      // Early exit if blocked
      if (decision.action === 'block') {
        break;
      }
    }

    // 4. Resolve sync conflicts
    const syncResult = resolveConflicts(syncDecisions);

    // 5. If blocked, return immediately
    if (syncResult.action === 'block') {
      return {
        allowed: false,
        reason: syncResult.reason,
        agentId: syncResult.agentId,
        duration: Date.now() - startTime
      };
    }

    // 6. Queue async agents (don't wait)
    for (const agent of asyncAgents) {
      await this.queue.add(`agent:${agent.id}`, {
        event,
        syncResult,
        triggeredAt: new Date()
      });
    }

    // 7. Return result with warnings
    return {
      allowed: true,
      warnings: syncResult.data?.warnings || [],
      duration: Date.now() - startTime
    };
  }
}
```

### Agent Decision Logging

All agent decisions are logged for debugging and audit:

```typescript
interface AgentDecisionLog {
  id: string;
  eventId: string;
  eventType: string;
  agentId: string;
  priority: number;
  decision: 'allow' | 'block' | 'warn' | 'modify' | 'skip' | 'timeout';
  reason: string;
  inputHash: string;      // Hash of input for debugging
  durationMs: number;
  createdAt: Date;
}

// Query example: Find all blocks by Payment Enforcement last 7 days
SELECT * FROM agent_decision_logs
WHERE agent_id = 'payment-enforcement'
  AND decision = 'block'
  AND created_at > NOW() - INTERVAL '7 days'
ORDER BY created_at DESC;
```

---

*Last Updated: 2026-01-11*
*Version: 1.1*
