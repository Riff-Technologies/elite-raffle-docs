## Models

These models are proposals and do not necessarily represent the definitive and final models. Changes should be suggested where required.

All currency values are in the lowest denomination, e.g. for USD they are in cents.

### Raffle event

```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Summer Festival Raffle",
  "description": "Annual summer fundraiser raffle",
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
  "venueId": "venue-987f6543-a21c-43d5-b678-123456789abc",
  "eventIdentifier": "RAFFLE-2025-001",
  "state": "active",
  "configuration": {
    "salesStartTime": "2025-06-28T09:00:00Z",
    "salesEndTime": "2025-06-30T18:00:00Z",
    "drawTime": "2025-06-30T19:00:00Z",
    "jackpotSeedAmount": 100000, // the guaranteed minimum jackpot
    "jackpotStartingAmount": 0, // any starting cash amount for the jackpot
    "revenueCalculationMethod": "gross_revenue",
    "maxTickets": 200000,
    "ticketPackageIds": [
      "pkg-123e4567-e89b-12d3-a456-426614174001",
      "pkg-123e4567-e89b-12d3-a456-426614174002",
      "pkg-123e4567-e89b-12d3-a456-426614174003"
    ],
    "prizeIds": [
      "prize-123e4567-e89b-12d3-a456-426614174001",
      "prize-123e4567-e89b-12d3-a456-426614174002"
    ]
  },
  "ticketAllocation": {
    "totalSold": 17234,  // NEW: Running total of tickets sold
    "ranges": [
      {
        "id": "range-123e4567-e89b-12d3-a456-426614174001",
        "type": "rsu",
        "start": 1,
        "end": 50000,
        "defaultAllocationSize": 100,
        "allocated": 2000,  // Running total of allocated ranges
        "sold": 1875,       // NEW: Running total of tickets actually sold
        "available": 48000
      },
      {
        "id": "range-123e4567-e89b-12d3-a456-426614174002",
        "type": "online",
        "start": 50001,
        "end": 180000,
        "defaultAllocationSize": 1,
        "allocated": 0,     // Online doesn't pre-allocate
        "sold": 15234,      // NEW: Running total of tickets sold
        "available": 114766
      },
      {
        "id": "range-123e4567-e89b-12d3-a456-426614174003",
        "type": "reserve",
        "start": 180001,
        "end": 200000,
        "defaultAllocationSize": 1000,
        "allocated": 5000,
        "sold": 125,        // NEW: Running total of tickets sold
        "available": 14875
      }
    ]
  },
  "reconciliationConfirmedAt": null,
  "reconciliationConfirmedBy": null,
  "createdAt": "2025-06-27T14:30:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-28T09:15:00Z",
  "updatedBy": "admin-user-id"
}
```

### Ticket

```json
{
  "id": "ticket-789a0123-c45d-67e8-f901-234567890abc",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "drawNumber": 1001,
  "state": "sold",  // Now only: "sold", "voided", "winner", "claimed"
  "orderId": "order-321b4567-d89e-12f3-a456-426614174000",  // Always has an order
  "soldAt": "2025-06-28T14:30:00Z",
  "soldBy": "operator-user-id",  // null for online sales
  "voidedAt": null,
  "voidedBy": null,
  "voidedReason": null,
  "createdAt": "2025-06-28T14:30:00Z",  // Created when sold
  "updatedAt": "2025-06-28T14:30:00Z"
}
```

### Prize template (reusable)

```json
{
  "id": "prize-template-123e4567-e89b-12d3-a456-426614174001",
  "name": "Grand Prize",
  "description": "iPad Pro + Apple Watch",
  "type": "fixed", // fixed or percentage
  "value": 1500.0, // null for percentage prizes
  "percentage": null, // null for fixed prizes
  "currency": "USD",
  "createdAt": "2025-06-27T13:00:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-27T13:00:00Z",
  "updatedBy": "admin-user-id"
}
```

### Prize

```json
{
  "id": "prize-123e4567-e89b-12d3-a456-426614174001",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Raffle Winner",
  "description": "50/50 Raffle Winner",
  "type": "percentage", // fixed or percentage
  "value": null, // null for percentage prizes
  "percentage": 40, // null for fixed prizes
  "currency": "USD",
  "position": 1,
  "winnerCount": 1,
  "createdAt": "2025-06-27T13:00:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-27T13:00:00Z",
  "updatedBy": "admin-user-id"
}
```

