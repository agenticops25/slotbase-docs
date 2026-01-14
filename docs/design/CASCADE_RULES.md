# Cascade Delete & Data Integrity Rules

## Overview

This document defines how deletions and data modifications cascade through the system. SlotBase prioritizes data integrity and audit trails over convenience.

**Core Principle**: Financial and booking data is NEVER hard-deleted.

---

## Delete Behavior by Entity

### Identity Domain

| Entity | Delete Type | Cascade Behavior | Retention |
|--------|-------------|------------------|-----------|
| User | Anonymize | Keep bookings, anonymize PII, preserve audit trail | 7 years |
| UserRole | Hard delete | None (junction table) | N/A |
| FamilyGroup | Soft delete | Members retain individual access | 7 years |
| FamilyMember | Hard delete | None (junction table) | N/A |
| ParentChildLink | Soft delete | Child retains account, loses parent control | 7 years |

### Organization Domain

| Entity | Delete Type | Cascade Behavior | Retention |
|--------|-------------|------------------|-----------|
| Organization | Soft delete | All facilities -> inactive | Forever |
| Facility | Soft delete | Resources -> inactive, bookings orphan-checked | Forever |
| Resource | Soft delete | Block new bookings, preserve history | Forever |
| ResourceAvailability | Hard delete | None | N/A |
| ResourcePricing | Soft delete | Historical prices preserved | 7 years |
| OperatingHours | Hard delete | None | N/A |
| Holiday | Hard delete | None | N/A |

### Booking Domain

| Entity | Delete Type | Cascade Behavior | Retention |
|--------|-------------|------------------|-----------|
| Booking | NEVER delete | Cancellation status only | Forever |
| BookingParticipant | NEVER delete | Part of booking history | Forever |
| RecurringBooking | Soft delete | Future instances cancelled | Forever |
| BookingHold | Hard delete | Automatic on expiration | N/A |
| WaitlistEntry | Hard delete | On booking or expiration | N/A |
| BookingEvent | NEVER delete | Audit trail | Forever |

### Payment Domain

| Entity | Delete Type | Cascade Behavior | Retention |
|--------|-------------|------------------|-----------|
| BillingAccount | Soft delete | Preserve transactions | 7 years |
| PaymentMethod | Soft delete | Detach from Stripe | 7 years |
| Transaction | NEVER delete | Legal requirement | 7 years |
| BookingPayment | NEVER delete | Part of transaction history | 7 years |
| Refund | NEVER delete | Legal requirement | 7 years |
| OfflinePayment | NEVER delete | Audit trail | 7 years |
| Invoice | NEVER delete | Legal requirement | 7 years |

### Coach Domain

| Entity | Delete Type | Cascade Behavior | Retention |
|--------|-------------|------------------|-----------|
| CoachProfile | Soft delete | Preserve student history | Forever |
| CoachFacilityAffiliation | Soft delete | End relationship | Forever |
| CoachStudent | Soft delete | Preserve feedback history | Forever |
| CoachAvailability | Hard delete | None | N/A |
| CoachBlock | Soft delete | Release time slot | Forever |

### Feedback Domain

| Entity | Delete Type | Cascade Behavior | Retention |
|--------|-------------|------------------|-----------|
| CoachingSession | Soft delete | Preserve feedback | Forever |
| SessionFeedback | Soft delete | Keep for progress history | Forever |
| FeedbackMedia | Soft delete | Queue for storage cleanup | 90 days |
| PlayerSkillProgress | NEVER delete | Progress tracking | Forever |
| PlayerProgressSummary | Regenerate | On source data change | N/A |

---

## Soft Delete Implementation

### Schema Pattern

```prisma
model Facility {
  id        String    @id @default(cuid())
  // ... fields

  deletedAt DateTime? @map("deleted_at")
  deletedBy String?   @map("deleted_by")

  @@index([deletedAt])
}
```

### Query Middleware

```typescript
// Prisma middleware to filter soft-deleted records
prisma.$use(async (params, next) => {
  if (params.action === 'findMany' || params.action === 'findFirst') {
    if (!params.args) params.args = {};
    if (!params.args.where) params.args.where = {};

    // Only apply to soft-deletable models
    if (SOFT_DELETE_MODELS.includes(params.model)) {
      params.args.where.deletedAt = null;
    }
  }
  return next(params);
});

// Explicit include deleted
const allFacilities = await prisma.facility.findMany({
  where: { deletedAt: { not: null } } // Include deleted
});
```

