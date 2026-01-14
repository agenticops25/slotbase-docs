# SlotBase Payment System Design

## Overview

SlotBase supports flexible payment collection with both online (Stripe) and offline payment methods. This document defines the complete payment architecture including flows, configurations, and edge cases.

---

## Payment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           PAYMENT SYSTEM ARCHITECTURE                            │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                              PAYMENT METHODS                                  │
├─────────────────┬─────────────────┬─────────────────┬────────────────────────┤
│   Credit Card   │   Apple Pay     │   Offline       │   Account Credit      │
│   Debit Card    │   Google Pay    │   (Cash/Check)  │   (Wallet Balance)    │
└────────┬────────┴────────┬────────┴────────┬────────┴───────────┬────────────┘
         │                 │                 │                    │
         └─────────────────┴─────────────────┴────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           PAYMENT PROCESSOR                                   │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                        Stripe Connect                                    ││
│  │                                                                          ││
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 ││
│  │  │   SlotBase   │    │  Facility   │    │   Player    │                 ││
│  │  │  Platform   │    │  Account    │    │  Customer   │                 ││
│  │  │  (Connect)  │    │  (Express)  │    │             │                 ││
│  │  └─────────────┘    └─────────────┘    └─────────────┘                 ││
│  │                                                                          ││
│  │  Payment Flow: Player → SlotBase Platform → Facility Account             ││
│  │  Platform Fee: Deducted before payout to facility                       ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           PAYMENT ENTITIES                                    │
│                                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │ Billing Account │  │   Transaction   │  │    Invoice      │             │
│  │                 │  │                 │  │                 │             │
│  │ - owner_type    │  │ - amount        │  │ - line_items    │             │
│  │ - stripe_cust   │  │ - status        │  │ - total         │             │
│  │ - balance       │  │ - reference     │  │ - due_date      │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │ Payment Method  │  │ Offline Payment │  │     Refund      │             │
│  │                 │  │                 │  │                 │             │
│  │ - type          │  │ - method        │  │ - amount        │             │
│  │ - card_last4    │  │ - recorded_by   │  │ - reason        │             │
│  │ - is_default    │  │ - attachments   │  │ - status        │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Payment Models

### Model 1: Stripe Connect (Marketplace)

**Flow:**
```
Player pays $50 for booking
        │
        ▼
┌─────────────────┐
│  Stripe Charge  │  $50.00
└────────┬────────┘
         │
         ├─── Stripe Fee ──────────▶ -$1.75 (2.9% + $0.30)
         │
         ├─── SlotBase Fee ─────────▶ -$1.00 (2% platform fee)
         │
         └─── Facility Payout ─────▶ $47.25
```

**Configuration:**
```typescript
interface StripeConnectConfig {
  // Platform
  platformAccountId: string;        // SlotBase's Stripe account
  platformFeePercent: number;       // e.g., 2.0 for 2%
  platformFeeFixed: number;         // e.g., 0 (no fixed fee)

  // Facility (per facility)
  facilityAccountId: string;        // Facility's Express account
  facilityAccountStatus: 'pending' | 'active' | 'restricted';

  // Payout settings
  payoutSchedule: 'daily' | 'weekly' | 'monthly';
  payoutDelay: number;              // Days to hold before payout
}
```

### Model 2: Offline Payments

**Supported Methods:**
- Cash
- Check
- Bank Transfer
- Venmo
- Zelle
- Other (custom)

**Flow:**
```
Player books without payment
        │
        ▼
┌─────────────────┐
│  Booking Created│  Status: UNPAID
└────────┬────────┘
         │
         ▼
Player pays at facility (cash/check/etc.)
         │
         ▼
┌─────────────────┐
│  Staff Records  │  Offline Payment
│    Payment      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Booking Updated │  Status: PAID
└─────────────────┘
```

---

## Facility Payment Configuration

### Configuration Options

