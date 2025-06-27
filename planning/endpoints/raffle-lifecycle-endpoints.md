## Raffle lifecycle

### POST /raffle - create new raffle

Request body:
```
{
  "name": "Summer Festival Raffle",
  "description": "Annual summer fundraiser raffle",
  "organization": "Community Center Inc",
  "licenseNumber": "LIC-2025-001",
  "eventIdentifier": "SUMMER-2025",
  "location": "Main Street Community Center"
}
```

Response (201 created):
```
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Summer Festival Raffle",
  "description": "Annual summer fundraiser raffle",
  "organization": "Community Center Inc",
  "licenseNumber": "LIC-2025-001",
  "eventIdentifier": "SUMMER-2025",
  "location": "Main Street Community Center",
  "state": "created",
  "createdAt": "2025-06-27T14:30:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-27T14:30:00Z"
}
```

### GET /raffles/{raffleId} - get raffle details

Response
```
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Summer Festival Raffle",
  "description": "Annual summer fundraiser raffle",
  "organization": "Community Center Inc",
  "licenseNumber": "LIC-2025-001",
  "eventIdentifier": "SUMMER-2025",
  "location": "Main Street Community Center",
  "state": "active",
  "configuration": {
    "ticketPrice": 10.00,
    "maxTickets": 1000,
    "salesStartTime": "2025-06-28T09:00:00Z",
    "salesEndTime": "2025-06-30T18:00:00Z",
    "drawTime": "2025-06-30T19:00:00Z",
    "prizes": [
      {
        "position": 1,
        "description": "Grand Prize - iPad Pro",
        "value": 1200.00
      },
      {
        "position": 2,
        "description": "Second Prize - Gift Card",
        "value": 500.00
      }
    ]
  },
  "stats": {
    "ticketsSold": 245,
    "totalRevenue": 2450.00,
    "ticketsVoided": 3
  },
  "createdAt": "2025-06-27T14:30:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-28T09:15:00Z",
  "updatedBy": "admin-user-id"
}
```

### PUT /raffles/{raffleId} - update raffle configuration

Request body:
```
{
  "name": "Summer Festival Raffle - Updated",
  "description": "Annual summer fundraiser raffle with bonus prizes",
  "configuration": {
    "ticketPrice": 10.00,
    "maxTickets": 1500,
    "salesStartTime": "2025-06-28T09:00:00Z",
    "salesEndTime": "2025-06-30T18:00:00Z",
    "drawTime": "2025-06-30T19:00:00Z",
    "prizes": [
      {
        "position": 1,
        "description": "Grand Prize - iPad Pro + Apple Watch",
        "value": 1500.00
      },
      {
        "position": 2,
        "description": "Second Prize - Gift Card",
        "value": 500.00
      },
      {
        "position": 3,
        "description": "Third Prize - Dinner Voucher",
        "value": 100.00
      }
    ]
  }
}
```

Response
```
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Summer Festival Raffle - Updated",
  "description": "Annual summer fundraiser raffle with bonus prizes",
  "organization": "Community Center Inc",
  "licenseNumber": "LIC-2025-001",
  "eventIdentifier": "SUMMER-2025",
  "location": "Main Street Community Center",
  "state": "configured",
  "configuration": {
    "ticketPrice": 10.00,
    "maxTickets": 1500,
    "salesStartTime": "2025-06-28T09:00:00Z",
    "salesEndTime": "2025-06-30T18:00:00Z",
    "drawTime": "2025-06-30T19:00:00Z",
    "prizes": [
      {
        "position": 1,
        "description": "Grand Prize - iPad Pro + Apple Watch",
        "value": 1500.00
      },
      {
        "position": 2,
        "description": "Second Prize - Gift Card",
        "value": 500.00
      },
      {
        "position": 3,
        "description": "Third Prize - Dinner Voucher",
        "value": 100.00
      }
    ]
  },
  "stats": {
    "ticketsSold": 0,
    "totalRevenue": 0.00,
    "ticketsVoided": 0
  },
  "createdAt": "2025-06-27T14:30:00Z",
  "createdBy": "admin-user-id",
  "updatedAt": "2025-06-27T15:45:00Z",
  "updatedBy": "admin-user-id"
}
```

### DELETE /raffles/{raffleId} - cancel raffle

Request body:
```
{
  "reason": "Event cancelled due to weather conditions"
}
```

Response
```
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "state": "cancelled",
  "cancelledAt": "2025-06-27T16:00:00Z",
  "cancelledBy": "admin-user-id",
  "cancellationReason": "Event cancelled due to weather conditions",
  "message": "Raffle has been cancelled. All ticket holders will be refunded."
}
```
