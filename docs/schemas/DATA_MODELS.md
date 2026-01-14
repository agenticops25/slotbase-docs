# SlotBase Data Models

## Overview

This document defines the complete data model for SlotBase, including all entities, relationships, and constraints.

---

## Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              IDENTITY DOMAIN                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │     User     │◄────▶│   UserRole   │◄────▶│     Role     │                  │
│  └──────┬───────┘      └──────────────┘      └──────────────┘                  │
│         │                                                                        │
│         │ 1:N                                                                    │
│         ▼                                                                        │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │ FamilyMember │◄────▶│ FamilyGroup  │      │ParentChildLink│                 │
│  └──────────────┘      └──────────────┘      └──────────────┘                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                            ORGANIZATION DOMAIN                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │ Organization │◄────▶│   Facility   │◄────▶│   Resource   │                  │
│  └──────────────┘      └──────┬───────┘      └──────────────┘                  │
│                               │                                                  │
│                               │ 1:N                                              │
│                               ▼                                                  │
│                        ┌──────────────┐      ┌──────────────┐                  │
│                        │FacilityPolicy│      │OperatingHours│                  │
│                        └──────────────┘      └──────────────┘                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                              BOOKING DOMAIN                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │   Booking    │◄────▶│BookingPartic.│◄────▶│     User     │                  │
│  └──────┬───────┘      └──────────────┘      └──────────────┘                  │
│         │                                                                        │
│         │ 1:1                                                                    │
│         ▼                                                                        │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │BookingPayment│      │RecurringBook.│      │ WaitlistEntry│                  │
│  └──────────────┘      └──────────────┘      └──────────────┘                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                              PAYMENT DOMAIN                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │BillingAccount│◄────▶│ Transaction  │◄────▶│    Refund    │                  │
│  └──────┬───────┘      └──────────────┘      └──────────────┘                  │
│         │                                                                        │
│         │ 1:N                                                                    │
│         ▼                                                                        │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │PaymentMethod │      │OfflinePayment│      │   Invoice    │                  │
│  └──────────────┘      └──────────────┘      └──────────────┘                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                            SUBSCRIPTION DOMAIN                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │PlatformPlan  │◄────▶│ Subscription │◄────▶│ Entitlement  │                  │
│  └──────────────┘      └──────────────┘      └──────────────┘                  │
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐                                        │
│  │    Coupon    │◄────▶│CouponRedempt.│                                        │
│  └──────────────┘      └──────────────┘                                        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                               COACH DOMAIN                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │ CoachProfile │◄────▶│CoachFacility │◄────▶│ CoachStudent │                  │
│  └──────┬───────┘      └──────────────┘      └──────────────┘                  │
│         │                                                                        │
│         │ 1:N                                                                    │
│         ▼                                                                        │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │  CoachBlock  │◄────▶│BlockApproval │      │CoachAvailab. │                  │
│  └──────────────┘      └──────────────┘      └──────────────┘                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                             FEEDBACK DOMAIN                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │CoachingSessn │◄────▶│SessionFeedbck│◄────▶│ FeedbackMedia│                  │
│  └──────────────┘      └──────────────┘      └──────────────┘                  │
│                                                                                  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                  │
│  │    Skill     │◄────▶│SkillProgress │◄────▶│   Drill      │                  │
│  └──────────────┘      └──────────────┘      └──────────────┘                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Prisma Schema

### Identity Domain