### Ticket Package template

```json
{
  "id": "pkg-template-123e4567-e89b-12d3-a456-426614174002",
  "name": "Value Pack",
  "description": "5 tickets - Best value!",
  "ticketCount": 5,
  "price": 2000,
  "currency": "USD",
  "displayOrder": 2,
  "createdAt": "2025-06-27T14:05:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-28T10:30:00Z",
  "updatedBy": "admin-user-id"
}
```

### Ticket Package

```json
{
  "id": "pkg-123e4567-e89b-12d3-a456-426614174002",
  "name": "Value Pack",
  "description": "5 tickets - Best value!",
  "ticketCount": 5,
  "price": 2000,
  "currency": "USD",
  "isActive": true,
  "displayOrder": 2,
  "activeFrom": "2025-06-28T09:00:00Z", // null if the raffle does not have dynamic ticket pricing
  "activeTo": "2025-06-28T15:00:00Z", // null if the raffle does not have dynamic ticket pricing
  "perkIds": ["perk-123e4567-e89b-12d3-a456-426614174001"],
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "createdAt": "2025-06-27T14:05:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-28T10:30:00Z",
  "updatedBy": "admin-user-id"
}
```

### Perk

```json
{
  "id": "perk-123e4567-e89b-12d3-a456-426614174001",
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
  "name": "VIP Parking",
  "description": "Reserved parking spot near the main entrance",
  "type": "benefit", // "benefit", "physical_item", "digital_item", "experience"
  "category": "parking",
  "requiresInventory": true,
  "inventory": {
    "total": 50,
    "allocated": 0,
    "claimed": 0,
    "available": 50
  },
  "isActive": true,
  "metadata": {
    "location": "Lot A, Spaces 1-50",
    "validityPeriod": "event_day" // "event_day", "week", "month", "custom"
  },
  "createdAt": "2025-01-01T00:00:00Z"
}
```

### Perk claim (tracks who claimed what)

```json
{
  "id": "perk-claim-123e4567-e89b-12d3-a456-426614174001",
  "perkInstanceId": "perk-123e4567-e89b-12d3-a456-426614174001",
  "orderId": "order-321b4567-d89e-12f3-a456-426614174000",
  "claimedAt": "2025-06-28T15:00:00Z",
  "claimedBy": "user-123", // Could be customer or admin
  "status": "claimed", // "pending", "claimed", "expired"
  "claimCode": "PARK-A1-2025", // For physical redemption
  "metadata": {
    "assignedSpace": "A-15"
  }
}
```

### RSU

```json
{
  "id": "rsu-456f7890-b12c-34d5-e678-901234567abc",
  "name": "RSU-001",
  "description": "Main entrance sales terminal",
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
  "venueId": null,
  "deviceIdentifier": "MAC:00:1B:44:11:3A:B7",
  "state": "active",
  "configuration": {
    "maxOfflineTickets": 100,
    "syncIntervalMinutes": 5,
    "printerSettings": {
      "paperSize": "58mm",
      "printSpeed": "normal"
    }
  },
  "activeAllocations": [  // Renamed from activeRanges
    {
      "allocationId": "alloc-123e4567-e89b-12d3-a456-426614174000",
      "raffleEventId": "123e4567-e89b-12d3-a456-426614174000",
      "startNumber": 1001,
      "endNumber": 1100,
      "nextDrawNumber": 1028,  // NEW: Next number to assign
      "ticketsSold": 27,
      "ticketsVoided": 2,
      "ticketsAvailable": 71,
      "validationNumberBlock": {  // NEW: For offline sales
        "blockId": "vnb-123e4567-e89b-12d3-a456-426614174000",
        "blockStart": "RSU001-2025-0001",
        "blockEnd": "RSU001-2025-0100",
        "nextSequence": 28,
        "numbersUsed": 27,
        "numbersAvailable": 73
      }
    }
  ],
  "validation": {
    "lastSync": "2025-06-29T15:45:00Z",
    "lastValidation": "2025-06-29T15:45:00Z",
    "status": "valid"
  },
  "isActive": true,
  "currentOperatorId": "operator-123e4567-e89b-12d3-a456-426614174000",
  "createdAt": "2025-06-20T08:00:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-29T15:45:00Z",
  "updatedBy": "system"
}
```