```typescript
interface FacilityPaymentConfig {
  facilityId: string;

  // === STRIPE SETTINGS ===
  stripeEnabled: boolean;
  stripeAccountId?: string;
  stripeAccountStatus?: 'pending' | 'active' | 'restricted' | 'disabled';

  // What Stripe collects
  acceptCards: boolean;
  acceptApplePay: boolean;
  acceptGooglePay: boolean;
  acceptAch: boolean;            // US bank transfers

  // === OFFLINE SETTINGS ===
  acceptOfflinePayments: boolean;
  offlineMethods: {
    cash: boolean;
    check: boolean;
    bankTransfer: boolean;
    venmo: boolean;
    zelle: boolean;
    other: boolean;
  };
  offlinePaymentInstructions?: string;

  // === PAYMENT TIMING ===
  paymentTiming: 'at_booking' | 'before_session' | 'after_session' | 'invoice';

  // Prepayment requirements
  requirePrepayment: boolean;
  prepaymentPercent: number;      // 100 = full prepayment

  // === TAX SETTINGS ===
  collectSalesTax: boolean;
  taxRate?: number;               // e.g., 0.0825 for 8.25%
  taxId?: string;

  // === REFUND SETTINGS ===
  allowOnlineRefunds: boolean;
  refundToOriginalMethod: boolean;
  allowCreditRefunds: boolean;    // Refund as account credit

  // === CURRENCY ===
  defaultCurrency: string;        // 'USD'
}
```

### Onboarding Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    FACILITY PAYMENT ONBOARDING FLOW                              │
└─────────────────────────────────────────────────────────────────────────────────┘

Step 1: Choose Payment Methods
        │
        ├─── "I want online payments" ──────────▶ Enable Stripe
        │                                               │
        │                                               ▼
        │                                        ┌─────────────┐
        │                                        │Connect Stripe│
        │                                        │   Account    │
        │                                        └─────────────┘
        │                                               │
        │                                               ▼
        │                                        Stripe Express
        │                                        Onboarding Flow
        │
        └─── "I'll collect payments myself" ────▶ Offline Only
                                                        │
                                                        ▼
                                                 Select Methods:
                                                 ☑ Cash
                                                 ☑ Check
                                                 ☐ Venmo
                                                 ☐ Zelle

Step 2: Payment Timing
        │
        ├─── "Require payment at booking" ──────▶ prepaymentPercent: 100
        │
        ├─── "Payment before session" ──────────▶ Reminder sent 24h before
        │
        └─── "Invoice monthly" ─────────────────▶ Monthly billing cycle

Step 3: Tax Settings (Optional)
        │
        └─── "I need to collect sales tax" ─────▶ Configure tax rate
```

---

## Payment Flows

### Flow 1: Online Payment at Booking

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     ONLINE PAYMENT AT BOOKING FLOW                               │
└─────────────────────────────────────────────────────────────────────────────────┘

User selects slot
        │
        ▼
┌─────────────────┐
│  Create Hold    │  booking.status = HELD
│  (5 min expiry) │  booking_hold.expires_at = now + 5min
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Create Payment  │  stripe.paymentIntents.create({
│    Intent       │    amount,
│                 │    customer,
│                 │    transfer_data: { destination: facilityAccountId }
│                 │  })
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Show Payment  │  Stripe Elements / Payment Sheet
│      Form       │
└────────┬────────┘
         │
         │◄────────────── User enters card ──────────────
         │
         ▼
┌─────────────────┐
│ Confirm Payment │  stripe.confirmPayment()
│                 │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
Success    Failure
    │         │
    ▼         ▼
┌─────────┐ ┌─────────┐
│ Confirm │ │ Release │
│ Booking │ │  Hold   │
│         │ │         │
│ Status: │ │ Status: │
│CONFIRMED│ │CANCELLED│
└─────────┘ └─────────┘
    │
    ▼
Emit: booking.confirmed
Emit: payment.succeeded
```

### Flow 2: Offline Payment

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        OFFLINE PAYMENT FLOW                                      │
└─────────────────────────────────────────────────────────────────────────────────┘