```prisma
// schema.prisma - Identity Domain

model User {
  id              String   @id @default(uuid())
  externalId      String?  @unique // Clerk user ID
  email           String   @unique
  phone           String?
  firstName       String
  lastName        String
  avatarUrl       String?
  status          UserStatus @default(ACTIVE)

  // Timestamps
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  lastActiveAt    DateTime?

  // Relations
  roles           UserRole[]
  bookings        Booking[] @relation("BookerBookings")
  participations  BookingParticipant[]
  familyMembers   FamilyMember[]
  parentLinks     ParentChildLink[] @relation("ParentLinks")
  childLinks      ParentChildLink[] @relation("ChildLinks")
  coachProfile    CoachProfile?
  billingAccounts BillingAccount[]

  @@index([email])
  @@index([externalId])
}

enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
  DELETED
}

model UserRole {
  id          String   @id @default(uuid())
  userId      String
  role        RoleType
  scopeType   ScopeType
  scopeId     String?  // Facility ID, Organization ID, etc.

  // Metadata
  grantedBy   String?
  grantedAt   DateTime @default(now())
  expiresAt   DateTime?

  // Relations
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, role, scopeType, scopeId])
  @@index([userId])
  @@index([scopeType, scopeId])
}

enum RoleType {
  PLATFORM_ADMIN
  FACILITY_OWNER
  FACILITY_ADMIN
  FACILITY_STAFF
  COACH
  PLAYER
  PARENT
}

enum ScopeType {
  PLATFORM
  ORGANIZATION
  FACILITY
}

model FamilyGroup {
  id                    String   @id @default(uuid())
  name                  String
  primaryBillingAccountId String?

  // Settings
  notificationPreferences Json?
  bookingPreferences      Json?

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  members               FamilyMember[]
  billingAccount        BillingAccount? @relation(fields: [primaryBillingAccountId], references: [id])
}

model FamilyMember {
  id            String   @id @default(uuid())
  familyId      String
  userId        String
  role          FamilyRole

  // Permissions
  canManageBilling Boolean @default(false)
  canAddMembers    Boolean @default(false)
  canBookForOthers Boolean @default(true)

  // Timestamps
  joinedAt      DateTime @default(now())

  // Relations
  family        FamilyGroup @relation(fields: [familyId], references: [id], onDelete: Cascade)
  user          User        @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([familyId, userId])
}

enum FamilyRole {
  PARENT
  CHILD
  GUARDIAN
}

model ParentChildLink {
  id              String   @id @default(uuid())
  parentUserId    String
  childUserId     String
  relationship    RelationshipType
  isPrimary       Boolean  @default(false)

  // Permissions
  canBook             Boolean @default(true)
  canCancel           Boolean @default(true)
  canPay              Boolean @default(true)
  canReceiveUpdates   Boolean @default(true)
  canViewProgress     Boolean @default(true)
  canCommunicateCoach Boolean @default(true)
  canModifyProfile    Boolean @default(false)
  isEmergencyContact  Boolean @default(false)

  // Custody (optional)
  custodySchedule     Json?

  // Verification
  verifiedAt          DateTime?
  validUntil          DateTime?

  // Timestamps
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  // Relations
  parent              User @relation("ParentLinks", fields: [parentUserId], references: [id], onDelete: Cascade)
  child               User @relation("ChildLinks", fields: [childUserId], references: [id], onDelete: Cascade)

  @@unique([parentUserId, childUserId])
  @@index([childUserId])
}

enum RelationshipType {
  PARENT
  GUARDIAN
  CAREGIVER
  COACH
}

model ChildPlayer {
  userId              String   @id
  birthDate           DateTime?
  ageVerified         Boolean  @default(false)

  // Skill levels per sport
  skillLevels         Json?    // {tennis: 3.5, swimming: 'intermediate'}

  // Health & safety
  medicalNotes        String?  // Encrypted
  emergencyContactName  String?
  emergencyContactPhone String?

  // Requirements
  requiresSupervision Boolean @default(true)
  supervisionNotes    String?

  // Timestamps
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  // Relations
  user                User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

### Organization Domain

```prisma
// schema.prisma - Organization Domain

model Organization {
  id                String   @id @default(uuid())
  name              String
  billingAccountId  String?  @unique

  // Settings
  settings          Json?

  // Timestamps
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  // Relations
  facilities        Facility[]
  billingAccount    BillingAccount? @relation(fields: [billingAccountId], references: [id])
}

model Facility {
  id              String   @id @default(uuid())
  organizationId  String?
  name            String
  slug            String   @unique

  // Location
  address         Json     // {street, city, state, zip, country}
  coordinates     Json?    // {lat, lng}
  timezone        String   // e.g., 'America/Los_Angeles'

  // Contact
  email           String?
  phone           String?
  website         String?

  // Branding
  logoUrl         String?
  coverImageUrl   String?
  description     String?

  // Sport types supported
  sportTypes      SportType[]

  // Status
  status          FacilityStatus @default(ACTIVE)

  // Timestamps
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  // Relations
  organization    Organization?      @relation(fields: [organizationId], references: [id])
  resources       Resource[]
  bookings        Booking[]
  policies        FacilityPolicy[]
  operatingHours  OperatingHours[]
  holidays        Holiday[]
  subscriptions   Subscription[]
  paymentConfig   FacilityPaymentConfig?
  coachAffiliations CoachFacilityAffiliation[]
  tournaments     Tournament[]
  rosters         Roster[]
  lostItems       LostItem[]
  devices         FacilityDevice[]

  @@index([organizationId])
  @@index([slug])
}

enum FacilityStatus {
  ACTIVE
  INACTIVE
  PENDING_SETUP
  SUSPENDED
}

enum SportType {
  TENNIS
  PICKLEBALL
  BASKETBALL
  BADMINTON
  SWIMMING
  CRICKET
  SQUASH
  VOLLEYBALL
  TABLE_TENNIS
  OTHER
}

model Resource {
  id              String   @id @default(uuid())
  facilityId      String
  name            String
  resourceType    ResourceType

  // Attributes
  attributes      Json?    // {indoor: true, surface: 'hard', lighting: true}

  // Status
  status          ResourceStatus @default(ACTIVE)

  // Display
  displayOrder    Int      @default(0)

  // Timestamps
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  // Relations
  facility        Facility @relation(fields: [facilityId], references: [id], onDelete: Cascade)
  availability    ResourceAvailability[]
  pricing         ResourcePricing[]
  bookings        Booking[]
  maintenanceWindows MaintenanceWindow[]
  coachBlocks     CoachBlock[]

  @@index([facilityId])
}

enum ResourceType {
  TENNIS_COURT
  PICKLEBALL_COURT
  BASKETBALL_COURT
  BADMINTON_COURT
  SWIMMING_LANE
  POOL_SECTION
  CRICKET_LANE
  BATTING_CAGE
  SQUASH_COURT
  MULTI_PURPOSE
}

enum ResourceStatus {
  ACTIVE
  MAINTENANCE
  INACTIVE
}

model ResourceAvailability {
  id              String   @id @default(uuid())
  resourceId      String
  dayOfWeek       Int      // 0 = Sunday, 6 = Saturday
  startTime       String   // HH:MM format
  endTime         String   // HH:MM format
  bookingTypesAllowed BookingType[]

  // Relations
  resource        Resource @relation(fields: [resourceId], references: [id], onDelete: Cascade)

  @@index([resourceId])
}