### RSU Allocation

```json
{
  "id": "alloc-123e4567-e89b-12d3-a456-426614174000",
  "raffleEventId": "123e4567-e89b-12d3-a456-426614174000",
  "rangeType": "rsu",
  "entityId": "rsu-456f7890-b12c-34d5-e678-901234567abc",
  "entityType": "rsu",
  "startNumber": 1001,
  "endNumber": 1100,
  "nextDrawNumber": 1028,  // NEW: Tracks next number to assign
  "totalTickets": 100,
  "state": "active",
  "ticketsSold": 27,
  "ticketsVoided": 2,
  "ticketsAvailable": 71,
  "validationNumberBlock": {  // NEW: Pre-allocated validation numbers
    "blockId": "vnb-123e4567-e89b-12d3-a456-426614174000",
    "blockStart": "RSU001-2025-0001",
    "blockEnd": "RSU001-2025-0100",
    "blockSize": 100,
    "nextSequence": 28,
    "numbersUsed": 27,
    "expiresAt": null
  },
  "allocatedAt": "2025-06-28T08:00:00Z",
  "allocatedBy": "admin-user-id",
  "exhaustedAt": null,
  "metadata": {
    "rsuName": "RSU-001",
    "location": "Main Entrance"
  }
}
```

### Order

```json
{
  "id": "order-321b4567-d89e-12f3-a456-426614174000",
  "validationNumber": "VAL-9F8E7D6C5B4A",  // Or "RSU001-2025-0001" for RSU sales
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "packageId": "pkg-123e4567-e89b-12d3-a456-426614174002",
  "totalAmount": 2000,
  "currency": "USD",
  "paymentStatus": "completed",
  "paymentReference": "stripe_pi_1234567890",
  "ticketIds": [  // Created after payment completion
    "ticket-789a0123-c45d-67e8-f901-234567890abc",
    "ticket-789a0123-c45d-67e8-f901-234567890abd",
    "ticket-789a0123-c45d-67e8-f901-234567890abe",
    "ticket-789a0123-c45d-67e8-f901-234567890abf",
    "ticket-789a0123-c45d-67e8-f901-234567890abg"
  ],
  "customerInfo": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+1-555-987-6543"
  },
  "source": "online",
  "rsuId": null,
  "rsuAllocationId": null,  // NEW: Links to RSU allocation
  "operatorId": null,
  "isOfflineSale": false,  // NEW: Track offline sales
  "offlineSyncStatus": null,  // NEW: "pending", "synced", "failed"
  "offlineCreatedAt": null,  // NEW: When created on RSU
  "createdAt": "2025-06-28T14:25:00Z",
  "createdBy": "operator-user-id",
  "updatedAt": "2025-06-28T14:30:00Z"
}
```

### RSU Offline Sale Record (Local SQLite on RSU)

```json
{
  "localId": "local-321b4567-d89e-12f3-a456-426614174000",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "allocationId": "alloc-123e4567-e89b-12d3-a456-426614174000",
  "validationNumber": "RSU001-2025-0001",  // Pre-allocated
  "drawNumbers": [1001, 1002, 1003, 1004, 1005],  // Pre-allocated
  "packageId": "pkg-123e4567-e89b-12d3-a456-426614174002",
  "ticketCount": 5,
  "totalAmount": 2000,
  "currency": "USD",
  "customerInfo": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+1-555-987-6543"
  },
  "operatorId": "operator-123e4567-e89b-12d3-a456-426614174000",
  "createdAt": "2025-06-28T14:25:00Z",
  "syncStatus": "pending",  // "pending", "synced", "failed"
  "syncAttempts": 0,
  "lastSyncAttempt": null
}
```

### Organization