---

## GDPR/CCPA Deletion Workflow

### User Data Deletion Request

```typescript
async function handleDeletionRequest(userId: string): Promise<void> {
  // 1. Validate request
  const user = await prisma.user.findUnique({ where: { id: userId } });
  if (!user) throw new NotFoundException('User not found');

  // 2. Check for blocking conditions
  const pendingBookings = await prisma.booking.count({
    where: {
      participants: { some: { userId } },
      status: 'CONFIRMED',
      startTime: { gt: new Date() }
    }
  });

  if (pendingBookings > 0) {
    throw new BadRequestException('Cancel pending bookings before deletion');
  }

  const pendingPayments = await prisma.offlinePayment.count({
    where: { recordedByUserId: userId, status: 'PENDING' }
  });

  if (pendingPayments > 0) {
    throw new BadRequestException('Resolve pending payments before deletion');
  }

  // 3. Queue deletion job (30-day grace period)
  await this.queue.add('user-deletion', {
    userId,
    requestedAt: new Date(),
    executeAt: addDays(new Date(), 30)
  });

  // 4. Mark account for deletion
  await prisma.user.update({
    where: { id: userId },
    data: {
      deletionRequestedAt: new Date(),
      status: 'PENDING_DELETION'
    }
  });

  // 5. Notify user
  await this.notifications.send({
    userId,
    type: 'DELETION_SCHEDULED',
    data: { executeDate: addDays(new Date(), 30) }
  });
}
```

### Anonymization Process

```typescript
async function anonymizeUser(userId: string): Promise<void> {
  const anonymousId = `anon_${randomBytes(16).toString('hex')}`;

  await prisma.$transaction([
    // 1. Anonymize user record
    prisma.user.update({
      where: { id: userId },
      data: {
        email: `${anonymousId}@deleted.slotbase.com`,
        firstName: 'Deleted',
        lastName: 'User',
        phone: null,
        avatarUrl: null,
        deletedAt: new Date(),
        clerkId: null, // Remove auth link
      }
    }),

    // 2. Remove from families
    prisma.familyMember.deleteMany({ where: { userId } }),
    prisma.parentChildLink.updateMany({
      where: { OR: [{ parentId: userId }, { childId: userId }] },
      data: { deletedAt: new Date() }
    }),

    // 3. Preserve booking history (anonymized)
    // Bookings retain structure but participant shows "Deleted User"

    // 4. Delete notification preferences
    prisma.notificationPreference.deleteMany({ where: { userId } }),

    // 5. Delete push tokens
    prisma.pushToken.deleteMany({ where: { userId } }),
  ]);

  // 6. Revoke Clerk session
  await this.clerk.users.deleteUser(userId);

  // 7. Emit event for other services
  await this.events.emit('user.anonymized', { userId, anonymousId });
}
```

### What We Retain (Legal Requirements)

| Data | Reason | Retention |
|------|--------|-----------|
| Transaction records | Tax/accounting law | 7 years |
| Invoice history | Tax/accounting law | 7 years |
| Booking records | Business records | 7 years |
| Payment disputes | Legal protection | 7 years |

---

## Cascade Scenarios

### Scenario 1: Facility Deactivation

```typescript
async function deactivateFacility(facilityId: string): Promise<void> {
  await prisma.$transaction([
    // 1. Soft-delete facility
    prisma.facility.update({
      where: { id: facilityId },
      data: { status: 'INACTIVE', deletedAt: new Date() }
    }),

    // 2. Deactivate all resources
    prisma.resource.updateMany({
      where: { facilityId },
      data: { status: 'INACTIVE', deletedAt: new Date() }
    }),

    // 3. Cancel future bookings
    prisma.booking.updateMany({
      where: {
        resource: { facilityId },
        startTime: { gt: new Date() },
        status: 'CONFIRMED'
      },
      data: {
        status: 'CANCELLED',
        cancellationReason: 'FACILITY_CLOSED'
      }
    }),

    // 4. Notify affected users
    // Handled by event listener
  ]);

  await this.events.emit('facility.deactivated', { facilityId });
}
```

### Scenario 2: Coach Leaves Facility