model ResourcePricing {
  id              String   @id @default(uuid())
  resourceId      String

  // Time-based pricing
  dayType         DayType  // WEEKDAY, WEEKEND, HOLIDAY
  startTime       String?  // HH:MM - null means all day
  endTime         String?  // HH:MM

  // Price
  pricePerHour    Decimal  @db.Decimal(10, 2)
  currency        String   @default("USD")

  // Validity
  validFrom       DateTime?
  validUntil      DateTime?

  // Relations
  resource        Resource @relation(fields: [resourceId], references: [id], onDelete: Cascade)

  @@index([resourceId])
}

enum DayType {
  WEEKDAY
  WEEKEND
  HOLIDAY
  PEAK
  OFF_PEAK
}

model OperatingHours {
  id              String   @id @default(uuid())
  facilityId      String
  dayOfWeek       Int      // 0 = Sunday
  openTime        String   // HH:MM
  closeTime       String   // HH:MM
  isClosed        Boolean  @default(false)

  // Relations
  facility        Facility @relation(fields: [facilityId], references: [id], onDelete: Cascade)

  @@unique([facilityId, dayOfWeek])
}

model Holiday {
  id              String   @id @default(uuid())
  facilityId      String
  date            DateTime @db.Date
  name            String
  isClosed        Boolean  @default(true)
  modifiedHours   Json?    // {openTime, closeTime} if not closed

  // Relations
  facility        Facility @relation(fields: [facilityId], references: [id], onDelete: Cascade)

  @@unique([facilityId, date])
}

model FacilityPolicy {
  id              String   @id @default(uuid())
  facilityId      String
  policyType      PolicyType
  config          Json     // Policy-specific configuration

  // Timestamps
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  // Relations
  facility        Facility @relation(fields: [facilityId], references: [id], onDelete: Cascade)

  @@unique([facilityId, policyType])
}

enum PolicyType {
  CANCELLATION
  NO_SHOW
  BOOKING_WINDOW
  FAIRNESS
  PAYMENT
}

model MaintenanceWindow {
  id              String   @id @default(uuid())
  resourceId      String
  startAt         DateTime
  endAt           DateTime
  reason          String?

  // Timestamps
  createdAt       DateTime @default(now())

  // Relations
  resource        Resource @relation(fields: [resourceId], references: [id], onDelete: Cascade)

  @@index([resourceId, startAt, endAt])
}
```

### Booking Domain

```prisma
// schema.prisma - Booking Domain

model Booking {
  id              String   @id @default(uuid())
  facilityId      String
  resourceId      String
  bookerId        String

  // Time
  startAt         DateTime
  endAt           DateTime
  timezone        String

  // Type
  bookingType     BookingType

  // Status
  status          BookingStatus @default(HELD)

  // Recurring reference
  recurringBookingId String?

  // Pricing
  priceAmount     Decimal  @db.Decimal(10, 2)
  priceCurrency   String   @default("USD")

  // Metadata
  notes           String?
  internalNotes   String?

  // Timestamps
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  confirmedAt     DateTime?
  cancelledAt     DateTime?
  checkedInAt     DateTime?
  completedAt     DateTime?

  // Cancellation
  cancellationReason String?
  cancelledBy     String?

  // Relations
  facility        Facility         @relation(fields: [facilityId], references: [id])
  resource        Resource         @relation(fields: [resourceId], references: [id])
  booker          User             @relation("BookerBookings", fields: [bookerId], references: [id])
  participants    BookingParticipant[]
  recurringBooking RecurringBooking? @relation(fields: [recurringBookingId], references: [id])
  payment         BookingPayment?
  coachingSession CoachingSession?
  events          BookingEvent[]

  @@index([facilityId, startAt])
  @@index([resourceId, startAt])
  @@index([bookerId])
  @@index([status])
}

enum BookingType {
  ONE_TIME
  RECURRING
  DROP_IN
  LESSON
  TEAM_PRACTICE
  TOURNAMENT
  GROUP
}

enum BookingStatus {
  HELD
  CONFIRMED
  CHECKED_IN
  COMPLETED
  CANCELLED
  NO_SHOW
}

model BookingParticipant {
  id              String   @id @default(uuid())
  bookingId       String
  userId          String
  role            ParticipantRole @default(PLAYER)

  // Status
  checkedIn       Boolean  @default(false)
  checkedInAt     DateTime?

  // Relations
  booking         Booking  @relation(fields: [bookingId], references: [id], onDelete: Cascade)
  user            User     @relation(fields: [userId], references: [id])

  @@unique([bookingId, userId])
}

enum ParticipantRole {
  PLAYER
  COACH
  GUEST
}

model RecurringBooking {
  id              String   @id @default(uuid())
  facilityId      String
  resourceId      String
  bookerId        String

  // Pattern (RRULE format)
  recurrenceRule  String   // e.g., "FREQ=WEEKLY;BYDAY=MO,WE;COUNT=12"

  // Time template
  startTime       String   // HH:MM
  endTime         String   // HH:MM

  // Date range
  startDate       DateTime @db.Date
  endDate         DateTime? @db.Date

  // Status
  status          RecurringStatus @default(ACTIVE)

  // Exceptions
  exceptions      DateTime[] // Dates to skip

  // Timestamps
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  // Relations
  bookings        Booking[]

  @@index([facilityId])
}

