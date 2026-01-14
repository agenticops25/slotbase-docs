# Error Recovery Workflows

## Overview

This document defines how SlotBase handles failures and recovers from error states. The system prioritizes data consistency and user experience.

**Core Principles:**
- Fail-fast for validation errors
- Fail-open for non-critical features (notifications, analytics)
- Fail-safe for financial operations (prefer retry over loss)
- Always leave audit trail

---

## Booking System Recovery

### 1. Booking Hold Expiration

**Scenario:** User creates a hold but doesn't confirm within 5 minutes.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Hold        │────▶│   5-min      │────▶│   Release    │
│  Created     │     │   Timer      │     │   Slot       │
└──────────────┘     └──────────────┘     └──────────────┘
                            │
                            │ Race condition?
                            ▼
                     ┌──────────────┐
                     │  Confirm     │
                     │  Arrives     │
                     └──────────────┘
```

**Implementation:**

```typescript
// BullMQ job for hold expiration
@Processor('booking-holds')
export class HoldExpirationProcessor {
  @Process('expire-hold')
  async expireHold(job: Job<{ holdId: string }>) {
    const { holdId } = job.data;

    // Use database transaction with row-level locking
    await this.prisma.$transaction(async (tx) => {
      // Lock the hold record
      const hold = await tx.bookingHold.findUnique({
        where: { id: holdId },
        select: { id: true, status: true, expiresAt: true }
      });

      // Already processed?
      if (!hold || hold.status !== 'ACTIVE') {
        return; // Idempotent - already released or converted
      }

      // Still within grace period? (network delay tolerance)
      if (hold.expiresAt > new Date()) {
        // Re-queue with remaining time
        const remainingMs = hold.expiresAt.getTime() - Date.now();
        await this.queue.add('expire-hold', { holdId }, { delay: remainingMs });
        return;
      }

      // Release the hold
      await tx.bookingHold.update({
        where: { id: holdId },
        data: { status: 'EXPIRED', releasedAt: new Date() }
      });

      // Emit event for waitlist processing
      await this.events.emit('booking.hold.expired', { holdId });
    });
  }
}
```

**Race Condition Handling:**

```typescript
// Confirmation arriving during expiration
async confirmBooking(holdId: string, paymentData: PaymentData) {
  return this.prisma.$transaction(async (tx) => {
    // Lock hold with FOR UPDATE
    const hold = await tx.$queryRaw`
      SELECT * FROM booking_holds
      WHERE id = ${holdId}
      FOR UPDATE
    `;

    if (hold.status === 'EXPIRED') {
      throw new GoneException('Hold has expired. Please create a new booking.');
    }

    if (hold.status === 'CONFIRMED') {
      // Return existing booking (idempotent)
      return tx.booking.findFirst({ where: { holdId } });
    }

    // Process confirmation...
    const booking = await tx.booking.create({ ... });

    await tx.bookingHold.update({
      where: { id: holdId },
      data: { status: 'CONFIRMED' }
    });

    return booking;
  }, {
    isolationLevel: 'Serializable' // Strictest isolation
  });
}
```

### 2. Double-Booking Prevention

**Scenario:** Two users try to book the same slot simultaneously.

```typescript
async createBookingHold(input: CreateHoldInput): Promise<BookingHold> {
  return this.prisma.$transaction(async (tx) => {
    // 1. Check for existing holds/bookings (with lock)
    const conflicting = await tx.$queryRaw`
      SELECT id, type FROM (
        SELECT id, 'hold' as type FROM booking_holds
        WHERE resource_id = ${input.resourceId}
          AND status = 'ACTIVE'
          AND start_time < ${input.endTime}
          AND end_time > ${input.startTime}
        UNION ALL
        SELECT id, 'booking' as type FROM bookings
        WHERE resource_id = ${input.resourceId}
          AND status IN ('CONFIRMED', 'CHECKED_IN')
          AND start_time < ${input.endTime}
          AND end_time > ${input.startTime}
      ) conflicts
      FOR UPDATE
    `;

    if (conflicting.length > 0) {
      throw new ConflictException('Slot is no longer available');
    }

    // 2. Create hold atomically
    const hold = await tx.bookingHold.create({
      data: {
        resourceId: input.resourceId,
        userId: input.userId,
        startTime: input.startTime,
        endTime: input.endTime,
        status: 'ACTIVE',
        expiresAt: addMinutes(new Date(), 5)
      }
    });

    // 3. Schedule expiration
    await this.queue.add('expire-hold', { holdId: hold.id }, {
      delay: 5 * 60 * 1000 // 5 minutes
    });

    return hold;
  }, {
    isolationLevel: 'Serializable'
  });
}
```

### 3. Booking Cancellation Recovery

**Scenario:** Cancellation partially completes (booking cancelled but refund fails).

```typescript
async cancelBooking(bookingId: string, userId: string): Promise<CancellationResult> {
  const booking = await this.prisma.booking.findUnique({
    where: { id: bookingId },
    include: { payments: true }
  });

  // Start saga pattern
  const saga = new CancellationSaga(bookingId);

  try {
    // Step 1: Mark booking as cancelling
    await saga.execute('mark-cancelling', async () => {
      await this.prisma.booking.update({
        where: { id: bookingId },
        data: { status: 'CANCELLING' }
      });
    });

    // Step 2: Process refund (if applicable)
    if (booking.payments.length > 0) {
      await saga.execute('process-refund', async () => {
        const refundResult = await this.paymentService.refund(booking);
        return refundResult;
      }, async (error) => {
        // Compensating action: Log failed refund for manual processing
        await this.prisma.refundFailure.create({
          data: {
            bookingId,
            error: error.message,
            status: 'PENDING_MANUAL'
          }
        });
      });
    }

    // Step 3: Complete cancellation
    await saga.execute('complete-cancellation', async () => {
      await this.prisma.booking.update({
        where: { id: bookingId },
        data: {
          status: 'CANCELLED',
          cancelledAt: new Date(),
          cancelledBy: userId
        }
      });
    });

    // Step 4: Notify waitlist
    await saga.execute('notify-waitlist', async () => {
      await this.events.emit('booking.cancelled', { bookingId });
    });

    return { success: true, refundProcessed: booking.payments.length > 0 };

  } catch (error) {
    // Saga handles rollback automatically
    await saga.rollback();

    // Booking stays in CANCELLING for manual review
    await this.alerting.notify('cancellation-failed', { bookingId, error });

    throw new ServiceUnavailableException(
      'Cancellation could not be completed. Our team has been notified.'
    );
  }
}
```

---

## Payment System Recovery

### 1. Offline Payment Recording Failure

**Scenario:** Staff records payment but database write fails.

```typescript
async recordOfflinePayment(input: RecordPaymentInput): Promise<OfflinePayment> {
  // Use idempotency key to prevent duplicates
  const idempotencyKey = `payment:${input.bookingId}:${input.amount}:${Date.now()}`;

  const cached = await this.redis.get(`idem:${idempotencyKey}`);
  if (cached) {
    return JSON.parse(cached);
  }

  // Lock to prevent concurrent recording
  const lock = await this.redlock.lock(`lock:payment:${input.bookingId}`, 10000);

  try {
    const payment = await this.prisma.$transaction(async (tx) => {
      // Check for duplicate
      const existing = await tx.offlinePayment.findFirst({
        where: {
          bookingId: input.bookingId,
          amount: input.amount,
          createdAt: { gt: subMinutes(new Date(), 5) }
        }
      });

      if (existing) {
        return existing; // Duplicate detected
      }

      // Create payment record
      const payment = await tx.offlinePayment.create({
        data: {
          bookingId: input.bookingId,
          amount: input.amount,
          method: input.method,
          recordedBy: input.staffId,
          status: 'RECORDED'
        }
      });

      // Update booking payment status
      await tx.booking.update({
        where: { id: input.bookingId },
        data: { paymentStatus: 'PAID' }
      });

      return payment;
    });

    // Cache for idempotency
    await this.redis.set(
      `idem:${idempotencyKey}`,
      JSON.stringify(payment),
      'EX', 3600
    );

    return payment;

  } finally {
    await lock.unlock();
  }
}
```

### 2. Stripe Webhook Delivery Failure

**Scenario:** Stripe sends webhook but our endpoint is temporarily down.

```typescript
// Stripe automatically retries for up to 3 days
// Our implementation handles eventual delivery