User selects slot (no prepayment required)
        │
        ▼
┌─────────────────┐
│ Create Booking  │  booking.status = CONFIRMED
│ (Unpaid)        │  booking_payment.status = PENDING
└────────┬────────┘
         │
         ▼
Emit: booking.confirmed (payment_method: 'offline')

         │
         │  ... Time passes, user arrives at facility ...
         │
         ▼
┌─────────────────┐
│  User Pays at   │  Cash/Check/Venmo/etc.
│   Facility      │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STAFF RECORDS OFFLINE PAYMENT                             │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Payment Method:  [Cash ▼]                                           │   │
│  │                                                                       │   │
│  │  Amount Received: [$45.00]                                           │   │
│  │                                                                       │   │
│  │  Payment Date:    [01/15/2024]                                       │   │
│  │                                                                       │   │
│  │  Reference #:     [___________]  (Optional: check number)            │   │
│  │                                                                       │   │
│  │  Notes:          [___________]                                       │   │
│  │                                                                       │   │
│  │  📷 [Attach Receipt Photo]                                           │   │
│  │                                                                       │   │
│  │                    [ Record Payment ]                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│ Create Offline  │  offline_payments record created
│ Payment Record  │  booking_payment.status = PAID
└────────┬────────┘
         │
         ▼
Emit: offline_payment.recorded
```

### Flow 3: Subscription Payment

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     SUBSCRIPTION PAYMENT FLOW                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

Facility selects plan during onboarding
        │
        ▼
┌─────────────────┐
│ Create Stripe   │  Subscription with trial
│  Subscription   │
└────────┬────────┘
         │
    ┌────┴────────────────────────────────────────┐
    │                                             │
    ▼                                             ▼
┌─────────────────┐                    ┌─────────────────┐
│  Trial Period   │  (14 days)         │ Payment Method  │
│                 │                     │    Required     │
└────────┬────────┘                    └─────────────────┘
         │
         ▼
┌─────────────────┐
│ Trial Ends      │
│                 │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
Card on   No Card
 File        │
    │        ▼
    │   ┌─────────────────┐
    │   │ Send Payment    │
    │   │ Request Email   │
    │   └────────┬────────┘
    │            │
    ▼            ▼
┌─────────────────┐
│ Charge Card     │
│                 │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
Success    Failure
    │         │
    ▼         ▼
┌─────────┐ ┌─────────────┐
│ Active  │ │ Past Due    │
│         │ │             │
│ Renew   │ │ Retry Logic │
│ monthly │ │ (3 attempts)│
└─────────┘ └─────────────┘
```

---

## Pricing & Fee Structure

### Platform Fees

```typescript
interface PlatformFeeConfig {
  // Per transaction
  bookingFee: {
    type: 'percentage' | 'fixed' | 'percentage_plus_fixed';
    percentageValue: number;      // e.g., 2.0 for 2%
    fixedValue: number;           // e.g., 0.30
    minFee: number;               // Minimum fee
    maxFee: number;               // Maximum fee (cap)
  };

  // Per subscription tier (override)
  tierOverrides: {
    starter: { percentageValue: 2.5 },
    growth: { percentageValue: 2.0 },
    pro: { percentageValue: 1.5 },
    enterprise: { percentageValue: 1.0 },
  };
}

// Default configuration
const DEFAULT_FEE_CONFIG: PlatformFeeConfig = {
  bookingFee: {
    type: 'percentage',
    percentageValue: 2.0,
    fixedValue: 0,
    minFee: 0.50,
    maxFee: 10.00,
  },
  tierOverrides: {
    starter: { percentageValue: 2.5 },
    growth: { percentageValue: 2.0 },
    pro: { percentageValue: 1.5 },
    enterprise: { percentageValue: 1.0 },
  },
};
```

### Fee Calculation Example

