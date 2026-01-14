# Sprint 16: Player Booking & Production Readiness

> **Design Authority**: Principal Architect
> **Implementation**: Codex (Senior Developer)
> **Duration**: 2 weeks (10 working days)
> **Pilot Sport**: Cricket (Cricket nets, bowling machines, practice lanes)

---

## Executive Summary

Sprint 16 delivers the **player-facing booking experience** required for production launch with our pilot cricket facility. This sprint transforms SlotBase from an admin-only tool into a complete booking platform.

### Sprint Goals
1. Players can discover and view the pilot cricket facility
2. Players can see real-time availability of cricket nets/lanes
3. Players can book Open Play sessions (casual practice)
4. Players can book Lessons with coaches
5. Players receive email confirmations
6. Facility admins can manage all bookings
7. Basic subscription tier enforcement (Free tier: 20 players, 3 resources)

### Out of Scope (Deferred)
- Stripe Connect integration
- Recurring bookings
- Waitlist functionality
- Mobile app feature screens
- SMS/WhatsApp notifications
- Analytics dashboards

---

## Daily Task Breakdown for Codex

### Pre-Sprint Checklist
Before starting any day, Codex should:
1. Pull latest from all repos (api, web, mobile)
2. Run `pnpm install` in each project
3. Ensure local database is migrated: `pnpm prisma migrate dev`
4. Start API server: `pnpm dev` (port 3001)
5. Start Web server: `pnpm dev` (port 3000)

---

## Day 1: Database Schema Updates

### Objective
Add missing tables for subscription usage tracking and update seed data for cricket.

### Tasks

#### 1.1 Add SubscriptionUsage Model
**File**: `api/prisma/schema.prisma`

```prisma
model SubscriptionUsage {
  id              String   @id @default(uuid()) @db.Uuid
  subscriptionId  String   @db.Uuid
  subscription    Subscription @relation(fields: [subscriptionId], references: [id], onDelete: Cascade)

  periodStart     DateTime @db.Timestamptz
  periodEnd       DateTime @db.Timestamptz

  resourceCount   Int      @default(0)
  playerCount     Int      @default(0)
  staffCount      Int      @default(0)
  coachCount      Int      @default(0)
  bookingCount    Int      @default(0)

  createdAt       DateTime @default(now()) @db.Timestamptz
  updatedAt       DateTime @updatedAt @db.Timestamptz

  @@unique([subscriptionId, periodStart])
  @@index([subscriptionId])
  @@index([periodStart])
}
```

#### 1.2 Add TierFeature Model
**File**: `api/prisma/schema.prisma`

```prisma
model TierFeature {
  id          String           @id @default(uuid()) @db.Uuid
  tier        SubscriptionTier
  featureKey  String           // 'online_payments', 'recurring_bookings', 'lessons', etc.
  enabled     Boolean          @default(true)
  limitValue  Int?             // null = unlimited

  createdAt   DateTime         @default(now()) @db.Timestamptz

  @@unique([tier, featureKey])
  @@index([tier])
}
```

#### 1.3 Update Seed Data for Cricket
**File**: `api/prisma/seed.ts`

Replace tennis references with cricket:

```typescript
// Pilot facility: Cricket Academy
const facility = await prisma.facility.create({
  data: {
    name: 'Pilot Cricket Academy',
    slug: 'pilot-cricket-academy',
    description: 'Premier cricket training facility with indoor nets and bowling machines',
    sportTypes: ['CRICKET'],
    address: {
      street: '456 Cricket Lane',
      city: 'San Francisco',
      state: 'CA',
      zip: '94102',
      country: 'US'
    },
    coordinates: { lat: 37.7749, lng: -122.4194 },
    timezone: 'America/Los_Angeles',
    status: 'ACTIVE',
    // ... rest of facility data
  }
});

// Resources: Cricket Nets and Bowling Machines
const resources = [
  { name: 'Net 1 - Indoor', type: 'CRICKET_NET', attributes: { surface: 'artificial', lighting: true, bowlingMachine: false } },
  { name: 'Net 2 - Indoor', type: 'CRICKET_NET', attributes: { surface: 'artificial', lighting: true, bowlingMachine: false } },
  { name: 'Net 3 - Machine Lane', type: 'CRICKET_NET', attributes: { surface: 'artificial', lighting: true, bowlingMachine: true, machineType: 'Bowling Machine Pro' } },
  { name: 'Net 4 - Outdoor', type: 'CRICKET_NET', attributes: { surface: 'turf', lighting: true, bowlingMachine: false } },
  { name: 'Bowling Machine Bay', type: 'BOWLING_MACHINE', attributes: { machineType: 'BOLA Professional', speedRange: '40-90 mph' } },
];
```

#### 1.4 Seed Tier Features
**File**: `api/prisma/seed.ts`

```typescript
// Tier feature limits
const tierFeatures = [
  // STARTER (Free)
  { tier: 'STARTER', featureKey: 'max_resources', limitValue: 3 },
  { tier: 'STARTER', featureKey: 'max_players', limitValue: 20 },
  { tier: 'STARTER', featureKey: 'max_staff', limitValue: 1 },
  { tier: 'STARTER', featureKey: 'max_coaches', limitValue: 1 },
  { tier: 'STARTER', featureKey: 'online_payments', enabled: false },
  { tier: 'STARTER', featureKey: 'recurring_bookings', enabled: false },
  { tier: 'STARTER', featureKey: 'lessons', enabled: true },

  // GROWTH
  { tier: 'GROWTH', featureKey: 'max_resources', limitValue: 10 },
  { tier: 'GROWTH', featureKey: 'max_players', limitValue: 200 },
  { tier: 'GROWTH', featureKey: 'max_staff', limitValue: 3 },
  { tier: 'GROWTH', featureKey: 'max_coaches', limitValue: 5 },
  { tier: 'GROWTH', featureKey: 'online_payments', enabled: true },
  { tier: 'GROWTH', featureKey: 'recurring_bookings', enabled: true, limitValue: 4 }, // weekly only
  { tier: 'GROWTH', featureKey: 'lessons', enabled: true },

  // PRO
  { tier: 'PRO', featureKey: 'max_resources', limitValue: 50 },
  { tier: 'PRO', featureKey: 'max_players', limitValue: 2000 },
  { tier: 'PRO', featureKey: 'max_staff', limitValue: 10 },
  { tier: 'PRO', featureKey: 'max_coaches', limitValue: 20 },
  { tier: 'PRO', featureKey: 'online_payments', enabled: true },
  { tier: 'PRO', featureKey: 'recurring_bookings', enabled: true },
  { tier: 'PRO', featureKey: 'lessons', enabled: true },

  // ENTERPRISE (unlimited)
  { tier: 'ENTERPRISE', featureKey: 'max_resources', limitValue: null },
  { tier: 'ENTERPRISE', featureKey: 'max_players', limitValue: null },
  { tier: 'ENTERPRISE', featureKey: 'max_staff', limitValue: null },
  { tier: 'ENTERPRISE', featureKey: 'max_coaches', limitValue: null },
  { tier: 'ENTERPRISE', featureKey: 'online_payments', enabled: true },
  { tier: 'ENTERPRISE', featureKey: 'recurring_bookings', enabled: true },
  { tier: 'ENTERPRISE', featureKey: 'lessons', enabled: true },
];
```

### Deliverables
- [ ] Schema updated with SubscriptionUsage and TierFeature models
- [ ] Migration created and applied
- [ ] Seed script updated with cricket facility data
- [ ] Database reset and seeded successfully