```json
{
  "id": "org-123e4567-e89b-12d3-a456-426614174000",
  "name": "Community Center Inc",
  "description": "Local community organization",
  "licenseNumber": "LIC-2025-001",
  "primaryCurrency": "USD",
  "contactInfo": {
    "email": "admin@communitycenter.org",
    "phone": "+1-555-123-4567",
    "addressId": "addr-123e4567-e89b-12d3-a456-426614174000"
  },
  "settings": {
    "defaultRevenueCalculationMethod": "gross_revenue",
    "taxSettings": {
      "taxRate": 8.25,
      "taxIncluded": false
    },
    "rsuSettings": {  // NEW: RSU-specific settings
      "defaultValidationNumberBlockSize": 100,
      "validationNumberPrefix": "RSU",
      "offlineSyncTimeoutMinutes": 30
    }
  },
  "defaultVenueId": "venue-987f6543-a21c-43d5-b678-123456789abc",
  "venues": ["venue-987f6543-a21c-43d5-b678-123456789abc"],
  "operatingFee": {
    "type": "percentage",
    "value": null,
    "percent": 4.5
  },
  "isActive": true,
  "createdAt": "2025-01-15T10:00:00Z",
  "createdBy": "system",
  "updatedAt": "2025-06-20T14:30:00Z",
  "updatedBy": "admin-user-id"
}
```

### Venue

```json
{
  "id": "venue-987f6543-a21c-43d5-b678-123456789abc",
  "name": "Main Street Community Center",
  "description": "Large event hall with full amenities",
  "addressId": "addr-123e4567-e89b-12d3-a456-426614174000",
  "capacity": 500,
  "amenities": ["parking", "wifi", "sound_system", "stage"],
  "jurisdictionId": "jur-123e4567-e89b-12d3-a456-426614174000",
  "acceptedCurrencies": ["USD"],
  "timezone": "America/Los_Angeles",
  "isActive": true,
  "createdAt": "2025-02-01T09:00:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-15T11:20:00Z",
  "updatedBy": "admin-user-id"
}
```

### Address

```json
{
  "id": "addr-123e4567-e89b-12d3-a456-426614174000",
  "street": "123 Main Street",
  "city": "Anytown",
  "state": "CA",
  "zipCode": "90210",
  "country": "US"
}
```

### Contact

```json
{
  "id": "contact-123e4567-e89b-12d3-a456-426614174002",
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+1-555-987-6543",
  "addressId": "addr-123e4567-e89b-12d3-a456-426614174000"
}
```

### Jurisdiction

```json
{
  "id": "jur-123e4567-e89b-12d3-a456-426614174000",
  "name": "San Francisco",
  "code": "SF-CA-US", // jurisdiction-specific identifier
  "type": "state", // state, province, country, local
  "parentJurisdictionId": "jur-123e4567-e89b-12d3-a456-426614174001",
  "isActive": true,
  "createdAt": "2025-01-15T10:00:00Z",
  "createdBy": "system",
  "updatedAt": "2025-06-20T14:30:00Z",
  "updatedBy": "admin-user-id"
}
```

### Jurisdiction Regulation

```json
{
  "id": "regulation-123e4567-e89b-12d3-a456-426614174000",
  "jurisdictionId": "jurisdiction-123e4567-e89b-12d3-a456-426614174000",
  "name": "Maximum Prize Value",
  "type": "prize_limit", // "ticket_price_limit", "event_duration", "sales_method", "reporting_requirement", "data_retention"
  "value": {
    "amount": 5000000,
    "currency": "USD"
  },
  "effectiveDate": "2025-01-01T00:00:00Z",
  "expirationDate": null,
  "overridable": false, // Can child jurisdictions override?
  "createdAt": "2025-01-01T00:00:00Z",
  "updatedAt": "2025-01-01T00:00:00Z"
}
```

### User
```json
{
  "id": "user-123e4567-e89b-12d3-a456-426614174000",
  "cognitoSub": "cognito-sub-123", // Primary key link to Cognito
  "displayName": "John Admin", // Cached for display only
  "organizations": [
    {
      "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
      "role": "admin",
      "isDefault": true,
      "joinedAt": "2025-01-01T00:00:00Z"
    },
    {
      "organizationId": "org-another-123",
      "role": "operator",
      "isDefault": false,
      "joinedAt": "2025-03-01T00:00:00Z"
    }
  ],
  "lastLogin": "2025-06-28T14:00:00Z",
  "isActive": true,
  "createdAt": "2025-01-01T00:00:00Z"
}
```

### Draw execution

