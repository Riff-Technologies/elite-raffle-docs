## Models

These models are proposals and do not necessarily represent the definitive and final models.  Changes should be suggested where required.

### Raffle event
```
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
    "jackpotSeed": 1000.00,
    "revenueCalculationMethod": "gross_revenue",
    "maxTickets": 2000,
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

### Prize

```
{
  "id": "prize-123e4567-e89b-12d3-a456-426614174001",
  "name": "Grand Prize",
  "description": "iPad Pro + Apple Watch",
  "type": "fixed",
  "value": 1500.00,
  "position": 1,
  "winnerCount": 1,
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
  "createdAt": "2025-06-27T13:00:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-27T13:00:00Z",
  "updatedBy": "admin-user-id"
}
```

### Ticket Package

```
{
  "id": "pkg-123e4567-e89b-12d3-a456-426614174002",
  "name": "Value Pack",
  "description": "5 tickets - Best value!",
  "ticketCount": 5,
  "price": 20.00,
  "isActive": true,
  "displayOrder": 2,
  "activeFrom": "2025-06-28T09:00:00Z",
  "activeTo": "2025-06-28T15:00:00Z",
  "perkIds": [
    "perk-123e4567-e89b-12d3-a456-426614174001"
  ],
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
  "createdAt": "2025-06-27T14:05:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-28T10:30:00Z",
  "updatedBy": "admin-user-id"
}
```

### Organization

```
{
  "id": "org-123e4567-e89b-12d3-a456-426614174000",
  "name": "Community Center Inc",
  "description": "Local community organization",
  "licenseNumber": "LIC-2025-001",
  "contactInfo": {
    "email": "admin@communitycenter.org",
    "phone": "+1-555-123-4567",
    "address": {
      "street": "123 Main Street",
      "city": "Anytown",
      "state": "CA",
      "zipCode": "90210",
      "country": "US"
    }
  },
  "settings": {
    "defaultRevenueCalculationMethod": "gross_revenue",
    "taxSettings": {
      "taxRate": 8.25,
      "taxIncluded": false
    }
  },
  "isActive": true,
  "createdAt": "2025-01-15T10:00:00Z",
  "createdBy": "system",
  "updatedAt": "2025-06-20T14:30:00Z",
  "updatedBy": "admin-user-id"
}
```

### Venue

```
{
  "id": "venue-987f6543-a21c-43d5-b678-123456789abc",
  "name": "Main Street Community Center",
  "description": "Large event hall with full amenities",
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
  "address": {
    "street": "123 Main Street",
    "city": "Anytown",
    "state": "CA",
    "zipCode": "90210",
    "country": "US"
  },
  "capacity": 500,
  "amenities": ["parking", "wifi", "sound_system", "stage"],
  "isActive": true,
  "createdAt": "2025-02-01T09:00:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-15T11:20:00Z",
  "updatedBy": "admin-user-id"
}
```

### RSU

```
{
  "id": "rsu-456f7890-b12c-34d5-e678-901234567abc",
  "name": "RSU-001",
  "description": "Main entrance sales terminal",
  "organizationId": "org-123e4567-e89b-12d3-a456-426614174000",
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
    "raffleId": "123e4567-e89b-12d3-a456-426614174000",
    "startTicketNumber": 1001,
    "endTicketNumber": 1100,
    "ticketsRemaining": 73
  },
  "lastSync": "2025-06-29T15:45:00Z",
  "isActive": true,
  "createdAt": "2025-06-20T08:00:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-29T15:45:00Z",
  "updatedBy": "system"
}
```

### Ticket

```
{
  "id": "ticket-789a0123-c45d-67e8-f901-234567890abc",
  "raffleId": "123e4567-e89b-12d3-a456-426614174000",
  "drawNumber": 1001,
  "validationNumber": "VAL-9F8E7D6C5B4A",
  "state": "sold",
  "saleInfo": {
    "packageId": "pkg-123e4567-e89b-12d3-a456-426614174002",
    "packageName": "Value Pack",
    "price": 20.00,
    "soldAt": "2025-06-28T14:30:00Z",
    "soldBy": "rsu-456f7890-b12c-34d5-e678-901234567abc",
    "operatorId": "operator-user-id"
  },
  "customerInfo": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+1-555-987-6543"
  },
  "createdAt": "2025-06-28T14:30:00Z",
  "updatedAt": "2025-06-28T14:30:00Z"
}
```

### Order

```
{
  "id": "order-321b4567-d89e-12f3-a456-426614174000",
  "raffleId": "123e4567-e89b-12d3-a456-426614174000",
  "packageId": "pkg-123e4567-e89b-12d3-a456-426614174002",
  "packageName": "Value Pack",
  "ticketCount": 5,
  "totalAmount": 20.00,
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
  "source": "online",
  "createdAt": "2025-06-28T14:25:00Z",
  "updatedAt": "2025-06-28T14:30:00Z"
}
```

### Draw

```
{
  "id": "draw-654c7890-e12f-34a5-b678-901234567def",
  "raffleId": "123e4567-e89b-12d3-a456-426614174000",
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
      "calculatedValue": 1225.00,
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

```
{
  "id": "event-987d1234-f56e-78a9-b012-345678901cde",
  "eventType": "RAFFLE_ACTIVATED",
  "raffleId": "123e4567-e89b-12d3-a456-426614174000",
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

### Missing Models to Consider:

- User/Admin Model - For authentication and authorization
- Operator Model - RSU operators with device assignments
- Payment Provider Model - Configuration for Stripe, etc.
- Audit Log Model - For non-critical events and admin actions
- Report Model - For GLI-31 required reports storage
- Reconciliation Model - Track reconciliation process state
