## API Gateway Endpoints

### Authentication & User Management

```
# User reference management (sync with Cognito)
GET    /v1/users/me             # Get/create user reference from Cognito token
PUT    /v1/users/me/organizations/{orgId}  # Set default organization
GET    /v1/users                # List users in organization (admin only)
```

### Organization & Venue Management

```
GET    /v1/organizations
GET    /v1/organizations/{id}
GET    /v1/organizations/{id}/venues
GET    /v1/organizations/{id}/rsus
GET    /v1/organizations/{id}/users  # Users in this org

GET    /v1/venues
GET    /v1/venues/{id}
```

### Event Management

```
POST   /v1/events
GET    /v1/events               # List with filters
GET    /v1/events/{id}          # Includes all event data
PUT    /v1/events/{id}          # Update configuration (includes packages/prizes)
DELETE /v1/events/{id}          # Cancel with reason

# State transitions
POST   /v1/events/{id}/activate
POST   /v1/events/{id}/suspend
POST   /v1/events/{id}/resume
POST   /v1/events/{id}/close    # Manual close

# Admin override
POST   /v1/admin/events/{id}/state  # Force state change

# Event Data
GET    /v1/events/{id}/packages          # List ticket packages for event
GET    /v1/events/{id}/prizes            # List prizes for event
```

### Package & Prize Templates (Organization-scoped)

```
# Ticket Package Templates
GET    /v1/organizations/{orgId}/packages     # List org's templates
POST   /v1/organizations/{orgId}/packages     # Create template
GET    /v1/organizations/{orgId}/packages/{id}
PUT    /v1/organizations/{orgId}/packages/{id}
DELETE /v1/organizations/{orgId}/packages/{id}

# Prize Templates
GET    /v1/organizations/{orgId}/prizes       # List org's templates
POST   /v1/organizations/{orgId}/prizes       # Create template
GET    /v1/organizations/{orgId}/prizes/{id}
PUT    /v1/organizations/{orgId}/prizes/{id}
DELETE /v1/organizations/{orgId}/prizes/{id}
```

### RSU Management

```
GET    /v1/rsus                 # List with current state/stats
GET    /v1/rsus/{id}           # Full RSU details including state
POST   /v1/rsus                 # Register new RSU
PUT    /v1/rsus/{id}           # Update RSU config

# RSU Operations
GET    /v1/rsus/{id}/events    # Available events for enrollment
POST   /v1/rsus/{id}/enroll    # Enroll in event
POST   /v1/rsus/{id}/sync      # RSU heartbeat / sync

# Allocation Management (supports both sequential and random)
POST   /v1/rsus/{id}/allocations  # Request allocation
GET    /v1/rsus/{id}/allocations  # List all allocations for RSU
```

### Sales & Orders

```
POST   /v1/orders               # Create order (online/RSU)
GET    /v1/orders/{id}
POST   /v1/orders/{id}/void    # Void entire order
GET    /v1/orders               # List with filters

# Payment endpoints
POST   /v1/orders/{id}/payment/initiate  # Start payment flow
POST   /v1/orders/{id}/payment/confirm   # Confirm payment (webhook)
POST   /v1/orders/{id}/refund           # Initiate refund
POST   /v1/orders/{id}/reprint          # Reprint tickets (audit log)

# RSU batch operations
POST   /v1/rsus/{id}/orders/batch  # Upload offline sales
```

### Reconciliation

```
GET    /v1/events/{id}/reconciliation      # Full reconciliation status
POST   /v1/events/{id}/reconciliation/confirm  # Manual confirmation
```

### Draw Management & Prize Claims

```
POST   /v1/events/{id}/draw    # Initiate draw
GET    /v1/events/{id}/draw    # Get draw info + results if available

# Winner validation and prize claims
POST   /v1/events/{id}/claims/validate  # Validate ticket + validation number
POST   /v1/events/{id}/claims/{claimId}/confirm  # Confirm prize claimed
```

The validation endpoint would accept:

```json
{
  "ticketNumber": 12345,
  "validationNumber": "VAL-9F8E7D6C5B4A"
}
```

### Reporting (GLI-31 Section 2.8)

```
GET    /v1/reports/events/{id}/summary      # Event summary report
GET    /v1/reports/events/{id}/sales        # Detailed sales report
GET    /v1/reports/events/{id}/reconciliation  # Reconciliation report
GET    /v1/reports/events/{id}/prizes       # Prize distribution report
GET    /v1/reports/revenue                  # Revenue reports (date range)
GET    /v1/reports/rsus/{id}/activity      # RSU activity report
```

### System & Compliance

```
GET    /v1/system/verification  # System checksums
GET    /v1/system/health       # Health check
POST   /v1/system/verify       # Trigger verification
GET    /v1/system/configuration         # Retrieve system configuration

# Critical Events
GET    /v1/events/critical     # Query critical events log
POST   /v1/events/critical/export  # Export for regulators
```

### Public Endpoints (No Auth)

```
GET    /v1/public/events/{id}   # Public event info
GET    /v1/public/events/{id}/jackpot  # Current jackpot
GET    /v1/public/events/{id}/results  # Draw results
GET    /v1/public/verification  # System checksums
```