enum RecurringStatus {
  ACTIVE
  PAUSED
  CANCELLED
  COMPLETED
}

model BookingHold {
  id              String   @id @default(uuid())
  bookingId       String   @unique
  paymentIntentId String?
  heldUntil       DateTime

  // Status
  status          HoldStatus @default(ACTIVE)

  // Timestamps
  createdAt       DateTime @default(now())
  releasedAt      DateTime?

  @@index([heldUntil])
}

enum HoldStatus {
  ACTIVE
  CONVERTED
  EXPIRED
  RELEASED
}

model WaitlistEntry {
  id              String   @id @default(uuid())
  userId          String
  facilityId      String
  resourceId      String?  // Null = any resource

  // Desired time
  desiredDate     DateTime @db.Date
  desiredStartTime String?  // HH:MM - null = any time
  desiredEndTime  String?

  // Priority
  priority        Int      @default(0)

  // Status
  status          WaitlistStatus @default(WAITING)

  // Notification
  notifiedAt      DateTime?
  expiresAt       DateTime?

  // Timestamps
  createdAt       DateTime @default(now())

  @@index([facilityId, desiredDate])
  @@index([userId])
}

enum WaitlistStatus {
  WAITING
  NOTIFIED
  BOOKED
  EXPIRED
  CANCELLED
}

model BookingEvent {
  id              String   @id @default(uuid())
  bookingId       String
  eventType       BookingEventType
  actorId         String?

  // Event data
  metadata        Json?

  // Timestamp
  occurredAt      DateTime @default(now())

  // Relations
  booking         Booking  @relation(fields: [bookingId], references: [id], onDelete: Cascade)

  @@index([bookingId])
  @@index([eventType, occurredAt])
}

enum BookingEventType {
  CREATED
  HELD
  CONFIRMED
  MODIFIED
  CANCELLED
  CHECKED_IN
  NO_SHOW_MARKED
  COMPLETED
  PAYMENT_RECEIVED
  REFUND_ISSUED
}
```

### Payment Domain

```prisma
// schema.prisma - Payment Domain

model BillingAccount {
  id                    String   @id @default(uuid())
  accountType           BillingAccountType
  ownerType             String   // 'User', 'Facility', 'Organization'
  ownerId               String

  // Stripe
  stripeCustomerId      String?  @unique

  // Default payment method
  defaultPaymentMethodId String?

  // Contact
  billingEmail          String?
  billingAddress        Json?
  taxId                 String?

  // Balance
  balance               Decimal  @db.Decimal(10, 2) @default(0)
  currency              String   @default("USD")

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  paymentMethods        PaymentMethod[]
  transactions          Transaction[]
  invoices              Invoice[]
  subscriptions         Subscription[]
  offlinePayments       OfflinePayment[]
  familyGroups          FamilyGroup[]
  organizations         Organization[]

  @@index([ownerType, ownerId])
}

enum BillingAccountType {
  FACILITY
  PLAYER
  ORGANIZATION
}

model PaymentMethod {
  id                    String   @id @default(uuid())
  billingAccountId      String
  stripePaymentMethodId String   @unique

  // Type
  type                  PaymentMethodType

  // Card details (if card)
  cardBrand             String?
  cardLast4             String?
  cardExpMonth          Int?
  cardExpYear           Int?

  // Status
  isDefault             Boolean  @default(false)

  // Timestamps
  createdAt             DateTime @default(now())

  // Relations
  billingAccount        BillingAccount @relation(fields: [billingAccountId], references: [id], onDelete: Cascade)

  @@index([billingAccountId])
}

enum PaymentMethodType {
  CARD
  BANK_ACCOUNT
  APPLE_PAY
  GOOGLE_PAY
}

model Transaction {
  id                    String   @id @default(uuid())
  billingAccountId      String

  // Amount
  amount                Decimal  @db.Decimal(10, 2)
  currency              String   @default("USD")

  // Type
  type                  TransactionType

  // Status
  status                TransactionStatus @default(PENDING)

  // Reference
  referenceType         String?  // 'Booking', 'Subscription', 'Invoice'
  referenceId           String?

  // Stripe
  stripePaymentIntentId String?
  stripeChargeId        String?

  // Failure
  failureReason         String?

  // Timestamps
  createdAt             DateTime @default(now())
  processedAt           DateTime?

  // Relations
  billingAccount        BillingAccount @relation(fields: [billingAccountId], references: [id])
  refunds               Refund[]
  bookingPayment        BookingPayment?

  @@index([billingAccountId])
  @@index([referenceType, referenceId])
  @@index([stripePaymentIntentId])
}

enum TransactionType {
  BOOKING_PAYMENT
  SUBSCRIPTION_PAYMENT
  REFUND
  CREDIT
  ADJUSTMENT
}

enum TransactionStatus {
  PENDING
  PROCESSING
  SUCCEEDED
  FAILED
  CANCELLED
  REFUNDED
}

model BookingPayment {
  id                    String   @id @default(uuid())
  bookingId             String   @unique
  transactionId         String?  @unique

  // For offline
  offlinePaymentId      String?  @unique

  // Amount
  amount                Decimal  @db.Decimal(10, 2)
  currency              String   @default("USD")

  // Status
  status                PaymentStatus @default(PENDING)

  // Timestamps
  createdAt             DateTime @default(now())
  paidAt                DateTime?

  // Relations
  booking               Booking       @relation(fields: [bookingId], references: [id])
  transaction           Transaction?  @relation(fields: [transactionId], references: [id])
  offlinePayment        OfflinePayment? @relation(fields: [offlinePaymentId], references: [id])
}