@Post('/webhooks/stripe')
async handleStripeWebhook(
  @Req() req: RawBodyRequest,
  @Headers('stripe-signature') signature: string
) {
  let event: Stripe.Event;

  try {
    event = this.stripe.webhooks.constructEvent(
      req.rawBody,
      signature,
      this.config.stripeWebhookSecret
    );
  } catch (err) {
    throw new BadRequestException(`Webhook signature verification failed`);
  }

  // Idempotency: Check if already processed
  const processed = await this.redis.get(`stripe:event:${event.id}`);
  if (processed) {
    return { received: true, duplicate: true };
  }

  // Process event in transaction
  try {
    await this.processStripeEvent(event);

    // Mark as processed
    await this.redis.set(
      `stripe:event:${event.id}`,
      'processed',
      'EX', 86400 * 7 // 7 days
    );

    return { received: true };

  } catch (error) {
    // Log for retry, don't throw (Stripe will retry anyway)
    this.logger.error('Stripe webhook processing failed', {
      eventId: event.id,
      type: event.type,
      error
    });

    // Store for manual processing if critical
    if (this.isCriticalEvent(event.type)) {
      await this.prisma.webhookFailure.create({
        data: {
          provider: 'stripe',
          eventId: event.id,
          eventType: event.type,
          payload: event as any,
          error: error.message
        }
      });
    }

    // Return 200 to stop Stripe retries (we'll handle manually)
    return { received: true, processingFailed: true };
  }
}
```

### 3. Refund Processing Failure

**Scenario:** Stripe refund API call fails.

```typescript
async processRefund(
  transactionId: string,
  amount: number,
  reason: string
): Promise<RefundResult> {
  const transaction = await this.prisma.transaction.findUnique({
    where: { id: transactionId }
  });

  // Already refunded?
  const existingRefund = await this.prisma.refund.findFirst({
    where: { transactionId, status: 'COMPLETED' }
  });
  if (existingRefund) {
    return { success: true, refund: existingRefund, duplicate: true };
  }

  // Create pending refund record first
  const refund = await this.prisma.refund.create({
    data: {
      transactionId,
      amount,
      reason,
      status: 'PENDING',
      attempts: 0
    }
  });

  try {
    // Call Stripe
    const stripeRefund = await this.stripe.refunds.create({
      payment_intent: transaction.stripePaymentIntentId,
      amount: Math.round(amount * 100),
      reason: 'requested_by_customer'
    }, {
      idempotencyKey: `refund:${refund.id}`
    });

    // Update refund record
    await this.prisma.refund.update({
      where: { id: refund.id },
      data: {
        status: 'COMPLETED',
        stripeRefundId: stripeRefund.id,
        completedAt: new Date()
      }
    });

    return { success: true, refund };

  } catch (error) {
    // Update with failure
    await this.prisma.refund.update({
      where: { id: refund.id },
      data: {
        status: 'FAILED',
        attempts: { increment: 1 },
        lastError: error.message
      }
    });

    // Queue for retry
    await this.queue.add('retry-refund', {
      refundId: refund.id
    }, {
      delay: 5 * 60 * 1000, // 5 minutes
      attempts: 3,
      backoff: { type: 'exponential', delay: 60000 }
    });

    // Alert if critical
    if (amount > 100) {
      await this.alerting.notify('refund-failed', { refundId: refund.id, amount });
    }

    return { success: false, refund, willRetry: true };
  }
}
```

---

## Notification System Recovery

### 1. Email Delivery Failure

**Scenario:** Resend API is temporarily unavailable.

```typescript
@Processor('notifications')
export class NotificationProcessor {
  @Process('send-email')
  async sendEmail(job: Job<EmailJob>) {
    const { to, template, data, notificationId } = job.data;

    try {
      const result = await this.resend.emails.send({
        from: 'SlotBase <noreply@slotbase.com>',
        to,
        subject: this.getSubject(template, data),
        html: await this.renderTemplate(template, data)
      });

      // Update notification record
      await this.prisma.notification.update({
        where: { id: notificationId },
        data: {
          status: 'SENT',
          externalId: result.id,
          sentAt: new Date()
        }
      });

    } catch (error) {
      // Log attempt
      await this.prisma.notificationAttempt.create({
        data: {
          notificationId,
          attemptNumber: job.attemptsMade + 1,
          error: error.message,
          attemptedAt: new Date()
        }
      });

      // Let BullMQ retry (configured with exponential backoff)
      throw error;
    }
  }
}