### Verification
```bash
pnpm prisma migrate dev --name add_tier_features
pnpm prisma db seed
```

---

## Day 2: Availability API Endpoint

### Objective
Create the availability endpoint that returns bookable time slots for a facility/resource.

### Tasks

#### 2.1 Create Availability DTO
**File**: `api/src/booking/dto/availability.dto.ts`

```typescript
import { IsDateString, IsOptional, IsUUID } from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class GetAvailabilityDto {
  @ApiProperty({ description: 'Date to check availability (YYYY-MM-DD)' })
  @IsDateString()
  date: string;

  @ApiPropertyOptional({ description: 'Filter by specific resource' })
  @IsOptional()
  @IsUUID()
  resourceId?: string;
}

export class TimeSlotDto {
  @ApiProperty()
  startTime: string; // HH:mm format

  @ApiProperty()
  endTime: string;

  @ApiProperty()
  available: boolean;

  @ApiProperty()
  price: number;

  @ApiPropertyOptional()
  holdId?: string; // If current user has a hold
}

export class ResourceAvailabilityDto {
  @ApiProperty()
  resourceId: string;

  @ApiProperty()
  resourceName: string;

  @ApiProperty()
  resourceType: string;

  @ApiProperty({ type: [TimeSlotDto] })
  slots: TimeSlotDto[];
}

export class AvailabilityResponseDto {
  @ApiProperty()
  facilityId: string;

  @ApiProperty()
  date: string;

  @ApiProperty()
  timezone: string;

  @ApiProperty({ type: [ResourceAvailabilityDto] })
  resources: ResourceAvailabilityDto[];
}
```

#### 2.2 Create Availability Service Method
**File**: `api/src/booking/booking.service.ts`

Add method:

```typescript
async getAvailability(
  facilityId: string,
  date: string,
  resourceId?: string,
  userId?: string,
): Promise<AvailabilityResponseDto> {
  const facility = await this.prisma.facility.findUnique({
    where: { id: facilityId },
    include: {
      resources: resourceId ? { where: { id: resourceId } } : true,
      operatingHours: true,
      holidays: true,
    },
  });

  if (!facility) {
    throw new NotFoundException('Facility not found');
  }

  const targetDate = new Date(date);
  const dayOfWeek = targetDate.getDay(); // 0 = Sunday

  // Check if holiday
  const isHoliday = facility.holidays.some(
    (h) => h.date.toISOString().split('T')[0] === date
  );

  // Get operating hours for this day
  const hours = facility.operatingHours.find(
    (h) => h.dayOfWeek === dayOfWeek
  );

  if (!hours || hours.isClosed || isHoliday) {
    return {
      facilityId,
      date,
      timezone: facility.timezone,
      resources: facility.resources.map((r) => ({
        resourceId: r.id,
        resourceName: r.name,
        resourceType: r.type,
        slots: [], // Closed
      })),
    };
  }

  // Get all bookings and holds for this date
  const startOfDay = new Date(`${date}T00:00:00`);
  const endOfDay = new Date(`${date}T23:59:59`);

  const [bookings, holds] = await Promise.all([
    this.prisma.booking.findMany({
      where: {
        facilityId,
        resourceId: resourceId || undefined,
        startAt: { gte: startOfDay, lte: endOfDay },
        status: { in: ['CONFIRMED', 'CHECKED_IN'] },
      },
    }),
    this.prisma.bookingHold.findMany({
      where: {
        resourceId: resourceId || undefined,
        startAt: { gte: startOfDay, lte: endOfDay },
        expiresAt: { gt: new Date() },
      },
    }),
  ]);

  // Generate time slots for each resource
  const resources = await Promise.all(
    facility.resources.map(async (resource) => {
      const pricing = await this.prisma.resourcePricing.findFirst({
        where: { resourceId: resource.id },
      });

      const slots = this.generateTimeSlots(
        hours.openTime,
        hours.closeTime,
        60, // 60-minute slots default
        bookings.filter((b) => b.resourceId === resource.id),
        holds.filter((h) => h.resourceId === resource.id),
        pricing,
        isHoliday,
        dayOfWeek,
        userId,
      );

      return {
        resourceId: resource.id,
        resourceName: resource.name,
        resourceType: resource.type,
        slots,
      };
    })
  );

  return {
    facilityId,
    date,
    timezone: facility.timezone,
    resources,
  };
}

private generateTimeSlots(
  openTime: string,
  closeTime: string,
  durationMinutes: number,
  bookings: Booking[],
  holds: BookingHold[],
  pricing: ResourcePricing | null,
  isHoliday: boolean,
  dayOfWeek: number,
  userId?: string,
): TimeSlotDto[] {
  const slots: TimeSlotDto[] = [];

  const [openHour, openMin] = openTime.split(':').map(Number);
  const [closeHour, closeMin] = closeTime.split(':').map(Number);

  let currentHour = openHour;
  let currentMin = openMin;

  while (
    currentHour < closeHour ||
    (currentHour === closeHour && currentMin < closeMin)
  ) {
    const startTime = `${String(currentHour).padStart(2, '0')}:${String(currentMin).padStart(2, '0')}`;

    // Calculate end time
    let endHour = currentHour;
    let endMin = currentMin + durationMinutes;
    if (endMin >= 60) {
      endHour += Math.floor(endMin / 60);
      endMin = endMin % 60;
    }
    const endTime = `${String(endHour).padStart(2, '0')}:${String(endMin).padStart(2, '0')}`;

    // Check if slot is booked or held
    const isBooked = bookings.some((b) => {
      const bookingStart = b.startAt.toTimeString().slice(0, 5);
      const bookingEnd = b.endAt.toTimeString().slice(0, 5);
      return startTime >= bookingStart && startTime < bookingEnd;
    });

    const hold = holds.find((h) => {
      const holdStart = h.startAt.toTimeString().slice(0, 5);
      const holdEnd = h.endAt.toTimeString().slice(0, 5);
      return startTime >= holdStart && startTime < holdEnd;
    });

    const isHeldByOther = hold && hold.userId !== userId;
    const isHeldByUser = hold && hold.userId === userId;

    // Calculate price
    const isWeekend = dayOfWeek === 0 || dayOfWeek === 6;
    let price = pricing?.basePrice || 30;
    if (isHoliday && pricing?.holidayPrice) {
      price = pricing.holidayPrice;
    } else if (isWeekend && pricing?.weekendPrice) {
      price = pricing.weekendPrice;
    }

    slots.push({
      startTime,
      endTime,
      available: !isBooked && !isHeldByOther,
      price,
      holdId: isHeldByUser ? hold.id : undefined,
    });

    // Move to next slot
    currentMin += durationMinutes;
    if (currentMin >= 60) {
      currentHour += Math.floor(currentMin / 60);
      currentMin = currentMin % 60;
    }
  }

  return slots;
}
```

#### 2.3 Add Controller Endpoint
**File**: `api/src/organization/facility.controller.ts`

```typescript
@Get(':id/availability')
@Public() // Allow unauthenticated access
@ApiOperation({ summary: 'Get facility availability for a date' })
@ApiResponse({ status: 200, type: AvailabilityResponseDto })
async getAvailability(
  @Param('id') id: string,
  @Query() query: GetAvailabilityDto,
  @Req() req: Request,
): Promise<AvailabilityResponseDto> {
  const userId = req.user?.id; // May be undefined for public access
  return this.bookingService.getAvailability(
    id,
    query.date,
    query.resourceId,
    userId,
  );
}
```