enum PaymentStatus {
  PENDING
  PAID
  FAILED
  REFUNDED
  WAIVED
}

model Refund {
  id                    String   @id @default(uuid())
  transactionId         String

  // Amount
  amount                Decimal  @db.Decimal(10, 2)

  // Reason
  reason                String?

  // Status
  status                RefundStatus @default(PENDING)

  // Stripe
  stripeRefundId        String?

  // Timestamps
  createdAt             DateTime @default(now())
  processedAt           DateTime?

  // Relations
  transaction           Transaction @relation(fields: [transactionId], references: [id])

  @@index([transactionId])
}

enum RefundStatus {
  PENDING
  PROCESSING
  SUCCEEDED
  FAILED
}

model OfflinePayment {
  id                    String   @id @default(uuid())
  facilityId            String
  billingAccountId      String

  // Reference
  bookingId             String?
  invoiceId             String?
  subscriptionId        String?

  // Amount
  amount                Decimal  @db.Decimal(10, 2)
  currency              String   @default("USD")

  // Method
  paymentMethod         OfflinePaymentMethod
  referenceNumber       String?  // Check number, transfer ID
  payerName             String?

  // Recording
  recordedByUserId      String
  paymentDate           DateTime @db.Date

  // Status
  status                OfflinePaymentStatus @default(RECORDED)

  // Notes
  notes                 String?
  attachments           Json?    // [{type, url, name}]

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  billingAccount        BillingAccount @relation(fields: [billingAccountId], references: [id])
  bookingPayment        BookingPayment?

  @@index([facilityId])
  @@index([billingAccountId])
}

enum OfflinePaymentMethod {
  CASH
  CHECK
  BANK_TRANSFER
  VENMO
  ZELLE
  OTHER
}

enum OfflinePaymentStatus {
  RECORDED
  VERIFIED
  BOUNCED
  REFUNDED
}

model Invoice {
  id                    String   @id @default(uuid())
  billingAccountId      String

  // Invoice details
  invoiceNumber         String   @unique

  // Line items
  lineItems             Json     // [{description, amount, quantity}]

  // Amounts
  subtotal              Decimal  @db.Decimal(10, 2)
  taxAmount             Decimal  @db.Decimal(10, 2) @default(0)
  total                 Decimal  @db.Decimal(10, 2)
  currency              String   @default("USD")

  // Status
  status                InvoiceStatus @default(DRAFT)

  // Dates
  issuedAt              DateTime?
  dueDate               DateTime?
  paidAt                DateTime?

  // Stripe
  stripeInvoiceId       String?

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  billingAccount        BillingAccount @relation(fields: [billingAccountId], references: [id])

  @@index([billingAccountId])
  @@index([status])
}

enum InvoiceStatus {
  DRAFT
  SENT
  PAID
  VOID
  UNCOLLECTIBLE
}

model FacilityPaymentConfig {
  facilityId            String   @id

  // Stripe
  stripeEnabled         Boolean  @default(false)
  stripeAccountId       String?
  stripeAccountStatus   StripeAccountStatus?

  // Payment modes
  acceptOnlinePayments  Boolean  @default(true)
  acceptOfflinePayments Boolean  @default(true)
  acceptCash            Boolean  @default(true)
  acceptCheck           Boolean  @default(true)
  acceptBankTransfer    Boolean  @default(false)

  // Settings
  offlinePaymentInstructions String?
  requirePaymentBeforeBooking Boolean @default(false)
  paymentTiming         PaymentTiming @default(AT_BOOKING)

  // Tax
  collectSalesTax       Boolean  @default(false)
  taxRate               Decimal? @db.Decimal(5, 4)
  taxId                 String?

  // Refund settings
  allowOnlineRefunds    Boolean  @default(true)
  refundToOriginalMethod Boolean @default(true)
  allowCreditRefunds    Boolean  @default(true)

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  facility              Facility @relation(fields: [facilityId], references: [id], onDelete: Cascade)
}

enum StripeAccountStatus {
  PENDING
  ACTIVE
  RESTRICTED
  DISABLED
}

enum PaymentTiming {
  AT_BOOKING
  BEFORE_SESSION
  AFTER_SESSION
  MONTHLY_INVOICE
}
```

### Subscription Domain

```prisma
// schema.prisma - Subscription Domain

model PlatformPlan {
  id                    String   @id @default(uuid())
  name                  String
  internalName          String   @unique

  // Pricing
  monthlyPrice          Decimal  @db.Decimal(10, 2)
  annualPrice           Decimal  @db.Decimal(10, 2)
  currency              String   @default("USD")

  // Limits
  activePlayerLimit     Int?     // Null = unlimited
  adminUserLimit        Int?
  resourceLimit         Int?

  // Features
  features              Json     // Feature flags and limits

  // Trial
  trialDays             Int      @default(14)

  // Stripe
  stripeMonthlyPriceId  String?
  stripeAnnualPriceId   String?

  // Display
  displayOrder          Int      @default(0)
  isPopular             Boolean  @default(false)
  isActive              Boolean  @default(true)

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  subscriptions         Subscription[]
}