// Queue configuration
const emailQueue = new Queue('notifications', {
  defaultJobOptions: {
    attempts: 5,
    backoff: {
      type: 'exponential',
      delay: 60000 // Start with 1 minute
    },
    removeOnComplete: 100,
    removeOnFail: false // Keep for debugging
  }
});
```

### 2. Push Notification Failure

**Scenario:** FCM reports invalid device token.

```typescript
async sendPushNotification(userId: string, notification: PushPayload) {
  const tokens = await this.prisma.pushToken.findMany({
    where: { userId, valid: true }
  });

  const results = await Promise.allSettled(
    tokens.map(async (token) => {
      try {
        await this.fcm.send({
          token: token.token,
          notification: {
            title: notification.title,
            body: notification.body
          },
          data: notification.data
        });
        return { token: token.id, success: true };
      } catch (error) {
        return { token: token.id, success: false, error };
      }
    })
  );

  // Process results
  for (const result of results) {
    if (result.status === 'fulfilled' && !result.value.success) {
      const error = result.value.error;

      // Invalid token - mark for cleanup
      if (error.code === 'messaging/registration-token-not-registered') {
        await this.prisma.pushToken.update({
          where: { id: result.value.token },
          data: { valid: false, invalidatedAt: new Date() }
        });
      }
    }
  }

  // Cleanup invalid tokens periodically
  await this.maybeCleanupTokens(userId);
}