```typescript
function calculateFees(
  bookingAmount: number,
  facilityTier: string,
  currency: string
): FeeBreakdown {
  const config = getFeeConfig(facilityTier);

  // Platform fee
  let platformFee = bookingAmount * (config.percentageValue / 100);
  platformFee = Math.max(platformFee, config.minFee);
  platformFee = Math.min(platformFee, config.maxFee);

  // Stripe fee (approximate)
  const stripeFee = bookingAmount * 0.029 + 0.30;

  // Net to facility
  const netToFacility = bookingAmount - platformFee - stripeFee;

  return {
    grossAmount: bookingAmount,
    platformFee: round(platformFee, 2),
    stripeFee: round(stripeFee, 2),
    netToFacility: round(netToFacility, 2),
    currency,
  };
}

// Example: $50 booking on Growth tier
// Gross: $50.00
// Stripe: $1.75 (2.9% + $0.30)
// Platform: $1.00 (2%)
// Net to Facility: $47.25
```

---

## Refund System

### Refund Policies

```typescript
interface RefundPolicy {
  // Cancellation-based refunds
  cancellation: {
    rules: Array<{
      hoursBeforeStart: number;
      refundPercent: number;
    }>;
    // Example:
    // > 24 hours: 100% refund
    // 2-24 hours: 50% refund
    // < 2 hours: 0% refund
  };

  // No-show policy
  noShow: {
    chargePercent: number;        // 100 = charge full amount
    allowWaiver: boolean;         // Staff can waive
  };

  // Facility cancellation
  facilityCancellation: {
    refundPercent: number;        // Usually 100%
    creditBonus: number;          // Extra credit as apology
  };

  // Weather/emergency
  weatherPolicy: {
    autoCancel: boolean;
    refundPercent: number;
  };
}

// Default policy
const DEFAULT_REFUND_POLICY: RefundPolicy = {
  cancellation: {
    rules: [
      { hoursBeforeStart: 24, refundPercent: 100 },
      { hoursBeforeStart: 2, refundPercent: 50 },
      { hoursBeforeStart: 0, refundPercent: 0 },
    ],
  },
  noShow: {
    chargePercent: 100,
    allowWaiver: true,
  },
  facilityCancellation: {
    refundPercent: 100,
    creditBonus: 0,
  },
  weatherPolicy: {
    autoCancel: false,
    refundPercent: 100,
  },
};
```

### Refund Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           REFUND PROCESSING FLOW                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

Refund Request (cancellation or manual)
        │
        ▼
┌─────────────────┐
│ Calculate Refund│  Based on policy + time to booking
│    Amount       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Determine Refund│
│    Method       │
└────────┬────────┘
         │
    ┌────┴────────────────────────────────────────┐
    │                                             │
    ▼                                             ▼
Original Payment                           Account Credit
was Online                                 (Always available)
    │                                             │
    ▼                                             ▼
┌─────────────────┐                    ┌─────────────────┐
│ Stripe Refund   │                    │ Add to Billing  │
│                 │                    │ Account Balance │
│ refunds.create({│                    │                 │
│   payment_intent│                    │ balance += amt  │
│   amount        │                    │                 │
│ })              │                    │                 │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         └──────────────────┬───────────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │ Create Refund   │
                    │   Record        │
                    └────────┬────────┘
                             │
                             ▼
                    Emit: payment.refunded
```

---

## Billing Account Management

### Account Types

```typescript
enum BillingAccountType {
  FACILITY = 'facility',    // Facility pays subscription
  PLAYER = 'player',        // Player pays for bookings
  ORGANIZATION = 'org',     // Multi-facility organization
}

interface BillingAccount {
  id: string;
  accountType: BillingAccountType;
  ownerType: 'User' | 'Facility' | 'Organization';
  ownerId: string;

  // Stripe
  stripeCustomerId?: string;
  defaultPaymentMethodId?: string;

  // Contact
  billingEmail: string;
  billingAddress?: Address;
  taxId?: string;

  // Balance (for credits/refunds)
  balance: number;
  currency: string;

  // Status
  status: 'active' | 'suspended' | 'closed';
}
```

### Account Credit System

```typescript
interface AccountCredit {
  id: string;
  billingAccountId: string;