```typescript
async function endCoachAffiliation(
  coachId: string,
  facilityId: string
): Promise<void> {
  await prisma.$transaction([
    // 1. End affiliation
    prisma.coachFacilityAffiliation.update({
      where: { coachId_facilityId: { coachId, facilityId } },
      data: {
        status: 'INACTIVE',
        endDate: new Date(),
        deletedAt: new Date()
      }
    }),

    // 2. Release future blocks
    prisma.coachBlock.updateMany({
      where: {
        coachId,
        resource: { facilityId },
        startTime: { gt: new Date() }
      },
      data: {
        status: 'CANCELLED',
        deletedAt: new Date()
      }
    }),

    // 3. Preserve student relationships (historical)
    // CoachStudent records are NOT deleted

    // 4. Preserve feedback history
    // SessionFeedback records are NOT deleted
  ]);
}
```

### Scenario 3: Resource Removal

```typescript
async function removeResource(resourceId: string): Promise<void> {
  // 1. Check for future bookings
  const futureBookings = await prisma.booking.count({
    where: {
      resourceId,
      startTime: { gt: new Date() },
      status: 'CONFIRMED'
    }
  });

  if (futureBookings > 0) {
    throw new BadRequestException(
      `Cannot remove resource with ${futureBookings} future bookings. Cancel them first.`
    );
  }

  // 2. Soft-delete resource
  await prisma.resource.update({
    where: { id: resourceId },
    data: {
      status: 'INACTIVE',
      deletedAt: new Date()
    }
  });

  // 3. Historical bookings remain linked to resource
  // Resource name/details preserved for history
}
```

---

## Orphan Prevention

### Foreign Key Constraints

```prisma
model Booking {
  id         String   @id
  resourceId String
  resource   Resource @relation(fields: [resourceId], references: [id], onDelete: Restrict)
  // Restrict prevents deletion if bookings exist
}
```

### Orphan Check Query

```typescript
async function checkOrphans(): Promise<OrphanReport> {
  const orphanedBookings = await prisma.booking.findMany({
    where: {
      resource: { deletedAt: { not: null } },
      status: 'CONFIRMED',
      startTime: { gt: new Date() }
    }
  });

  const orphanedPayments = await prisma.transaction.findMany({
    where: {
      billingAccount: { deletedAt: { not: null } },
      status: 'PENDING'
    }
  });

  return {
    bookings: orphanedBookings,
    payments: orphanedPayments,
    hasOrphans: orphanedBookings.length > 0 || orphanedPayments.length > 0
  };
}
```

---

## Audit Trail

### Delete Event Logging

```typescript
// Every deletion emits an event
await this.events.emit('entity.deleted', {
  entityType: 'Facility',
  entityId: facilityId,
  deleteType: 'soft',
  deletedBy: currentUser.id,
  deletedAt: new Date(),
  reason: 'User requested',
  cascadedTo: ['Resource:res_1', 'Resource:res_2']
});
```

### Deletion Audit Table

```prisma
model DeletionAudit {
  id          String   @id @default(cuid())
  entityType  String
  entityId    String
  deleteType  String   // soft, anonymize, hard
  deletedBy   String
  deletedAt   DateTime
  reason      String?
  cascadedTo  String[] // Related entities affected
  metadata    Json?

  @@index([entityType, entityId])
  @@index([deletedAt])
}
```

---

## Data Retention Schedule

| Category | Retention | Cleanup Action |
|----------|-----------|----------------|
| Active records | Forever | N/A |
| Soft-deleted | Forever | Archive after 2 years |
| Anonymized users | 7 years | Hard delete |
| Media files | 90 days after soft-delete | S3 lifecycle |
| Logs | 1 year | Archive to cold storage |
| Metrics | 3 years | Aggregate and delete |

### Cleanup Job

```typescript
@Cron('0 2 * * 0') // Weekly at 2 AM Sunday
async function cleanupRetiredData(): Promise<void> {
  const twoYearsAgo = subYears(new Date(), 2);
  const sevenYearsAgo = subYears(new Date(), 7);

  // Archive old soft-deleted records
  await this.archiveService.archive({
    table: 'bookings',
    where: { deletedAt: { lt: twoYearsAgo } }
  });

  // Hard-delete expired anonymized users
  await prisma.user.deleteMany({
    where: {
      deletedAt: { lt: sevenYearsAgo },
      email: { contains: '@deleted.slotbase.com' }
    }
  });
}
```

---

*Last Updated: 2026-01-11*
