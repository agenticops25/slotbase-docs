# Idempotency Specification

## Overview

Idempotency ensures that retried requests (due to network failures, timeouts, or user double-clicks) produce the same result as the original request. This is critical for:
- Payment recording
- Booking creation/confirmation
- Any state-changing operation

---

## Idempotency Key Format

```
{userId}:{operation}:{resourceId}:{clientTimestamp}
```

### Examples
```
usr_abc123:booking.create:res_xyz:1704067200000
usr_abc123:payment.record:bkg_456:1704067200001
usr_abc123:booking.confirm:bkg_789:1704067200002
```

### Key Components
| Component | Description |
|-----------|-------------|
| userId | Clerk user ID (ensures user isolation) |
| operation | Action being performed |
| resourceId | Target resource ID |
| clientTimestamp | Client-provided timestamp (prevents collisions) |

---

## Storage Strategy

### Primary: Redis
```typescript
// Key structure
idempotency:{idempotencyKey}

// Value structure
{
  status: 'processing' | 'completed' | 'failed',
  response: { statusCode: number, body: any } | null,
  createdAt: ISO8601,
  expiresAt: ISO8601
}
```

### TTL Configuration
| Scenario | TTL |
|----------|-----|
| Default | 24 hours |
| Payment operations | 72 hours |
| During processing | 5 minutes (lock timeout) |

### Fallback: PostgreSQL
For persistence across Redis restarts:

```prisma
model IdempotencyKey {
  id           String   @id @default(cuid())
  key          String   @unique
  status       String   // processing, completed, failed
  response     Json?
  createdAt    DateTime @default(now())
  expiresAt    DateTime

  @@index([expiresAt])
}
```

---

## Required Endpoints

All mutation endpoints that change critical state:

### Booking Operations
| Endpoint | Idempotency Required | Notes |
|----------|---------------------|-------|
| POST /bookings | Yes | Creates booking from hold |
| POST /bookings/:id/confirm | Yes | Confirms and processes payment |
| DELETE /bookings/:id | Yes | Prevents double-cancellation |
| POST /bookings/:id/check-in | Yes | One check-in per booking |

### Payment Operations
| Endpoint | Idempotency Required | Notes |
|----------|---------------------|-------|
| POST /offline-payments | Yes | Records payment |
| POST /refunds | Yes | Prevents double-refund |
| POST /payment-intents | Yes | Stripe payment creation |

### User Operations
| Endpoint | Idempotency Required | Notes |
|----------|---------------------|-------|
| POST /coach-blocks | Yes | Block requests |
| POST /coaches/:id/students | Yes | Student assignment |

---

## Implementation

### Request Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Client     │────▶│  API Guard   │────▶│   Handler    │
│              │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       │                    ▼                    │
       │            ┌──────────────┐             │
       │            │    Redis     │             │
       │            │  (Check Key) │             │
       │            └──────────────┘             │
       │                    │                    │
       │         ┌──────────┴──────────┐        │
       │         ▼                     ▼        │
       │   Key Exists?           Key Missing    │
       │         │                     │        │
       │    ┌────┴────┐               │        │
       │    ▼         ▼               ▼        │
       │ completed  processing   Lock & Process │
       │    │         │               │        │
       │    ▼         ▼               ▼        │
       │ Return    409 Conflict   Store Result  │
       │ Cached                       │        │
       │                              ▼        │
       └─────────────────────── Return New ────┘