model Subscription {
  id                    String   @id @default(uuid())
  billingAccountId      String
  facilityId            String
  planId                String

  // Status
  status                SubscriptionStatus @default(TRIALING)

  // Billing cycle
  billingCycle          BillingCycle @default(MONTHLY)

  // Period
  currentPeriodStart    DateTime
  currentPeriodEnd      DateTime

  // Trial
  trialEnd              DateTime?

  // Cancellation
  cancelAtPeriodEnd     Boolean  @default(false)
  cancelledAt           DateTime?

  // Stripe
  stripeSubscriptionId  String?  @unique

  // Metadata
  metadata              Json?

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  billingAccount        BillingAccount @relation(fields: [billingAccountId], references: [id])
  facility              Facility       @relation(fields: [facilityId], references: [id])
  plan                  PlatformPlan   @relation(fields: [planId], references: [id])
  entitlements          Entitlement[]
  couponRedemptions     CouponRedemption[]

  @@index([billingAccountId])
  @@index([facilityId])
  @@index([status])
}

enum SubscriptionStatus {
  TRIALING
  ACTIVE
  PAST_DUE
  CANCELLED
  PAUSED
  UNPAID
}

enum BillingCycle {
  MONTHLY
  ANNUAL
}

model Entitlement {
  id                    String   @id @default(uuid())
  subscriptionId        String

  // Type and limit
  entitlementType       EntitlementType
  limitValue            Int?     // Null = unlimited
  usedValue             Int      @default(0)

  // Period tracking
  periodStart           DateTime
  periodEnd             DateTime

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  subscription          Subscription @relation(fields: [subscriptionId], references: [id], onDelete: Cascade)

  @@unique([subscriptionId, entitlementType])
}

enum EntitlementType {
  ACTIVE_PLAYERS
  ADMIN_USERS
  RESOURCES
  BOOKINGS_PER_MONTH
  VIDEO_STORAGE_GB
  API_CALLS_PER_MONTH
}

model Coupon {
  id                    String   @id @default(uuid())
  code                  String   @unique
  description           String?

  // Discount type
  discountType          DiscountType
  discountValue         Decimal  @db.Decimal(10, 2)

  // Applicability
  applicablePlans       String[] // Plan IDs, empty = all
  applicableBillingCycles BillingCycle[]

  // Duration
  durationType          CouponDuration
  durationMonths        Int?

  // Limits
  maxRedemptions        Int?
  maxRedemptionsPerUser Int      @default(1)
  currentRedemptions    Int      @default(0)

  // Validity
  validFrom             DateTime @default(now())
  validUntil            DateTime?

  // Restrictions
  newCustomersOnly      Boolean  @default(true)
  minimumAmount         Decimal? @db.Decimal(10, 2)

  // Stripe
  stripeCouponId        String?

  // Tracking
  createdByUserId       String?
  campaignName          String?

  // Status
  isActive              Boolean  @default(true)

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  redemptions           CouponRedemption[]
}

enum DiscountType {
  PERCENTAGE
  FIXED_AMOUNT
  FREE_TRIAL_EXTENSION
}

enum CouponDuration {
  ONCE
  REPEATING
  FOREVER
}

model CouponRedemption {
  id                    String   @id @default(uuid())
  couponId              String
  facilityId            String
  subscriptionId        String

  // Redeemer
  redeemedByUserId      String
  redeemedAt            DateTime @default(now())

  // Value
  discountApplied       Decimal  @db.Decimal(10, 2)

  // Status
  status                RedemptionStatus @default(ACTIVE)

  // Timestamps
  createdAt             DateTime @default(now())

  // Relations
  coupon                Coupon       @relation(fields: [couponId], references: [id])
  subscription          Subscription @relation(fields: [subscriptionId], references: [id])

  @@index([couponId])
  @@index([subscriptionId])
}

enum RedemptionStatus {
  ACTIVE
  EXPIRED
  REVOKED
}
```

### Coach Domain

```prisma
// schema.prisma - Coach Domain

model CoachProfile {
  id                    String   @id @default(uuid())
  userId                String   @unique

  // Profile
  bio                   String?
  certifications        String[]
  sports                SportType[]
  hourlyRate            Decimal? @db.Decimal(10, 2)
  profilePhotoUrl       String?

  // Status
  status                CoachStatus @default(PENDING)

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  user                  User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  facilityAffiliations  CoachFacilityAffiliation[]
  students              CoachStudent[]
  availability          CoachAvailability[]
  blocks                CoachBlock[]
  sessions              CoachingSession[]
  sessionFeedback       SessionFeedback[]
  skillProgress         PlayerSkillProgress[]
}

enum CoachStatus {
  PENDING
  ACTIVE
  INACTIVE
  SUSPENDED
}

model CoachFacilityAffiliation {
  id                    String   @id @default(uuid())
  coachId               String
  facilityId            String

  // Status
  status                AffiliationStatus @default(PENDING)

  // Permissions
  permissions           String[]

  // Rates
  commissionRate        Decimal? @db.Decimal(5, 4)
  facilityRate          Decimal? @db.Decimal(10, 2)

  // Timestamps
  startedAt             DateTime @default(now())
  endedAt               DateTime?

  // Relations
  coach                 CoachProfile @relation(fields: [coachId], references: [id], onDelete: Cascade)
  facility              Facility     @relation(fields: [facilityId], references: [id], onDelete: Cascade)

  @@unique([coachId, facilityId])
}