  // Credit details
  amount: number;
  originalAmount: number;
  usedAmount: number;
  currency: string;

  // Source
  source: 'refund' | 'promotion' | 'compensation' | 'manual';
  sourceReferenceId?: string;

  // Validity
  expiresAt?: Date;

  // Status
  status: 'active' | 'used' | 'expired' | 'cancelled';

  createdAt: Date;
  usedAt?: Date;
}

// Credit usage at checkout
function applyCredits(
  billingAccountId: string,
  amount: number
): CreditApplication {
  const availableCredits = await getAvailableCredits(billingAccountId);

  let remainingAmount = amount;
  const appliedCredits: AppliedCredit[] = [];

  for (const credit of availableCredits) {
    if (remainingAmount <= 0) break;

    const useAmount = Math.min(credit.amount - credit.usedAmount, remainingAmount);

    appliedCredits.push({
      creditId: credit.id,
      amount: useAmount,
    });

    remainingAmount -= useAmount;
  }

  return {
    originalAmount: amount,
    creditsApplied: amount - remainingAmount,
    amountDue: remainingAmount,
    appliedCredits,
  };
}
```

---

## Offline Payment Aging & Collection

### Aging Report

```typescript
interface AgingReport {
  facilityId: string;
  generatedAt: Date;

  summary: {
    totalOutstanding: number;
    current: number;           // 0-7 days
    days7to14: number;
    days14to30: number;
    days30to60: number;
    over60Days: number;
  };

  bookings: Array<{
    bookingId: string;
    playerName: string;
    playerEmail: string;
    amount: number;
    bookingDate: Date;
    daysOutstanding: number;
    remindersSent: number;
    lastReminderAt?: Date;
  }>;
}
```

### Collection Actions

```typescript
const AGING_ACTIONS = [
  {
    daysOutstanding: 7,
    action: 'send_reminder',
    template: 'payment_reminder_friendly',
    channels: ['email'],
  },
  {
    daysOutstanding: 14,
    action: 'send_reminder',
    template: 'payment_reminder_urgent',
    channels: ['email', 'sms'],
  },
  {
    daysOutstanding: 30,
    action: 'flag_for_review',
    notifyFacilityAdmin: true,
  },
  {
    daysOutstanding: 45,
    action: 'send_final_notice',
    template: 'payment_final_notice',
    channels: ['email'],
  },
  {
    daysOutstanding: 60,
    action: 'restrict_booking',
    allowOverride: true,
  },
];
```

---

## Stripe Integration Details

### Stripe Connect Setup

```typescript
// Create Express account for facility
async function createFacilityStripeAccount(
  facilityId: string,
  facilityData: FacilityData
): Promise<string> {
  const account = await stripe.accounts.create({
    type: 'express',
    country: facilityData.country,
    email: facilityData.email,
    capabilities: {
      card_payments: { requested: true },
      transfers: { requested: true },
    },
    business_type: 'company',
    business_profile: {
      name: facilityData.name,
      mcc: '7941', // Sports clubs/fields
      url: facilityData.website,
    },
    metadata: {
      facilityId,
      platform: 'slotbase',
    },
  });

  // Generate onboarding link
  const accountLink = await stripe.accountLinks.create({
    account: account.id,
    refresh_url: `${APP_URL}/facility/stripe/refresh`,
    return_url: `${APP_URL}/facility/stripe/complete`,
    type: 'account_onboarding',
  });

  return accountLink.url;
}
```

### Payment Intent with Transfer

```typescript
async function createBookingPayment(
  bookingId: string,
  amount: number,
  facilityStripeAccountId: string,
  customerStripeId: string
): Promise<Stripe.PaymentIntent> {
  const platformFee = calculatePlatformFee(amount);

  const paymentIntent = await stripe.paymentIntents.create({
    amount: Math.round(amount * 100), // Cents
    currency: 'usd',
    customer: customerStripeId,
    payment_method_types: ['card'],

    // Transfer to facility
    transfer_data: {
      destination: facilityStripeAccountId,
    },

    // Platform fee
    application_fee_amount: Math.round(platformFee * 100),

    // Metadata
    metadata: {
      bookingId,
      platform: 'slotbase',
    },

    // Description for statement
    statement_descriptor: 'NEXPLAY BOOKING',
    statement_descriptor_suffix: bookingId.slice(0, 10),
  });

  return paymentIntent;
}
```

### Webhook Handling

```typescript
// Stripe webhook events to handle
const STRIPE_WEBHOOK_EVENTS = [
  // Payment events
  'payment_intent.succeeded',
  'payment_intent.payment_failed',

  // Subscription events
  'customer.subscription.created',
  'customer.subscription.updated',
  'customer.subscription.deleted',
  'invoice.paid',
  'invoice.payment_failed',

  // Connect events
  'account.updated',
  'account.application.authorized',
  'account.application.deauthorized',

  // Dispute events
  'charge.dispute.created',
  'charge.dispute.updated',
  'charge.dispute.closed',

  // Refund events
  'charge.refunded',
];