### Deliverables
- [ ] DTO files created
- [ ] Service method implemented
- [ ] Controller endpoint added
- [ ] Swagger documentation visible at /docs
- [ ] Manual test with seed data works

### Verification
```bash
# Test availability endpoint
curl "http://localhost:3001/api/v1/facilities/{facilityId}/availability?date=2026-01-15"
```

---

## Day 3: Public Facility Page (Web)

### Objective
Create the public-facing facility detail page with availability grid.

### Tasks

#### 3.1 Create Facility Page Route
**File**: `web/src/app/facility/[slug]/page.tsx`

```typescript
import { Metadata } from 'next';
import { notFound } from 'next/navigation';
import { FacilityDetail } from '@/components/facility/facility-detail';
import { AvailabilityGrid } from '@/components/booking/availability-grid';
import { CoachList } from '@/components/coach/coach-list';

interface Props {
  params: { slug: string };
}

async function getFacility(slug: string) {
  const res = await fetch(
    `${process.env.NEXT_PUBLIC_API_URL}/api/v1/facilities/slug/${slug}`,
    { cache: 'no-store' }
  );
  if (!res.ok) return null;
  return res.json();
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const facility = await getFacility(params.slug);
  if (!facility) return { title: 'Facility Not Found' };

  return {
    title: `${facility.name} | SlotBase`,
    description: facility.description,
  };
}

export default async function FacilityPage({ params }: Props) {
  const facility = await getFacility(params.slug);

  if (!facility) {
    notFound();
  }

  return (
    <div className="container mx-auto px-4 py-8">
      {/* Hero Section */}
      <FacilityDetail facility={facility} />

      {/* Availability Section */}
      <section className="mt-8">
        <h2 className="text-2xl font-bold mb-4">Book a Session</h2>
        <AvailabilityGrid facilityId={facility.id} />
      </section>

      {/* Coaches Section */}
      <section className="mt-8">
        <h2 className="text-2xl font-bold mb-4">Our Coaches</h2>
        <CoachList facilityId={facility.id} />
      </section>
    </div>
  );
}
```

#### 3.2 Create FacilityDetail Component
**File**: `web/src/components/facility/facility-detail.tsx`

```typescript
'use client';

import { Badge } from '@/components/ui/badge';
import { MapPin, Clock, Star } from 'lucide-react';

interface FacilityDetailProps {
  facility: {
    id: string;
    name: string;
    description: string;
    sportTypes: string[];
    address: {
      street: string;
      city: string;
      state: string;
    };
    operatingHours: Array<{
      dayOfWeek: number;
      openTime: string;
      closeTime: string;
      isClosed: boolean;
    }>;
  };
}

export function FacilityDetail({ facility }: FacilityDetailProps) {
  const todayHours = facility.operatingHours.find(
    (h) => h.dayOfWeek === new Date().getDay()
  );

  return (
    <div className="bg-white rounded-lg shadow-md overflow-hidden">
      {/* Hero Image Placeholder */}
      <div className="h-64 bg-gradient-to-r from-green-600 to-green-800 flex items-center justify-center">
        <span className="text-6xl">🏏</span>
      </div>

      <div className="p-6">
        <div className="flex items-start justify-between">
          <div>
            <h1 className="text-3xl font-bold">{facility.name}</h1>
            <p className="text-gray-600 mt-2">{facility.description}</p>
          </div>
          <div className="flex gap-2">
            {facility.sportTypes.map((sport) => (
              <Badge key={sport} variant="secondary">
                {sport}
              </Badge>
            ))}
          </div>
        </div>

        <div className="mt-4 flex items-center gap-6 text-sm text-gray-600">
          <div className="flex items-center gap-1">
            <MapPin className="h-4 w-4" />
            <span>
              {facility.address.street}, {facility.address.city}, {facility.address.state}
            </span>
          </div>

          {todayHours && !todayHours.isClosed && (
            <div className="flex items-center gap-1">
              <Clock className="h-4 w-4" />
              <span>
                Today: {todayHours.openTime} - {todayHours.closeTime}
              </span>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

#### 3.3 Create AvailabilityGrid Component
**File**: `web/src/components/booking/availability-grid.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { format, addDays } from 'date-fns';
import { Calendar } from '@/components/ui/calendar';
import { Button } from '@/components/ui/button';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { useApiClient } from '@/lib/api/client';
import { cn } from '@/lib/utils';

interface AvailabilityGridProps {
  facilityId: string;
}