```

### NestJS Guard Implementation

```typescript
@Injectable()
export class IdempotencyGuard implements CanActivate {
  constructor(
    private readonly redis: RedisService,
    private readonly config: ConfigService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();

    const idempotencyKey = request.headers['idempotency-key'];
    if (!idempotencyKey) {
      throw new BadRequestException('Idempotency-Key header required');
    }

    const cacheKey = `idempotency:${idempotencyKey}`;
    const existing = await this.redis.get(cacheKey);

    if (existing) {
      const data = JSON.parse(existing);

      if (data.status === 'processing') {
        throw new ConflictException('Request is being processed');
      }

      if (data.status === 'completed') {
        // Return cached response
        response.status(data.response.statusCode);
        response.send(data.response.body);
        return false; // Skip handler
      }
    }

    // Lock the key while processing
    await this.redis.set(cacheKey, JSON.stringify({
      status: 'processing',
      createdAt: new Date().toISOString(),
    }), 'EX', 300); // 5 min lock

    // Attach cleanup function to request
    request.idempotencyKey = idempotencyKey;
    request.idempotencyCacheKey = cacheKey;

    return true;
  }
}
```

### Response Interceptor

```typescript
@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  constructor(private readonly redis: RedisService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();

    if (!request.idempotencyCacheKey) {
      return next.handle();
    }

    return next.handle().pipe(
      tap(async (data) => {
        // Store successful response
        await this.redis.set(
          request.idempotencyCacheKey,
          JSON.stringify({
            status: 'completed',
            response: {
              statusCode: response.statusCode,
              body: data,
            },
            createdAt: new Date().toISOString(),
          }),
          'EX', 86400 // 24 hours
        );
      }),
      catchError(async (error) => {
        // Remove lock on failure (allow retry)
        await this.redis.del(request.idempotencyCacheKey);
        throw error;
      }),
    );
  }
}
```

---

## Client Usage

### Required Header
```http
POST /bookings HTTP/1.1
Idempotency-Key: usr_abc123:booking.create:res_xyz:1704067200000
Content-Type: application/json

{
  "holdId": "hold_123",
  "paymentMethodId": "pm_456"
}
```

### Client-Side Key Generation

```typescript
function generateIdempotencyKey(
  userId: string,
  operation: string,
  resourceId: string
): string {
  return `${userId}:${operation}:${resourceId}:${Date.now()}`;
}

// Usage
const key = generateIdempotencyKey(user.id, 'booking.create', resourceId);
```

### Retry Behavior

```typescript
async function createBookingWithRetry(data: BookingData): Promise<Booking> {
  const idempotencyKey = generateIdempotencyKey(
    data.userId,
    'booking.create',
    data.resourceId
  );

  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      return await api.post('/bookings', data, {
        headers: { 'Idempotency-Key': idempotencyKey }
      });
    } catch (error) {
      if (error.status === 409) {
        // Request in progress, wait and retry
        await sleep(1000 * (attempt + 1));
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}
```

---

## Edge Cases

### 1. Redis Unavailable
- Fall back to PostgreSQL
- Log warning for monitoring
- Continue processing (degrade gracefully)

### 2. Response Too Large
- Store response hash + metadata only
- Return 200 with "cached response" indicator
- Client should request full data via GET

### 3. Clock Skew
- Client timestamp is informational only
- Server validates key format, not timestamp accuracy
- Keys are unique by full string match

### 4. Key Collision
- Extremely unlikely with timestamp + userId + resourceId
- If occurs, second request gets 409 Conflict
- Client should generate new key and retry

---

## Monitoring

### Metrics to Track
| Metric | Description |
|--------|-------------|
| idempotency.cache.hit | Cached response returned |
| idempotency.cache.miss | New request processed |
| idempotency.conflict | 409 returned (in-progress) |
| idempotency.fallback | Redis unavailable, used DB |

### Alerts
- Cache hit rate > 10% (may indicate client bugs)
- Conflict rate > 5% (may indicate slow processing)
- Fallback rate > 0 (Redis issues)

---

## Cleanup

### Scheduled Job
```typescript
@Cron('0 * * * *') // Every hour
async cleanupExpiredKeys() {
  // Redis handles TTL automatically

  // PostgreSQL cleanup
  await this.prisma.idempotencyKey.deleteMany({
    where: {
      expiresAt: { lt: new Date() }
    }
  });
}
```

---

*Last Updated: 2026-01-11*
