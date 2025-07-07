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
  "createdAt": "2025-06-27T14:30:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-28T09:15:00Z",
  "updatedBy": "admin-user-id"
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
  "activeFrom": "2025-06-28T09:00:00Z",
  "activeTo": "2025-06-28T15:00:00Z",
  "perkIds": ["perk-123e4567-e89b-12d3-a456-426614174001"],
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
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
  "totalInventory": 50, // null if unlimited
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
    "defaultRevenueCalculationMethod": "gross_revenue", // gross_revenue, net_revenue
    "taxSettings": {
      "taxRate": 8.25,
      "taxIncluded": false
    }
  },
  "defaultVenueId": "venue-987f6543-a21c-43d5-b678-123456789abc",
  "venues": ["venue-987f6543-a21c-43d5-b678-123456789abc"],
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

### RSU

Note: if both `organizationId` and `venueId` are null, the device is an "admin" RSU and can enroll in any active raffle event.

```json
{
  "id": "rsu-456f7890-b12c-34d5-e678-901234567abc",
  "name": "RSU-001",
  "description": "Main entrance sales terminal",
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000", // can be null if it's not associated with an organization
  "venueId": null, // can be null if it's not associated with a specific venue
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
  "currentAllocation": {
    "id": "allocation-123e4567-e89b-12d3-a456-426614174000",
    "eventId": "123e4567-e89b-12d3-a456-426614174000",
    "rsuId": "rsu-456f7890-b12c-34d5-e678-901234567abc",
    "startTicketNumber": 1001,
    "endTicketNumber": 1100,
    "allocatedAt": "2025-06-28T08:00:00Z",
    "allocatedBy": "admin-user-id",
    "ticketsSold": 27,
    "ticketsVoided": 0,
    "isActive": true
  },
  "lastSync": "2025-06-29T15:45:00Z",
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
  "id": "allocation-123e4567-e89b-12d3-a456-426614174000",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "rsuId": "rsu-456f7890-b12c-34d5-e678-901234567abc",
  "startTicketNumber": 1001,
  "endTicketNumber": 1100,
  "allocatedAt": "2025-06-28T08:00:00Z",
  "allocatedBy": "admin-user-id",
  "ticketsSold": 27,
  "ticketsVoided": 0,
  "isActive": true
}
```

### Ticket

```json
{
  "id": "ticket-789a0123-c45d-67e8-f901-234567890abc",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "drawNumber": 1001,
  "state": "sold", // "available", "allocated", "sold", "voided"
  "allocationId": "allocation-123e4567-e89b-12d3-a456-426614174000", // null if not allocated
  "orderId": "order-321b4567-d89e-12f3-a456-426614174000", // null if not sold
  "createdAt": "2025-06-28T14:30:00Z",
  "updatedAt": "2025-06-28T14:30:00Z"
}
```

### Order

```json
{
  "id": "order-321b4567-d89e-12f3-a456-426614174000",
  "validationNumber": "VAL-9F8E7D6C5B4A",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "packageId": "pkg-123e4567-e89b-12d3-a456-426614174002",
  "totalAmount": 2000,
  "currency": "USD",
  "paymentStatus": "completed",
  "paymentReference": "stripe_pi_1234567890",
  "ticketIds": [
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
  "source": "online", // "online", "rsu", "admin"
  "rsuId": null, // which RSU if source="rsu"
  "operatorId": null, // who processed the sale if it was an RSU
  "createdAt": "2025-06-28T14:25:00Z",
  "createdBy": "operator-user-id",
  "updatedAt": "2025-06-28T14:30:00Z"
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

### Operator

```json
{
  "id": "operator-123e4567-e89b-12d3-a456-426614174000",
  "userId": "user-123e4567-e89b-12d3-a456-426614174000",
  "isActive": true,
  "createdAt": "2025-06-01T00:00:00Z",
  "updatedAt": "2025-06-28T00:00:00Z"
}
```

### Reconciliation

```json
{
  "id": "reconciliation-123e4567-e89b-12d3-a456-426614174000",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "state": "in_progress", // "in_progress", "completed", "failed"
  "rsuStatuses": [
    {
      "rsuId": "rsu-456f7890-b12c-34d5-e678-901234567abc",
      "status": "synced", // "pending", "syncing", "synced", "error"
      "ticketsSold": 27,
      "ticketsVoided": 0,
      "lastSyncAt": "2025-06-30T18:30:00Z"
    }
  ],
  "totalTicketsSold": 1245,
  "totalRevenue": 498000,
  "currency": "USD",
  "startedAt": "2025-06-30T18:00:00Z",
  "completedAt": null,
  "completedBy": null,
  "notes": null
}
```

### Draw

```json
{
  "id": "draw-654c7890-e12f-34a5-b678-901234567def",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "state": "completed",
  "drawTime": "2025-06-30T19:00:00Z",
  "results": [
    {
      "prizeId": "prize-123e4567-e89b-12d3-a456-426614174001",
      "prizeName": "Grand Prize",
      "position": 1,
      "winningNumbers": [1245],
      "winners": [
        {
          "ticketId": "ticket-winner-123",
          "drawNumber": 1245,
          "validationNumber": "VAL-WINNER123",
          "customerName": "Jane Smith",
          "claimStatus": "unclaimed"
        }
      ]
    },
    {
      "prizeId": "prize-123e4567-e89b-12d3-a456-426614174002",
      "prizeName": "50/50 Split",
      "position": 2,
      "calculatedValue": 122500,
      "winningNumbers": [1678],
      "winners": [
        {
          "ticketId": "ticket-winner-456",
          "drawNumber": 1678,
          "validationNumber": "VAL-WINNER456",
          "customerName": "Bob Johnson",
          "claimStatus": "claimed",
          "claimedAt": "2025-06-30T20:15:00Z"
        }
      ]
    }
  ],
  "rngSeed": "SEED-9876543210ABCDEF",
  "verificationChecksum": "SHA256-ABCDEF1234567890",
  "conductedBy": "admin-user-id",
  "createdAt": "2025-06-30T19:00:00Z",
  "updatedAt": "2025-06-30T20:15:00Z"
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