// Webhook handler
async function handleStripeWebhook(
  event: Stripe.Event
): Promise<void> {
  switch (event.type) {
    case 'payment_intent.succeeded':
      await handlePaymentSuccess(event.data.object as Stripe.PaymentIntent);
      break;

    case 'payment_intent.payment_failed':
      await handlePaymentFailure(event.data.object as Stripe.PaymentIntent);
      break;

    case 'customer.subscription.updated':
      await handleSubscriptionUpdate(event.data.object as Stripe.Subscription);
      break;

    case 'charge.dispute.created':
      await handleDisputeCreated(event.data.object as Stripe.Dispute);
      break;

    // ... more handlers
  }
}
```

---

## Security & Compliance

### PCI Compliance

```typescript
// PCI DSS compliance measures
const PCI_COMPLIANCE = {
  // Never store
  neverStore: [
    'Full card number (PAN)',
    'CVV/CVC',
    'PIN',
    'Full magnetic stripe data',
  ],

  // Use Stripe for
  useStripeFor: [
    'Card tokenization',
    'Payment processing',
    'Card storage (via Stripe Customer)',
  ],

  // We store
  weStore: [
    'Last 4 digits (for display)',
    'Card brand',
    'Expiration month/year',
    'Stripe token references',
  ],
};
```

### Fraud Prevention

```typescript
interface FraudPreventionConfig {
  // Stripe Radar
  radarEnabled: boolean;
  radarRules: string[];

  // Custom rules
  maxTransactionsPerDay: number;
  maxAmountPerTransaction: number;
  requireCvv: boolean;
  require3ds: boolean;

  // Velocity checks
  velocityChecks: {
    sameCardPerHour: number;
    sameIpPerHour: number;
    failedAttemptsBeforeBlock: number;
  };

  // High-risk indicators
  blockHighRisk: {
    prepaidCards: boolean;
    internationalCards: boolean;
    missingAvs: boolean;
  };
}
```

---

## Reporting

### Revenue Reports

```typescript
interface RevenueReport {
  facilityId: string;
  period: {
    start: Date;
    end: Date;
  };

  summary: {
    grossRevenue: number;
    refunds: number;
    netRevenue: number;
    platformFees: number;
    stripeFees: number;
    netPayout: number;
  };

  byPaymentMethod: {
    online: { count: number; amount: number };
    cash: { count: number; amount: number };
    check: { count: number; amount: number };
    other: { count: number; amount: number };
  };

  byBookingType: {
    oneTime: { count: number; amount: number };
    recurring: { count: number; amount: number };
    lesson: { count: number; amount: number };
  };

  byResource: Array<{
    resourceId: string;
    resourceName: string;
    bookings: number;
    revenue: number;
  }>;

  dailyBreakdown: Array<{
    date: Date;
    revenue: number;
    bookings: number;
  }>;
}
```

---

*Last Updated: 2024-01-10*
*Version: 1.0*