enum AffiliationStatus {
  PENDING
  ACTIVE
  SUSPENDED
  ENDED
}

model CoachStudent {
  id                    String   @id @default(uuid())
  coachId               String
  studentUserId         String
  parentUserId          String?
  facilityId            String

  // Status
  status                StudentStatus @default(ACTIVE)

  // Skill
  sport                 SportType
  skillLevel            Decimal? @db.Decimal(3, 1)

  // Timestamps
  startedAt             DateTime @default(now())
  endedAt               DateTime?

  // Relations
  coach                 CoachProfile @relation(fields: [coachId], references: [id], onDelete: Cascade)

  @@unique([coachId, studentUserId, facilityId])
}

enum StudentStatus {
  ACTIVE
  INACTIVE
  GRADUATED
}

model CoachAvailability {
  id                    String   @id @default(uuid())
  coachId               String
  facilityId            String

  // Schedule
  dayOfWeek             Int      // 0 = Sunday
  startTime             String   // HH:MM
  endTime               String   // HH:MM
  isAvailable           Boolean  @default(true)

  // Relations
  coach                 CoachProfile @relation(fields: [coachId], references: [id], onDelete: Cascade)

  @@index([coachId, facilityId])
}

model CoachBlock {
  id                    String   @id @default(uuid())
  coachId               String
  facilityId            String
  resourceId            String

  // Time
  startAt               DateTime
  endAt                 DateTime

  // Type
  blockType             CoachBlockType
  reason                String?

  // Recurring
  recurrenceRule        String?
  parentBlockId         String?

  // Approval
  status                BlockStatus @default(PENDING)
  requiresApproval      Boolean  @default(true)
  approvedBy            String?
  approvedAt            DateTime?
  rejectionReason       String?

  // Notification
  notificationSent      Boolean  @default(false)
  notificationSentAt    DateTime?

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  coach                 CoachProfile @relation(fields: [coachId], references: [id], onDelete: Cascade)
  resource              Resource     @relation(fields: [resourceId], references: [id])
  parentBlock           CoachBlock?  @relation("BlockHierarchy", fields: [parentBlockId], references: [id])
  childBlocks           CoachBlock[] @relation("BlockHierarchy")
  approvalHistory       BlockApprovalHistory[]

  @@index([coachId, facilityId])
  @@index([resourceId, startAt, endAt])
}

enum CoachBlockType {
  LESSON
  RECURRING_LESSON
  PERSONAL
  FACILITY_REQUESTED
}

enum BlockStatus {
  PENDING
  APPROVED
  REJECTED
  CANCELLED
}

model BlockApprovalHistory {
  id                    String   @id @default(uuid())
  blockId               String
  action                BlockAction
  actorId               String
  notes                 String?

  // Timestamp
  createdAt             DateTime @default(now())

  // Relations
  block                 CoachBlock @relation(fields: [blockId], references: [id], onDelete: Cascade)

  @@index([blockId])
}

enum BlockAction {
  SUBMITTED
  APPROVED
  REJECTED
  MODIFIED
  CANCELLED
}
```

### Feedback Domain

```prisma
// schema.prisma - Feedback Domain

model CoachingSession {
  id                    String   @id @default(uuid())
  bookingId             String   @unique
  coachId               String
  facilityId            String

  // Session type
  sessionType           SessionType

  // Details
  plannedFocus          String?
  actualNotes           String?
  drillsUsed            Json?    // [{name, duration, notes}]

  // Attendance
  expectedStudents      String[]
  attendedStudents      String[]

  // Status
  status                SessionStatus @default(SCHEDULED)

  // Timestamps
  startedAt             DateTime?
  endedAt               DateTime?
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  booking               Booking        @relation(fields: [bookingId], references: [id])
  coach                 CoachProfile   @relation(fields: [coachId], references: [id])
  feedback              SessionFeedback?
}

enum SessionType {
  PRIVATE
  SEMI_PRIVATE
  GROUP
  CLINIC
  CAMP
}

enum SessionStatus {
  SCHEDULED
  IN_PROGRESS
  COMPLETED
  CANCELLED
  NO_SHOW
}

model SessionFeedback {
  id                    String   @id @default(uuid())
  coachingSessionId     String   @unique
  coachId               String

  // Ratings
  overallRating         Int?     // 1-5
  effortRating          Int?     // 1-5
  focusRating           Int?     // 1-5

  // Content
  summary               String?
  aiEnhancedSummary     String?
  parentFriendlySummary String?

  // Extracted data
  skillsWorked          Json?    // [{skillId, name, status, notes}]
  keyInsights           String[]

  // Homework
  homework              String?
  focusForNextSession   String?

  // AI data
  aiInsights            Json?
  aiSuggestedDrills     Json?

  // Status
  status                FeedbackStatus @default(DRAFT)
  publishedAt           DateTime?

  // Timestamps
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  coachingSession       CoachingSession @relation(fields: [coachingSessionId], references: [id])
  coach                 CoachProfile    @relation(fields: [coachId], references: [id])
  media                 FeedbackMedia[]
}

enum FeedbackStatus {
  DRAFT
  ENHANCING
  PENDING_REVIEW
  PUBLISHED
  ARCHIVED
}

