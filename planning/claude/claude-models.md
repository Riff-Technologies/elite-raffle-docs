# Data models

## Raffle events

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