export function AvailabilityGrid({ facilityId }: AvailabilityGridProps) {
  const [selectedDate, setSelectedDate] = useState<Date>(new Date());
  const [selectedResource, setSelectedResource] = useState<string>('all');
  const [selectedSlot, setSelectedSlot] = useState<{
    resourceId: string;
    startTime: string;
    endTime: string;
    price: number;
  } | null>(null);

  const apiClient = useApiClient();
  const dateStr = format(selectedDate, 'yyyy-MM-dd');

  const { data: availability, isLoading } = useQuery({
    queryKey: ['availability', facilityId, dateStr, selectedResource],
    queryFn: () =>
      apiClient.get(
        `/facilities/${facilityId}/availability?date=${dateStr}${
          selectedResource !== 'all' ? `&resourceId=${selectedResource}` : ''
        }`
      ),
  });

  const handleSlotClick = (
    resourceId: string,
    slot: { startTime: string; endTime: string; price: number; available: boolean }
  ) => {
    if (!slot.available) return;

    setSelectedSlot({
      resourceId,
      startTime: slot.startTime,
      endTime: slot.endTime,
      price: slot.price,
    });
  };

  return (
    <div className="bg-white rounded-lg shadow-md p-6">
      {/* Controls */}
      <div className="flex flex-wrap gap-4 mb-6">
        <div className="flex-1 min-w-[200px]">
          <label className="block text-sm font-medium mb-2">Select Date</label>
          <Calendar
            mode="single"
            selected={selectedDate}
            onSelect={(date) => date && setSelectedDate(date)}
            disabled={(date) => date < new Date()}
            className="rounded-md border"
          />
        </div>

        <div className="flex-1 min-w-[200px]">
          <label className="block text-sm font-medium mb-2">Filter by Net/Lane</label>
          <Select value={selectedResource} onValueChange={setSelectedResource}>
            <SelectTrigger>
              <SelectValue placeholder="All Resources" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="all">All Nets & Lanes</SelectItem>
              {availability?.resources?.map((r: any) => (
                <SelectItem key={r.resourceId} value={r.resourceId}>
                  {r.resourceName}
                </SelectItem>
              ))}
            </SelectContent>
          </Select>
        </div>
      </div>

      {/* Availability Grid */}
      {isLoading ? (
        <div className="text-center py-8">Loading availability...</div>
      ) : (
        <div className="overflow-x-auto">
          <table className="w-full border-collapse">
            <thead>
              <tr>
                <th className="p-2 border bg-gray-50 text-left">Time</th>
                {availability?.resources?.map((r: any) => (
                  <th key={r.resourceId} className="p-2 border bg-gray-50 text-center min-w-[100px]">
                    {r.resourceName}
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {availability?.resources?.[0]?.slots?.map((slot: any, idx: number) => (
                <tr key={slot.startTime}>
                  <td className="p-2 border font-medium">
                    {slot.startTime}
                  </td>
                  {availability.resources.map((r: any) => {
                    const resourceSlot = r.slots[idx];
                    const isSelected =
                      selectedSlot?.resourceId === r.resourceId &&
                      selectedSlot?.startTime === resourceSlot.startTime;

                    return (
                      <td
                        key={r.resourceId}
                        className={cn(
                          'p-2 border text-center cursor-pointer transition-colors',
                          resourceSlot.available
                            ? isSelected
                              ? 'bg-green-600 text-white'
                              : 'bg-green-100 hover:bg-green-200'
                            : 'bg-gray-300 cursor-not-allowed'
                        )}
                        onClick={() => handleSlotClick(r.resourceId, resourceSlot)}
                      >
                        {resourceSlot.available ? (
                          <span className="font-medium">${resourceSlot.price}</span>
                        ) : (
                          <span className="text-gray-500">Booked</span>
                        )}
                      </td>
                    );
                  })}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}

      {/* Selected Slot Summary */}
      {selectedSlot && (
        <div className="mt-6 p-4 bg-green-50 rounded-lg border border-green-200">
          <h3 className="font-semibold">Selected Session</h3>
          <p className="text-sm text-gray-600 mt-1">
            {format(selectedDate, 'EEEE, MMMM d, yyyy')} at {selectedSlot.startTime} - {selectedSlot.endTime}
          </p>
          <p className="text-lg font-bold mt-2">${selectedSlot.price}</p>
          <Button className="mt-4 w-full" size="lg">
            Continue to Book
          </Button>
        </div>
      )}
    </div>
  );
}
```

### Deliverables
- [ ] Facility page route created
- [ ] FacilityDetail component implemented
- [ ] AvailabilityGrid component implemented
- [ ] Page renders with seed data
- [ ] Slots are clickable and highlight correctly

### Verification
Navigate to `http://localhost:3000/facility/pilot-cricket-academy`

---

## Day 4: Booking Hold Creation Flow

### Objective
When player selects a slot and clicks "Continue to Book", create a 5-minute hold.

### Tasks

#### 4.1 Create useCreateHold Hook
**File**: `web/src/lib/api/hooks/use-bookings.ts`

Add to existing file:

```typescript
export function useCreateHold() {
  const apiClient = useApiClient();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: {
      facilityId: string;
      resourceId: string;
      startAt: string; // ISO datetime
      endAt: string;
    }) => {
      return apiClient.post('/bookings/hold', data);
    },
    onSuccess: (_, variables) => {
      // Invalidate availability for this facility
      queryClient.invalidateQueries({
        queryKey: ['availability', variables.facilityId],
      });
    },
  });
}
```

#### 4.2 Update AvailabilityGrid to Create Hold
**File**: `web/src/components/booking/availability-grid.tsx`

Update the component to use authentication and create holds:

```typescript
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { useAuth, SignInButton } from '@clerk/nextjs';
import { useQuery, useMutation } from '@tanstack/react-query';
import { format, parseISO } from 'date-fns';
import { useCreateHold } from '@/lib/api/hooks/use-bookings';
import { toast } from 'sonner';

// ... existing imports and interface

export function AvailabilityGrid({ facilityId }: AvailabilityGridProps) {
  const router = useRouter();
  const { isSignedIn } = useAuth();
  const createHold = useCreateHold();

  // ... existing state

  const handleBookClick = async () => {
    if (!selectedSlot) return;

    if (!isSignedIn) {
      toast.error('Please sign in to book');
      return;
    }

    try {
      const startAt = `${dateStr}T${selectedSlot.startTime}:00`;
      const endAt = `${dateStr}T${selectedSlot.endTime}:00`;

      const hold = await createHold.mutateAsync({
        facilityId,
        resourceId: selectedSlot.resourceId,
        startAt,
        endAt,
      });

      // Navigate to confirmation page with hold ID
      router.push(`/book/confirm?holdId=${hold.id}`);
    } catch (error: any) {
      toast.error(error.message || 'Failed to create hold');
    }
  };

  return (
    // ... existing JSX with updated button
    {selectedSlot && (
      <div className="mt-6 p-4 bg-green-50 rounded-lg border border-green-200">
        <h3 className="font-semibold">Selected Session</h3>
        <p className="text-sm text-gray-600 mt-1">
          {format(selectedDate, 'EEEE, MMMM d, yyyy')} at {selectedSlot.startTime} - {selectedSlot.endTime}
        </p>
        <p className="text-lg font-bold mt-2">${selectedSlot.price}</p>

        {isSignedIn ? (
          <Button
            className="mt-4 w-full"
            size="lg"
            onClick={handleBookClick}
            disabled={createHold.isPending}
          >
            {createHold.isPending ? 'Creating Hold...' : 'Continue to Book'}
          </Button>
        ) : (
          <SignInButton mode="modal">
            <Button className="mt-4 w-full" size="lg">
              Sign In to Book
            </Button>
          </SignInButton>
        )}
      </div>
    )}
  );
}
```

### Deliverables
- [ ] useCreateHold hook created
- [ ] AvailabilityGrid creates hold on button click
- [ ] Unauthenticated users see sign-in prompt
- [ ] Successful hold redirects to /book/confirm

### Verification
1. Sign in as a player
2. Select an available slot
3. Click "Continue to Book"
4. Verify hold is created (check API logs)
5. Verify redirect to /book/confirm?holdId=xxx

---

## Day 5: Booking Confirmation Page

### Objective
Create the confirmation page where players select booking type, payment method, and confirm.

### Tasks

#### 5.1 Create Confirmation Page
**File**: `web/src/app/book/confirm/page.tsx`

```typescript
'use client';

import { useEffect, useState } from 'react';
import { useSearchParams, useRouter } from 'next/navigation';
import { useAuth } from '@clerk/nextjs';
import { useQuery, useMutation } from '@tanstack/react-query';
import { format, differenceInSeconds } from 'date-fns';
import { Button } from '@/components/ui/button';
import { RadioGroup, RadioGroupItem } from '@/components/ui/radio-group';
import { Label } from '@/components/ui/label';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { Clock, MapPin, Calendar, CreditCard, Banknote } from 'lucide-react';
import { useApiClient } from '@/lib/api/client';
import { toast } from 'sonner';

export default function BookingConfirmPage() {
  const searchParams = useSearchParams();
  const router = useRouter();
  const { isSignedIn } = useAuth();
  const apiClient = useApiClient();
  const holdId = searchParams.get('holdId');

  const [bookingType, setBookingType] = useState('OPEN_PLAY');
  const [paymentMethod, setPaymentMethod] = useState('CASH');
  const [timeRemaining, setTimeRemaining] = useState(300); // 5 minutes

  // Fetch hold details
  const { data: hold, isLoading } = useQuery({
    queryKey: ['hold', holdId],
    queryFn: () => apiClient.get(`/bookings/holds/${holdId}`),
    enabled: !!holdId && isSignedIn,
    refetchInterval: 10000, // Refresh every 10 seconds
  });

  // Countdown timer
  useEffect(() => {
    if (!hold?.expiresAt) return;

    const interval = setInterval(() => {
      const remaining = differenceInSeconds(new Date(hold.expiresAt), new Date());
      setTimeRemaining(Math.max(0, remaining));

      if (remaining <= 0) {
        toast.error('Your hold has expired');
        router.push('/');
      }
    }, 1000);

    return () => clearInterval(interval);
  }, [hold?.expiresAt, router]);

  // Confirm booking mutation
  const confirmBooking = useMutation({
    mutationFn: async () => {
      return apiClient.post('/bookings/confirm', {
        holdId,
        bookingType,
        paymentMethod: paymentMethod === 'CASH' ? 'OFFLINE' : 'ONLINE',
        offlinePaymentType: paymentMethod !== 'ONLINE' ? paymentMethod : undefined,
      });
    },
    onSuccess: (booking) => {
      toast.success('Booking confirmed!');
      router.push(`/dashboard/bookings/${booking.id}`);
    },
    onError: (error: any) => {
      toast.error(error.message || 'Failed to confirm booking');
    },
  });

  const formatTime = (seconds: number) => {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${mins}:${secs.toString().padStart(2, '0')}`;
  };

  if (!holdId) {
    return (
      <div className="container mx-auto px-4 py-8">
        <Alert variant="destructive">
          <AlertDescription>No booking hold found. Please select a time slot first.</AlertDescription>
        </Alert>
      </div>
    );
  }

  if (isLoading) {
    return (
      <div className="container mx-auto px-4 py-8">
        <div className="text-center">Loading booking details...</div>
      </div>
    );
  }

  return (
    <div className="container mx-auto px-4 py-8 max-w-2xl">
      {/* Timer Alert */}
      <Alert className={timeRemaining < 60 ? 'border-red-500 bg-red-50' : 'border-yellow-500 bg-yellow-50'}>
        <Clock className="h-4 w-4" />
        <AlertDescription>
          Hold expires in <strong>{formatTime(timeRemaining)}</strong>
        </AlertDescription>
      </Alert>

      <Card className="mt-6">
        <CardHeader>
          <CardTitle>Confirm Your Booking</CardTitle>
        </CardHeader>
        <CardContent className="space-y-6">
          {/* Session Details */}
          <div className="space-y-3">
            <div className="flex items-center gap-2 text-gray-600">
              <MapPin className="h-4 w-4" />
              <span>{hold?.facility?.name}</span>
            </div>
            <div className="flex items-center gap-2 text-gray-600">
              <span className="text-xl">🏏</span>
              <span>{hold?.resource?.name}</span>
            </div>
            <div className="flex items-center gap-2 text-gray-600">
              <Calendar className="h-4 w-4" />
              <span>
                {hold?.startAt && format(new Date(hold.startAt), 'EEEE, MMMM d, yyyy')}
              </span>
            </div>
            <div className="flex items-center gap-2 text-gray-600">
              <Clock className="h-4 w-4" />
              <span>
                {hold?.startAt && format(new Date(hold.startAt), 'h:mm a')} -
                {hold?.endAt && format(new Date(hold.endAt), 'h:mm a')}
              </span>
            </div>
          </div>

          <hr />

          {/* Booking Type */}
          <div>
            <Label className="text-base font-semibold">Session Type</Label>
            <RadioGroup value={bookingType} onValueChange={setBookingType} className="mt-3">
              <div className="flex items-center space-x-2">
                <RadioGroupItem value="OPEN_PLAY" id="open-play" />
                <Label htmlFor="open-play" className="cursor-pointer">
                  Open Play / Practice Session
                </Label>
              </div>
              <div className="flex items-center space-x-2">
                <RadioGroupItem value="PRIVATE" id="private" />
                <Label htmlFor="private" className="cursor-pointer">
                  Private Booking (exclusive use)
                </Label>
              </div>
              <div className="flex items-center space-x-2">
                <RadioGroupItem value="LESSON" id="lesson" />
                <Label htmlFor="lesson" className="cursor-pointer">
                  Lesson with Coach (select coach below)
                </Label>
              </div>
            </RadioGroup>
          </div>

          <hr />

          {/* Payment Method */}
          <div>
            <Label className="text-base font-semibold">Payment Method</Label>
            <RadioGroup value={paymentMethod} onValueChange={setPaymentMethod} className="mt-3">
              <div className="flex items-center space-x-2">
                <RadioGroupItem value="CASH" id="cash" />
                <Label htmlFor="cash" className="cursor-pointer flex items-center gap-2">
                  <Banknote className="h-4 w-4" />
                  Pay with Cash (at facility)
                </Label>
              </div>
              <div className="flex items-center space-x-2">
                <RadioGroupItem value="CHECK" id="check" />
                <Label htmlFor="check" className="cursor-pointer flex items-center gap-2">
                  <CreditCard className="h-4 w-4" />
                  Pay with Check (at facility)
                </Label>
              </div>
              <div className="flex items-center space-x-2">
                <RadioGroupItem value="VENMO" id="venmo" />
                <Label htmlFor="venmo" className="cursor-pointer">
                  Pay via Venmo
                </Label>
              </div>
              <div className="flex items-center space-x-2">
                <RadioGroupItem value="ZELLE" id="zelle" />
                <Label htmlFor="zelle" className="cursor-pointer">
                  Pay via Zelle
                </Label>
              </div>
            </RadioGroup>
          </div>

          <hr />

          {/* Total */}
          <div className="flex justify-between items-center text-lg">
            <span className="font-semibold">Total</span>
            <span className="font-bold text-2xl">${hold?.price || 30}</span>
          </div>

          {/* Confirm Button */}
          <Button
            className="w-full"
            size="lg"
            onClick={() => confirmBooking.mutate()}
            disabled={confirmBooking.isPending || timeRemaining === 0}
          >
            {confirmBooking.isPending ? 'Confirming...' : 'Confirm Booking'}
          </Button>

          <p className="text-xs text-gray-500 text-center">
            By confirming, you agree to the facility's booking terms and cancellation policy.
          </p>
        </CardContent>
      </Card>
    </div>
  );
}
```

### Deliverables
- [ ] Confirmation page created
- [ ] Hold details fetched and displayed
- [ ] Countdown timer working
- [ ] Booking type selection working
- [ ] Payment method selection working
- [ ] Confirm mutation calls API

### Verification
1. Create a hold from facility page
2. Verify redirect to /book/confirm
3. Verify countdown timer shows
4. Select booking type and payment
5. Click Confirm Booking
6. Verify booking created in database

---

## Day 6: Coach Directory & Selection

### Objective
Display coaches on facility page and allow selection for lesson bookings.

### Tasks

#### 6.1 Create Coaches API Endpoint
**File**: `api/src/organization/facility.controller.ts`

```typescript
@Get(':id/coaches')
@Public()
@ApiOperation({ summary: 'Get facility coaches' })
async getFacilityCoaches(@Param('id') id: string) {
  return this.coachService.getFacilityCoaches(id);
}
```

#### 6.2 Create CoachList Component
**File**: `web/src/components/coach/coach-list.tsx`

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Star, Award } from 'lucide-react';
import { useApiClient } from '@/lib/api/client';

interface CoachListProps {
  facilityId: string;
  onSelectCoach?: (coachId: string) => void;
  selectedCoachId?: string;
}

export function CoachList({ facilityId, onSelectCoach, selectedCoachId }: CoachListProps) {
  const apiClient = useApiClient();

  const { data: coaches, isLoading } = useQuery({
    queryKey: ['facility-coaches', facilityId],
    queryFn: () => apiClient.get(`/facilities/${facilityId}/coaches`),
  });

  if (isLoading) {
    return <div>Loading coaches...</div>;
  }

  if (!coaches?.length) {
    return (
      <p className="text-gray-500">No coaches available at this facility.</p>
    );
  }

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      {coaches.map((coach: any) => (
        <Card
          key={coach.id}
          className={`cursor-pointer transition-all ${
            selectedCoachId === coach.id
              ? 'ring-2 ring-green-500'
              : 'hover:shadow-md'
          }`}
          onClick={() => onSelectCoach?.(coach.id)}
        >
          <CardContent className="p-4">
            <div className="flex items-start gap-4">
              {/* Avatar */}
              <div className="w-16 h-16 rounded-full bg-gray-200 flex items-center justify-center text-2xl">
                {coach.user?.avatarUrl ? (
                  <img
                    src={coach.user.avatarUrl}
                    alt={coach.user.name}
                    className="w-full h-full rounded-full object-cover"
                  />
                ) : (
                  '👤'
                )}
              </div>

              {/* Info */}
              <div className="flex-1">
                <h3 className="font-semibold">{coach.user?.name || 'Coach'}</h3>

                <div className="flex items-center gap-1 text-sm text-yellow-600 mt-1">
                  <Star className="h-4 w-4 fill-current" />
                  <span>{coach.rating || '4.5'}</span>
                </div>

                {coach.specializations?.length > 0 && (
                  <div className="flex flex-wrap gap-1 mt-2">
                    {coach.specializations.slice(0, 3).map((spec: string) => (
                      <Badge key={spec} variant="outline" className="text-xs">
                        {spec}
                      </Badge>
                    ))}
                  </div>
                )}

                <div className="mt-3 flex items-center justify-between">
                  <span className="text-lg font-bold text-green-600">
                    ${coach.hourlyRate || 50}/hr
                  </span>

                  {coach.certifications?.length > 0 && (
                    <Award className="h-4 w-4 text-blue-500" title="Certified Coach" />
                  )}
                </div>
              </div>
            </div>
          </CardContent>
        </Card>
      ))}
    </div>
  );
}
```

#### 6.3 Update Confirmation Page for Coach Selection
Add coach selection when booking type is LESSON (in Day 5's confirmation page).

### Deliverables
- [ ] Coaches API endpoint created
- [ ] CoachList component implemented
- [ ] Coaches display on facility page
- [ ] Coach selection works in confirmation flow

---

## Day 7: Email Notifications

### Objective
Send booking confirmation emails via Resend.

### Tasks

#### 7.1 Create Email Templates
**File**: `api/src/email/templates/booking-confirmed.ts`

```typescript
export function bookingConfirmedTemplate(data: {
  playerName: string;
  facilityName: string;
  resourceName: string;
  date: string;
  time: string;
  bookingType: string;
  paymentMethod: string;
  amount: number;
  bookingId: string;
}) {
  return `
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
    .container { max-width: 600px; margin: 0 auto; padding: 20px; }
    .header { background: #16a34a; color: white; padding: 20px; text-align: center; border-radius: 8px 8px 0 0; }
    .content { background: #f9fafb; padding: 20px; border-radius: 0 0 8px 8px; }
    .detail-row { display: flex; justify-content: space-between; padding: 10px 0; border-bottom: 1px solid #e5e7eb; }
    .label { color: #6b7280; }
    .value { font-weight: bold; }
    .total { font-size: 24px; color: #16a34a; text-align: center; margin: 20px 0; }
    .footer { text-align: center; margin-top: 20px; color: #9ca3af; font-size: 12px; }
    .btn { display: inline-block; background: #16a34a; color: white; padding: 12px 24px; text-decoration: none; border-radius: 6px; margin: 20px 0; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>🏏 Booking Confirmed!</h1>
    </div>
    <div class="content">
      <p>Hi ${data.playerName},</p>
      <p>Your booking has been confirmed. Here are the details:</p>

      <div class="detail-row">
        <span class="label">Facility</span>
        <span class="value">${data.facilityName}</span>
      </div>
      <div class="detail-row">
        <span class="label">Net/Lane</span>
        <span class="value">${data.resourceName}</span>
      </div>
      <div class="detail-row">
        <span class="label">Date</span>
        <span class="value">${data.date}</span>
      </div>
      <div class="detail-row">
        <span class="label">Time</span>
        <span class="value">${data.time}</span>
      </div>
      <div class="detail-row">
        <span class="label">Session Type</span>
        <span class="value">${data.bookingType}</span>
      </div>
      <div class="detail-row">
        <span class="label">Payment</span>
        <span class="value">${data.paymentMethod}</span>
      </div>

      <div class="total">
        Total: $${data.amount}
      </div>

      <p style="text-align: center;">
        <a href="${process.env.WEB_URL}/dashboard/bookings/${data.bookingId}" class="btn">
          View Booking
        </a>
      </p>

      <p><strong>What to bring:</strong></p>
      <ul>
        <li>Cricket bat and gloves</li>
        <li>Comfortable athletic wear</li>
        <li>Water bottle</li>
      </ul>

      <p>See you at the nets!</p>
    </div>
    <div class="footer">
      <p>SlotBase - Book Your Game</p>
      <p>Questions? Reply to this email.</p>
    </div>
  </div>
</body>
</html>
  `;
}
```

#### 7.2 Send Email on Booking Confirmation
**File**: `api/src/booking/booking.service.ts`

In the `confirmBooking` method, add:

```typescript
// After booking is created successfully
await this.emailService.sendEmail({
  to: user.email,
  subject: `Booking Confirmed - ${facility.name}`,
  html: bookingConfirmedTemplate({
    playerName: user.name || user.email,
    facilityName: facility.name,
    resourceName: resource.name,
    date: format(booking.startAt, 'EEEE, MMMM d, yyyy'),
    time: `${format(booking.startAt, 'h:mm a')} - ${format(booking.endAt, 'h:mm a')}`,
    bookingType: booking.type,
    paymentMethod: booking.paymentMethod === 'OFFLINE' ? 'Pay at Facility' : 'Paid Online',
    amount: booking.totalAmount,
    bookingId: booking.id,
  }),
});
```

### Deliverables
- [ ] Email template created
- [ ] Email sent on booking confirmation
- [ ] Email received in inbox (test with real email)

---

## Day 8: Player Dashboard

### Objective
Create the player dashboard showing upcoming and past bookings.

### Tasks

#### 8.1 Create Player Dashboard Page
**File**: `web/src/app/(dashboard)/player/page.tsx`

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { format } from 'date-fns';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Calendar, Clock, MapPin, ChevronRight } from 'lucide-react';
import { useApiClient } from '@/lib/api/client';
import Link from 'next/link';

export default function PlayerDashboard() {
  const apiClient = useApiClient();

  const { data: upcomingBookings, isLoading: loadingUpcoming } = useQuery({
    queryKey: ['my-bookings', 'upcoming'],
    queryFn: () => apiClient.get('/bookings/upcoming'),
  });

  const { data: pastBookings, isLoading: loadingPast } = useQuery({
    queryKey: ['my-bookings', 'past'],
    queryFn: () => apiClient.get('/bookings/past?limit=5'),
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">My Bookings</h1>

      {/* Quick Actions */}
      <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
        <Link href="/">
          <Card className="hover:shadow-md transition-shadow cursor-pointer">
            <CardContent className="p-6 flex items-center gap-4">
              <div className="p-3 bg-green-100 rounded-full">
                <Calendar className="h-6 w-6 text-green-600" />
              </div>
              <div>
                <h3 className="font-semibold">Book a Session</h3>
                <p className="text-sm text-gray-500">Find available slots</p>
              </div>
              <ChevronRight className="ml-auto h-5 w-5 text-gray-400" />
            </CardContent>
          </Card>
        </Link>
      </div>

      {/* Upcoming Bookings */}
      <section className="mb-8">
        <h2 className="text-xl font-semibold mb-4">Upcoming Sessions</h2>

        {loadingUpcoming ? (
          <div>Loading...</div>
        ) : upcomingBookings?.length === 0 ? (
          <Card>
            <CardContent className="p-6 text-center text-gray-500">
              No upcoming bookings. Book a session to get started!
            </CardContent>
          </Card>
        ) : (
          <div className="space-y-4">
            {upcomingBookings?.map((booking: any) => (
              <Card key={booking.id}>
                <CardContent className="p-4">
                  <div className="flex items-start justify-between">
                    <div className="space-y-2">
                      <div className="flex items-center gap-2">
                        <span className="text-2xl">🏏</span>
                        <div>
                          <h3 className="font-semibold">{booking.resource?.name}</h3>
                          <p className="text-sm text-gray-500">{booking.facility?.name}</p>
                        </div>
                      </div>

                      <div className="flex items-center gap-4 text-sm text-gray-600">
                        <div className="flex items-center gap-1">
                          <Calendar className="h-4 w-4" />
                          {format(new Date(booking.startAt), 'EEE, MMM d')}
                        </div>
                        <div className="flex items-center gap-1">
                          <Clock className="h-4 w-4" />
                          {format(new Date(booking.startAt), 'h:mm a')}
                        </div>
                      </div>
                    </div>

                    <div className="text-right">
                      <Badge variant={booking.status === 'CONFIRMED' ? 'default' : 'secondary'}>
                        {booking.status}
                      </Badge>
                      <p className="text-lg font-bold mt-2">${booking.totalAmount}</p>
                    </div>
                  </div>

                  <div className="mt-4 flex gap-2">
                    <Button variant="outline" size="sm" asChild>
                      <Link href={`/dashboard/bookings/${booking.id}`}>View Details</Link>
                    </Button>
                    <Button variant="outline" size="sm" className="text-red-600">
                      Cancel
                    </Button>
                  </div>
                </CardContent>
              </Card>
            ))}
          </div>
        )}
      </section>

      {/* Past Bookings */}
      <section>
        <div className="flex items-center justify-between mb-4">
          <h2 className="text-xl font-semibold">Past Sessions</h2>
          <Link href="/dashboard/bookings/history" className="text-sm text-green-600 hover:underline">
            View All
          </Link>
        </div>

        {loadingPast ? (
          <div>Loading...</div>
        ) : pastBookings?.length === 0 ? (
          <p className="text-gray-500">No past bookings yet.</p>
        ) : (
          <div className="space-y-2">
            {pastBookings?.map((booking: any) => (
              <div
                key={booking.id}
                className="flex items-center justify-between p-3 bg-gray-50 rounded-lg"
              >
                <div className="flex items-center gap-3">
                  <span>🏏</span>
                  <div>
                    <p className="font-medium">{booking.resource?.name}</p>
                    <p className="text-sm text-gray-500">
                      {format(new Date(booking.startAt), 'MMM d, yyyy')}
                    </p>
                  </div>
                </div>
                <Badge variant="outline">{booking.status}</Badge>
              </div>
            ))}
          </div>
        )}
      </section>
    </div>
  );
}
```

### Deliverables
- [ ] Player dashboard page created
- [ ] Upcoming bookings displayed
- [ ] Past bookings displayed
- [ ] Navigation to booking details works

---

## Day 9: Tier Selection in Onboarding

### Objective
Add subscription tier selection to the facility onboarding wizard.

### Tasks

#### 9.1 Create Tier Selection Step
**File**: `web/src/app/onboard/steps/tier-selection.tsx`

```typescript
'use client';

import { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Check, X } from 'lucide-react';

interface TierSelectionProps {
  selectedTier: string;
  onSelect: (tier: string) => void;
  onContinue: () => void;
}

const TIERS = [
  {
    id: 'STARTER',
    name: 'Starter',
    price: 'Free',
    description: 'Perfect for getting started',
    features: [
      { name: '3 resources (nets/lanes)', included: true },
      { name: '20 active players', included: true },
      { name: '1 staff account', included: true },
      { name: '1 coach', included: true },
      { name: 'Offline payments (cash)', included: true },
      { name: 'Email notifications', included: true },
      { name: 'Online payments', included: false },
      { name: 'Recurring bookings', included: false },
      { name: 'Analytics', included: false },
    ],
    recommended: false,
  },
  {
    id: 'GROWTH',
    name: 'Growth',
    price: '$49/mo',
    description: 'For growing facilities',
    features: [
      { name: '10 resources', included: true },
      { name: '200 active players', included: true },
      { name: '3 staff accounts', included: true },
      { name: '5 coaches', included: true },
      { name: 'All offline payments', included: true },
      { name: 'Online payments (Stripe)', included: true },
      { name: 'Weekly recurring bookings', included: true },
      { name: 'Basic analytics', included: true },
      { name: 'Email support', included: true },
    ],
    recommended: true,
  },
  {
    id: 'PRO',
    name: 'Pro',
    price: '$149/mo',
    description: 'For established academies',
    features: [
      { name: '50 resources', included: true },
      { name: '2,000 active players', included: true },
      { name: '10 staff accounts', included: true },
      { name: '20 coaches', included: true },
      { name: 'Full recurring bookings', included: true },
      { name: 'Advanced analytics', included: true },
      { name: 'Priority support', included: true },
      { name: 'Custom branding', included: true },
      { name: 'API access', included: true },
    ],
    recommended: false,
  },
];

export function TierSelection({ selectedTier, onSelect, onContinue }: TierSelectionProps) {
  return (
    <div className="space-y-6">
      <div className="text-center">
        <h2 className="text-2xl font-bold">Choose Your Plan</h2>
        <p className="text-gray-600 mt-2">
          Start free and upgrade as you grow. No credit card required.
        </p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {TIERS.map((tier) => (
          <Card
            key={tier.id}
            className={`cursor-pointer transition-all ${
              selectedTier === tier.id
                ? 'ring-2 ring-green-500 shadow-lg'
                : 'hover:shadow-md'
            } ${tier.recommended ? 'border-green-500' : ''}`}
            onClick={() => onSelect(tier.id)}
          >
            {tier.recommended && (
              <div className="bg-green-500 text-white text-center py-1 text-sm font-medium">
                Recommended
              </div>
            )}
            <CardHeader>
              <CardTitle className="flex items-center justify-between">
                <span>{tier.name}</span>
                {selectedTier === tier.id && (
                  <Check className="h-5 w-5 text-green-500" />
                )}
              </CardTitle>
              <div className="mt-2">
                <span className="text-3xl font-bold">{tier.price}</span>
              </div>
              <p className="text-sm text-gray-500">{tier.description}</p>
            </CardHeader>
            <CardContent>
              <ul className="space-y-2">
                {tier.features.map((feature) => (
                  <li key={feature.name} className="flex items-center gap-2 text-sm">
                    {feature.included ? (
                      <Check className="h-4 w-4 text-green-500" />
                    ) : (
                      <X className="h-4 w-4 text-gray-300" />
                    )}
                    <span className={feature.included ? '' : 'text-gray-400'}>
                      {feature.name}
                    </span>
                  </li>
                ))}
              </ul>
            </CardContent>
          </Card>
        ))}
      </div>

      <div className="text-center">
        <Button size="lg" onClick={onContinue} disabled={!selectedTier}>
          Continue with {TIERS.find((t) => t.id === selectedTier)?.name || 'selected'} plan
        </Button>
      </div>
    </div>
  );
}
```

### Deliverables
- [ ] Tier selection component created
- [ ] Integrated into onboarding wizard
- [ ] Selected tier saved to facility subscription

---

## Day 10: Tier Limit Enforcement & Testing

### Objective
Enforce subscription limits and perform end-to-end testing.

### Tasks

#### 10.1 Create Subscription Guard Service
**File**: `api/src/subscription/subscription-guard.service.ts`

```typescript
import { Injectable, ForbiddenException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class SubscriptionGuardService {
  constructor(private prisma: PrismaService) {}

  async checkLimit(
    facilityId: string,
    limitType: 'players' | 'resources' | 'staff' | 'coaches',
  ): Promise<void> {
    const facility = await this.prisma.facility.findUnique({
      where: { id: facilityId },
      include: { subscription: true },
    });

    if (!facility?.subscription) {
      throw new ForbiddenException('No subscription found');
    }

    const tier = facility.subscription.tier;
    const featureKey = `max_${limitType}`;

    const tierFeature = await this.prisma.tierFeature.findUnique({
      where: {
        tier_featureKey: { tier, featureKey },
      },
    });

    if (!tierFeature || tierFeature.limitValue === null) {
      return; // No limit (unlimited)
    }

    // Count current usage
    let currentCount = 0;
    switch (limitType) {
      case 'players':
        currentCount = await this.prisma.userRole.count({
          where: {
            scopeId: facilityId,
            role: 'PLAYER',
          },
        });
        break;
      case 'resources':
        currentCount = await this.prisma.resource.count({
          where: { facilityId },
        });
        break;
      case 'staff':
        currentCount = await this.prisma.userRole.count({
          where: {
            scopeId: facilityId,
            role: { in: ['FACILITY_ADMIN', 'FACILITY_STAFF'] },
          },
        });
        break;
      case 'coaches':
        currentCount = await this.prisma.coachFacilityAffiliation.count({
          where: { facilityId },
        });
        break;
    }

    if (currentCount >= tierFeature.limitValue) {
      throw new ForbiddenException(
        `${tier} plan limit reached: maximum ${tierFeature.limitValue} ${limitType}. Please upgrade your plan.`
      );
    }
  }

  async checkFeature(facilityId: string, featureKey: string): Promise<boolean> {
    const facility = await this.prisma.facility.findUnique({
      where: { id: facilityId },
      include: { subscription: true },
    });

    if (!facility?.subscription) {
      return false;
    }

    const tierFeature = await this.prisma.tierFeature.findUnique({
      where: {
        tier_featureKey: {
          tier: facility.subscription.tier,
          featureKey,
        },
      },
    });

    return tierFeature?.enabled ?? false;
  }
}
```

#### 10.2 Apply Guards to Critical Endpoints
Add checks in:
- `POST /facilities/:id/resources` - Check resources limit
- `POST /bookings/confirm` - Check players limit when new player books
- `POST /admin/invitations` - Check staff/coaches limit

#### 10.3 End-to-End Testing Checklist

Run through these scenarios manually:

**Player Booking Flow**
- [ ] Visit `/facility/pilot-cricket-academy`
- [ ] See facility details and availability grid
- [ ] Select an available slot
- [ ] Sign in (if not already)
- [ ] Verify hold is created with countdown
- [ ] Select booking type (Open Play)
- [ ] Select payment method (Cash)
- [ ] Confirm booking
- [ ] Verify email received
- [ ] See booking in player dashboard

**Coach Lesson Flow**
- [ ] Visit facility page
- [ ] See coaches listed
- [ ] Select a time slot
- [ ] Choose "Lesson" booking type
- [ ] Select a coach
- [ ] Confirm booking
- [ ] Verify coach sees lesson in their dashboard

**Tier Limits**
- [ ] Try to add 4th resource on STARTER plan (should fail)
- [ ] Try to add 21st player on STARTER plan (should fail)
- [ ] Verify error message mentions upgrade

### Deliverables
- [ ] Subscription guard service created
- [ ] Guards applied to endpoints
- [ ] All manual test cases pass
- [ ] No console errors in browser
- [ ] API returns proper error messages

---

## API Reference for Codex

### Existing Endpoints (Already Implemented)
```
POST   /api/v1/bookings/hold           - Create 5-minute hold
GET    /api/v1/bookings/holds          - List user's active holds
DELETE /api/v1/bookings/holds/:id      - Release hold
POST   /api/v1/bookings/confirm        - Convert hold to booking
GET    /api/v1/bookings                - List bookings
GET    /api/v1/bookings/upcoming       - User's upcoming bookings
GET    /api/v1/bookings/past           - User's past bookings
GET    /api/v1/bookings/:id            - Get booking details
DELETE /api/v1/bookings/:id            - Cancel booking
POST   /api/v1/bookings/:id/check-in   - Check in to booking
```

### New Endpoints (Sprint 16)
```
GET    /api/v1/facilities/:id/availability  - Get availability grid
GET    /api/v1/facilities/:id/coaches       - Get facility coaches
GET    /api/v1/coaches/:id                  - Get coach profile
GET    /api/v1/coaches/:id/availability     - Get coach availability
```

---

## File Structure Summary

### API Changes
```
api/
├── prisma/
│   ├── schema.prisma          # Add SubscriptionUsage, TierFeature
│   └── seed.ts                # Update for cricket, add tier features
└── src/
    ├── booking/
    │   ├── dto/availability.dto.ts    # NEW
    │   └── booking.service.ts         # Add getAvailability
    ├── organization/
    │   └── facility.controller.ts     # Add availability, coaches endpoints
    ├── subscription/
    │   └── subscription-guard.service.ts  # NEW
    └── email/
        └── templates/
            └── booking-confirmed.ts   # NEW
```

### Web Changes
```
web/src/
├── app/
│   ├── facility/
│   │   └── [slug]/
│   │       └── page.tsx       # NEW - Public facility page
│   ├── book/
│   │   └── confirm/
│   │       └── page.tsx       # NEW - Booking confirmation
│   ├── (dashboard)/
│   │   └── player/
│   │       └── page.tsx       # NEW - Player dashboard
│   └── onboard/
│       └── steps/
│           └── tier-selection.tsx  # NEW
├── components/
│   ├── facility/
│   │   └── facility-detail.tsx    # NEW
│   ├── booking/
│   │   └── availability-grid.tsx  # NEW
│   └── coach/
│       └── coach-list.tsx         # NEW
└── lib/
    └── api/
        └── hooks/
            └── use-bookings.ts    # Add useCreateHold
```

---

## Definition of Done

Sprint 16 is complete when:

1. **Database**: Schema migrated, seed data is cricket-focused
2. **API**: Availability endpoint returns correct data
3. **Web**: Player can complete full booking flow
4. **Email**: Confirmation emails are sent and received
5. **Tiers**: Free tier limits are enforced
6. **Testing**: All manual test cases pass

---

## Notes for Codex

1. **Start each day** by reading this document and picking up from where you left off
2. **Commit frequently** with descriptive messages
3. **Test as you go** - don't wait until Day 10
4. **Ask for clarification** if any requirement is unclear
5. **Update this document** if you find issues or make design changes

Good luck! 🏏