model FeedbackMedia {
  id                    String   @id @default(uuid())
  sessionFeedbackId     String

  // Type
  mediaType             MediaType

  // Storage
  originalUrl           String
  thumbnailUrl          String?
  processedUrl          String?

  // Metadata
  fileName              String?
  fileSizeBytes         BigInt?
  durationSeconds       Int?
  mimeType              String?

  // AI analysis
  aiAnalysis            Json?
  aiAnnotations         Json?    // [{timestamp, annotation}]
  transcription         String?

  // Processing
  processingStatus      ProcessingStatus @default(PENDING)

  // Privacy
  visibility            MediaVisibility @default(PARENTS_AND_STUDENT)

  // Timestamps
  uploadedAt            DateTime @default(now())

  // Relations
  sessionFeedback       SessionFeedback @relation(fields: [sessionFeedbackId], references: [id], onDelete: Cascade)

  @@index([sessionFeedbackId])
}

enum MediaType {
  VIDEO
  PHOTO
  AUDIO
}

enum ProcessingStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
}

enum MediaVisibility {
  COACH_ONLY
  PARENTS_ONLY
  PARENTS_AND_STUDENT
}

model Skill {
  id                    String   @id @default(uuid())
  sportType             SportType
  category              String   // 'forehand', 'serve', 'footwork'
  name                  String
  description           String?
  difficultyLevel       Int      // 1-10
  prerequisites         String[] // Skill IDs

  // Assessment
  assessmentRubric      Json?

  // Timestamps
  createdAt             DateTime @default(now())

  // Relations
  progress              PlayerSkillProgress[]

  @@unique([sportType, category, name])
}

model PlayerSkillProgress {
  id                    String   @id @default(uuid())
  playerId              String
  skillId               String
  coachId               String

  // Levels
  currentLevel          Decimal  @db.Decimal(3, 1)
  targetLevel           Decimal? @db.Decimal(3, 1)

  // History
  levelHistory          Json     // [{date, level, coachId, notes}]

  // AI predictions
  predictedTimeToTarget Int?     // Days
  improvementRate       Decimal? @db.Decimal(5, 4)

  // Timestamps
  lastAssessedAt        DateTime @default(now())
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // Relations
  skill                 Skill        @relation(fields: [skillId], references: [id])
  coach                 CoachProfile @relation(fields: [coachId], references: [id])

  @@unique([playerId, skillId])
  @@index([playerId])
}

model PlayerProgressSummary {
  id                    String   @id @default(uuid())
  playerId              String
  sportType             SportType

  // Stats
  totalSessions         Int      @default(0)
  totalHours            Decimal  @db.Decimal(10, 2) @default(0)
  currentStreakDays     Int      @default(0)
  longestStreakDays     Int      @default(0)

  // Skill summary
  skillsCount           Int      @default(0)
  skillsImproving       Int      @default(0)
  skillsMastered        Int      @default(0)

  // AI generated
  overallTrend          Trend?
  aiSummary             String?
  aiRecommendations     Json?

  // Achievements
  achievements          Json?    // [{id, name, earnedAt}]

  // Timestamps
  lastUpdatedAt         DateTime @default(now())

  @@unique([playerId, sportType])
}

enum Trend {
  IMPROVING
  STABLE
  DECLINING
}

model Drill {
  id                    String   @id @default(uuid())
  sportType             SportType
  name                  String
  description           String?

  // Categorization
  skillIds              String[]
  difficultyLevel       Int      // 1-10
  durationMinutes       Int?
  equipmentNeeded       String[]

  // Media
  demoVideoUrl          String?
  diagramUrl            String?
  instructions          Json?

  // AI metadata
  aiTags                String[]
  effectivenessScore    Decimal? @db.Decimal(3, 2)

  // Source
  createdByCoachId      String?
  isPublic              Boolean  @default(false)

  // Timestamps
  createdAt             DateTime @default(now())

  @@index([sportType])
}
```

---

## Database Indexes

```sql
-- Critical indexes for performance

-- Booking queries
CREATE INDEX idx_bookings_facility_date ON bookings (facility_id, start_at);
CREATE INDEX idx_bookings_resource_date ON bookings (resource_id, start_at);
CREATE INDEX idx_bookings_booker ON bookings (booker_id);
CREATE INDEX idx_bookings_status ON bookings (status) WHERE status IN ('HELD', 'CONFIRMED');

-- Availability queries
CREATE INDEX idx_resource_availability_resource ON resource_availability (resource_id);

-- User queries
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_external_id ON users (external_id);

-- Payment queries
CREATE INDEX idx_transactions_billing ON transactions (billing_account_id);
CREATE INDEX idx_transactions_reference ON transactions (reference_type, reference_id);
CREATE INDEX idx_transactions_stripe ON transactions (stripe_payment_intent_id);

-- Subscription queries
CREATE INDEX idx_subscriptions_facility ON subscriptions (facility_id);
CREATE INDEX idx_subscriptions_status ON subscriptions (status);

-- Coach queries
CREATE INDEX idx_coach_blocks_resource_time ON coach_blocks (resource_id, start_at, end_at);
CREATE INDEX idx_coach_students_coach ON coach_students (coach_id);

-- Feedback queries
CREATE INDEX idx_session_feedback_coach ON session_feedback (coach_id);
CREATE INDEX idx_skill_progress_player ON player_skill_progress (player_id);

-- Event log (for analytics)
CREATE INDEX idx_booking_events_type_time ON booking_events (event_type, occurred_at);
```

---

*Last Updated: 2024-01-10*
*Version: 1.0*