```json
{
  "id": "draw-exec-123...",
  "raffleEventId": "123...",
  "state": "in_progress", // "pending", "in_progress", "completed", "failed"
  "startedAt": "2025-06-30T19:00:00Z",
  "rngSeed": "SEED-9876543210ABCDEF",
  "conductedBy": "admin-user-id",
  "systemChecksum": "SHA256-XXX...", // System state at draw time
  "createdAt": "2025-06-30T19:00:00Z",
  "updatedAt": "2025-06-30T19:00:00Z"
}
```

### Draw result

```json
{
  "id": "draw-result-123...",
  "drawExecutionId": "draw-exec-123...",
  "prizeId": "prize-123...",
  "position": 1,
  "winningTicketId": "ticket-789...",
  "winningNumber": 1245,
  "prizeValueAtDraw": 122500, // Calculated for percentage prizes
  "selectedAt": "2025-06-30T19:01:00Z"
}
```

### Prize claim

```json
{
  "id": "claim-123...",
  "drawResultId": "draw-result-123...",
  "claimedAt": "2025-06-30T20:15:00Z",
  "claimedBy": "winner-user-id",
  "verificationMethod": "validation_number",
  "verifiedBy": "admin-user-id"
}
```

### Critical event

```json
{
  "id": "event-987d1234-f56e-78a9-b012-345678901cde",
  "eventType": "RAFFLE_ACTIVATED",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "rsuId": null,
  "userId": "admin-user-id",
  "severity": "INFO",
  "description": "Raffle 'Summer Festival Raffle' transitioned from created to active state",
  "metadata": {
    "previousState": "created",
    "newState": "active",
    "salesStartTime": "2025-06-28T09:00:00Z"
  },
  "timestamp": "2025-06-28T09:00:00Z",
  "ttl": 1783142400
}
```

### System Configuration

```json
{
  "id": "sys-config-123...",
  "version": "1.0.0",
  "effectiveFrom": "2025-01-01T00:00:00Z",
  "effectiveTo": null,
  "settings": {
    "verification": {
      "scheduleIntervalHours": 24,
      "requiredBeforeDraw": true
    },
    "dataRetention": {
      "criticalEventsDays": 1095,
      "ticketDataDays": 2555
    },
    "rsu": {
      "defaultMaxOfflineTickets": 100,
      "defaultSyncIntervalMinutes": 5,
      "apiKeyRotationDays": 90,
      "validationNumberBlockSize": 100,  // NEW
      "offlineSaleExpirationMinutes": 1440  // NEW: 24 hours
    },
    "ticketGeneration": {  // NEW section
      "strategy": "just_in_time",  // vs "pre_generated"
      "batchSize": 1000  // For bulk operations
    }
  },
  "createdAt": "2025-01-01T00:00:00Z",
  "createdBy": "system"
}
```

### Validation Number Block (NEW)

```json
{
  "id": "vnb-123e4567-e89b-12d3-a456-426614174000",
  "rsuAllocationId": "alloc-123e4567-e89b-12d3-a456-426614174000",
  "rsuId": "rsu-456f7890-b12c-34d5-e678-901234567abc",
  "raffleEventId": "123e4567-e89b-12d3-a456-426614174000",
  "blockStart": "RSU001-2025-0001",
  "blockEnd": "RSU001-2025-0100",
  "blockSize": 100,
  "numbersUsed": 27,
  "nextSequence": 28,
  "createdAt": "2025-06-28T08:00:00Z",
  "expiresAt": null,
  "isActive": true
}
```

## Key Changes Summary

1. **Ticket Model**:
   - Removed `available` and `allocated` states
   - Tickets now always have an `orderId`
   - Added `soldAt` and `soldBy` fields

2. **RSU Model**:
   - Added `validationNumberBlock` to active allocations
   - Added `nextDrawNumber` tracking

3. **Order Model**:
   - Added `ticketCount` and `drawNumbers` for explicit tracking
   - Added offline sale tracking fields
   - Added `rsuAllocationId` link

4. **New Models**:
   - `RSU Offline Sale Record` for local SQLite storage
   - `Validation Number Block` for tracking pre-allocated validation numbers

5. **Raffle Event**:
   - Added `totalSold` counter
   - Added `sold` counter to each range

6. **Organization**:
   - Added RSU-specific settings for validation number management

These changes support:
- Just-in-time ticket creation
- Offline RSU sales with pre-allocated validation numbers
- Better tracking of ticket allocation and sales
- Simplified ticket state management