async maybeCleanupTokens(userId: string) {
  const invalidCount = await this.prisma.pushToken.count({
    where: { userId, valid: false }
  });

  if (invalidCount > 5) {
    await this.prisma.pushToken.deleteMany({
      where: {
        userId,
        valid: false,
        invalidatedAt: { lt: subDays(new Date(), 7) }
      }
    });
  }
}
```

---

## Database Transaction Recovery

### 1. Transaction Rollback

**Scenario:** Multi-step operation fails midway.

```typescript
// Saga pattern for complex operations
class BookingSaga {
  private steps: SagaStep[] = [];
  private completedSteps: string[] = [];

  async execute(
    stepName: string,
    action: () => Promise<any>,
    compensate?: (error: Error) => Promise<void>
  ): Promise<any> {
    this.steps.push({ name: stepName, compensate });

    try {
      const result = await action();
      this.completedSteps.push(stepName);
      return result;
    } catch (error) {
      // Trigger rollback
      await this.rollback(error);
      throw error;
    }
  }

  async rollback(originalError: Error): Promise<void> {
    // Execute compensations in reverse order
    for (const stepName of this.completedSteps.reverse()) {
      const step = this.steps.find(s => s.name === stepName);
      if (step?.compensate) {
        try {
          await step.compensate(originalError);
        } catch (compensateError) {
          // Log but continue rollback
          this.logger.error('Compensation failed', {
            step: stepName,
            originalError,
            compensateError
          });
        }
      }
    }
  }
}
```

### 2. Deadlock Recovery

```typescript
// Retry with exponential backoff on deadlock
async withDeadlockRetry<T>(
  operation: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (this.isDeadlock(error) && attempt < maxRetries - 1) {
        const delay = Math.pow(2, attempt) * 100 + Math.random() * 100;
        await sleep(delay);
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}

isDeadlock(error: any): boolean {
  return error.code === 'P2034' || // Prisma deadlock
         error.message.includes('deadlock detected');
}
```

---

## External Service Recovery

### 1. Clerk Authentication Failure

**Scenario:** Clerk API is temporarily unavailable.

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    try {
      // Primary: Verify with Clerk
      const session = await this.clerk.sessions.verifySession(token);
      request.user = await this.getOrCreateUser(session.userId);
      return true;

    } catch (error) {
      if (this.isClerkUnavailable(error)) {
        // Fallback: Check local cache
        const cached = await this.cache.get(`session:${token}`);
        if (cached && !this.isExpired(cached)) {
          request.user = cached.user;
          request.authFallback = true;
          return true;
        }
      }
      throw new UnauthorizedException();
    }
  }

  private isClerkUnavailable(error: any): boolean {
    return error.status >= 500 ||
           error.code === 'ECONNREFUSED' ||
           error.code === 'ETIMEDOUT';
  }
}
```

### 2. OpenAI API Failure

**Scenario:** AI enhancement fails.

```typescript
async enhanceFeedback(rawNotes: string): Promise<EnhancedFeedback> {
  try {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: [
        { role: 'system', content: ENHANCEMENT_PROMPT },
        { role: 'user', content: rawNotes }
      ],
      timeout: 30000
    });

    return this.parseEnhancement(response);

  } catch (error) {
    // AI is non-critical - graceful degradation
    this.logger.warn('AI enhancement failed, using raw notes', { error });

    return {
      enhanced: false,
      text: rawNotes, // Use original
      skills: [],     // No skill extraction
      drills: [],     // No drill recommendations
      fallbackReason: error.message
    };
  }
}
```

---

## Monitoring & Alerting

### Error Categories

| Category | Severity | Response |
|----------|----------|----------|
| Payment failure | Critical | Page on-call, auto-retry |
| Booking conflict | High | Auto-resolve, notify user |
| Notification failure | Medium | Auto-retry, batch alert |
| AI enhancement failure | Low | Graceful degradation |
| Analytics failure | Low | Silent retry |

### Health Checks

```typescript
@Controller('health')
export class HealthController {
  @Get()
  async check(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkStripe(),
      this.checkClerk(),
      this.checkResend()
    ]);

    const results = {
      database: checks[0],
      redis: checks[1],
      stripe: checks[2],
      clerk: checks[3],
      resend: checks[4]
    };

    const healthy = checks.every(c => c.status === 'fulfilled');

    return {
      status: healthy ? 'healthy' : 'degraded',
      checks: results,
      timestamp: new Date()
    };
  }
}
```

### Dead Letter Queue Processing

```typescript
@Cron('0 */15 * * * *') // Every 15 minutes
async processDLQ() {
  const failed = await this.queue.getFailed(0, 100);

  for (const job of failed) {
    const age = Date.now() - job.finishedOn;

    // Old failures: Move to manual queue
    if (age > 24 * 60 * 60 * 1000) { // 24 hours
      await this.manualQueue.add('review', {
        originalJob: job.name,
        data: job.data,
        error: job.failedReason
      });
      await job.remove();
      continue;
    }

    // Recent failures: Retry if transient
    if (this.isTransientError(job.failedReason)) {
      await job.retry();
    }
  }
}
```

---

*Last Updated: 2026-01-11*
