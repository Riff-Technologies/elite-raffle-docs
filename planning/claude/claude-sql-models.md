# Electronic Raffle System - Database Schema

Flow:

1. Raffle Created → Define max_tickets
1. Raffle Configured → Generate all ticket records (state: ‘available’)
1. RSU Allocation → Update tickets to ‘allocated’ state
1. Sale → Create order with validation number, update tickets to ‘sold’
1. Draw → Select winners from sold tickets

## Service Layer Responsibilities

The following logic should be implemented in the service layer, not database triggers:

- State transition validation
- Counter updates for allocations
- Cascade state changes (e.g., order → tickets)
- Validation number generation
- Critical event logging
- User reference synchronization with Cognito

## Core Tables

### user_reference

```sql
id UUID PRIMARY KEY  -- matches Cognito sub
email VARCHAR(255)  -- for display purposes only
display_name VARCHAR(255)
last_sync_at TIMESTAMPTZ DEFAULT NOW()

-- Note: Created/updated automatically when Cognito user first accesses system
```

### raffle_event

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id)
venue_id UUID NOT NULL REFERENCES venue(id)
name VARCHAR(255) NOT NULL
description TEXT
event_identifier VARCHAR(100) UNIQUE
state VARCHAR(50) NOT NULL DEFAULT 'created'  -- created, configured, active, sales_closed, reconciling, reconciled, drawing_ready, completed, cancelled
sales_start_time TIMESTAMPTZ
sales_end_time TIMESTAMPTZ
draw_time TIMESTAMPTZ
jackpot_seed_cents BIGINT DEFAULT 0
revenue_calculation_method VARCHAR(50) NOT NULL DEFAULT 'gross_revenue'
max_tickets INTEGER  -- defines total tickets to generate
reconciliation_confirmed_at TIMESTAMPTZ
reconciliation_confirmed_by UUID REFERENCES user_reference(id)
cancellation_reason TEXT  -- Required when state = 'cancelled'
suspension_reason TEXT  -- Populated when suspended
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

-- State constraints
CONSTRAINT check_raffle_state CHECK (state IN ('created', 'configured', 'active', 'sales_closed', 'reconciling', 'reconciled', 'drawing_ready', 'completed', 'cancelled'))
CONSTRAINT check_reconciliation_data CHECK (
    (state NOT IN ('reconciled', 'drawing_ready', 'completed') OR 
     (reconciliation_confirmed_at IS NOT NULL AND reconciliation_confirmed_by IS NOT NULL))
)
CONSTRAINT check_cancellation_reason CHECK (
    (state != 'cancelled' OR cancellation_reason IS NOT NULL)
)
```

### ticket_package

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id)
name VARCHAR(255) NOT NULL
description TEXT
ticket_count INTEGER NOT NULL CHECK (ticket_count > 0)
price_cents BIGINT NOT NULL CHECK (price_cents >= 0)
is_active BOOLEAN NOT NULL DEFAULT true
display_order INTEGER NOT NULL DEFAULT 0
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)
```

### raffle_ticket_package

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
ticket_package_id UUID NOT NULL REFERENCES ticket_package(id)
is_active BOOLEAN NOT NULL DEFAULT true
display_order INTEGER NOT NULL DEFAULT 0
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
UNIQUE(raffle_event_id, ticket_package_id)
```

### prize

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
name VARCHAR(255) NOT NULL
description TEXT
prize_type VARCHAR(50) NOT NULL  -- fixed, percentage
value_cents BIGINT  -- for fixed prizes
percentage DECIMAL(5,2)  -- for percentage prizes (e.g., 50.00 for 50%)
position INTEGER NOT NULL
winner_count INTEGER NOT NULL DEFAULT 1
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_prize_value CHECK (
    (prize_type = 'fixed' AND value_cents IS NOT NULL AND percentage IS NULL) OR
    (prize_type = 'percentage' AND percentage IS NOT NULL AND value_cents IS NULL)
)
```

### rsu

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id)
device_identifier VARCHAR(255) UNIQUE NOT NULL  -- MAC address or unique ID
name VARCHAR(100) NOT NULL
state VARCHAR(50) NOT NULL DEFAULT 'registered'  -- registered, enrolled, active, closing, closed, reconciled, suspended, error
max_offline_tickets INTEGER NOT NULL DEFAULT 100
last_sync_at TIMESTAMPTZ
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_rsu_state CHECK (state IN ('registered', 'enrolled', 'active', 'closing', 'closed', 'reconciled', 'suspended', 'error'))
```

### rsu_allocation

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
rsu_id UUID NOT NULL REFERENCES rsu(id)
start_number INTEGER NOT NULL
end_number INTEGER NOT NULL
tickets_sold INTEGER NOT NULL DEFAULT 0  -- counter for tracking (updated by service layer)
tickets_voided INTEGER NOT NULL DEFAULT 0  -- counter for tracking (updated by service layer)
allocated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
allocated_by UUID REFERENCES user_reference(id)
deallocated_at TIMESTAMPTZ
deallocated_by UUID REFERENCES user_reference(id)
is_active BOOLEAN NOT NULL DEFAULT true
UNIQUE(raffle_event_id, start_number, end_number)
CONSTRAINT check_number_range CHECK (end_number >= start_number)
CONSTRAINT check_allocation_counts CHECK (tickets_sold + tickets_voided <= (end_number - start_number + 1))
```

### customer

```sql
id UUID PRIMARY KEY
name VARCHAR(255) NOT NULL
email VARCHAR(255)
phone VARCHAR(50)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### order

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
customer_id UUID REFERENCES customer(id)
ticket_package_id UUID NOT NULL REFERENCES ticket_package(id)
ticket_count INTEGER NOT NULL
total_amount_cents BIGINT NOT NULL
validation_number VARCHAR(100) UNIQUE NOT NULL  -- Generated by service layer on completion
payment_status VARCHAR(50) NOT NULL DEFAULT 'pending'  -- pending, completed, failed, refunded
payment_reference VARCHAR(255)  -- Stripe payment ID, etc.
source VARCHAR(50) NOT NULL  -- online, rsu
rsu_id UUID REFERENCES rsu(id)  -- if sold via RSU
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_payment_status CHECK (payment_status IN ('pending', 'completed', 'failed', 'refunded'))
CONSTRAINT check_validation_number CHECK (
    (payment_status != 'completed' OR validation_number IS NOT NULL)
)
```

### ticket

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
draw_number INTEGER NOT NULL
state VARCHAR(50) NOT NULL DEFAULT 'available'  -- available, allocated, sold, voided, winner, claimed

-- Allocation fields
allocated_to_rsu_id UUID REFERENCES rsu(id)
allocated_at TIMESTAMPTZ
allocation_id UUID REFERENCES rsu_allocation(id)

-- Sale fields (NULL until sold)
order_id UUID REFERENCES order(id)
sold_at TIMESTAMPTZ
sold_by UUID REFERENCES user_reference(id)

-- Void fields
voided_at TIMESTAMPTZ
voided_by UUID REFERENCES user_reference(id)
voided_reason TEXT

created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

UNIQUE(raffle_event_id, draw_number)
CONSTRAINT check_ticket_state CHECK (state IN ('available', 'allocated', 'sold', 'voided', 'winner', 'claimed'))
CONSTRAINT check_sold_data CHECK (
    (state != 'sold' OR (order_id IS NOT NULL AND sold_at IS NOT NULL))
)
CONSTRAINT check_voided_data CHECK (
    (state != 'voided' OR (voided_at IS NOT NULL AND voided_by IS NOT NULL AND voided_reason IS NOT NULL))
)
```

### draw

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id) UNIQUE
state VARCHAR(50) NOT NULL DEFAULT 'pending'  -- pending, generating, completed
draw_time TIMESTAMPTZ
rng_seed VARCHAR(255)  -- for audit trail
verification_checksum VARCHAR(255)
conducted_by UUID REFERENCES user_reference(id)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_draw_state CHECK (state IN ('pending', 'generating', 'completed'))
CONSTRAINT check_completed_data CHECK (
    (state != 'completed' OR (draw_time IS NOT NULL AND conducted_by IS NOT NULL))
)
```

### draw_result

```sql
id UUID PRIMARY KEY
draw_id UUID NOT NULL REFERENCES draw(id)
prize_id UUID NOT NULL REFERENCES prize(id)
winning_ticket_id UUID NOT NULL REFERENCES ticket(id)
prize_value_cents BIGINT NOT NULL  -- actual value at time of draw (calculated for percentage prizes)
claim_status VARCHAR(50) NOT NULL DEFAULT 'unclaimed'  -- unclaimed, claimed
claimed_at TIMESTAMPTZ
claimed_by UUID REFERENCES user_reference(id)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

CONSTRAINT check_claim_status CHECK (claim_status IN ('unclaimed', 'claimed'))
CONSTRAINT check_claimed_data CHECK (
    (claim_status != 'claimed' OR (claimed_at IS NOT NULL AND claimed_by IS NOT NULL))
)
```

### address

```sql
id UUID PRIMARY KEY
address_type VARCHAR(50) NOT NULL  -- physical, mailing
street_line_1 VARCHAR(500) NOT NULL
street_line_2 VARCHAR(500)
city VARCHAR(255) NOT NULL
state_province VARCHAR(100) NOT NULL
postal_code VARCHAR(20) NOT NULL
country VARCHAR(2) NOT NULL
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

### contact

```sql
id UUID PRIMARY KEY
contact_type VARCHAR(50) NOT NULL  -- primary, regulatory, emergency
name VARCHAR(255) NOT NULL
email VARCHAR(255)
phone VARCHAR(50)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

### license

```sql
id UUID PRIMARY KEY
license_type VARCHAR(100) NOT NULL  -- venue_permit, charitable_gaming
license_number VARCHAR(100) NOT NULL
issuing_authority VARCHAR(255) NOT NULL
issue_date TIMESTAMPTZ NOT NULL
expiry_date TIMESTAMPTZ
status VARCHAR(20) NOT NULL DEFAULT 'active'  -- active, suspended, expired, revoked
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

## Jurisdiction Tables

### jurisdiction_regulation

```sql
id UUID PRIMARY KEY
max_prize_value_cents BIGINT
max_ticket_price_cents BIGINT
max_event_duration_hours INTEGER
data_retention_years INTEGER NOT NULL DEFAULT 7
allows_online_sales BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

### jurisdiction

```sql
id UUID PRIMARY KEY
name VARCHAR(255) NOT NULL
code VARCHAR(50) NOT NULL UNIQUE  -- e.g., "CA-US", "SF-CA-US"
type VARCHAR(50) NOT NULL  -- state, province, country, local
country VARCHAR(2) NOT NULL
parent_jurisdiction_id UUID REFERENCES jurisdiction(id)
primary_contact_id UUID REFERENCES contact(id)
current_regulations_id UUID REFERENCES jurisdiction_regulation(id)
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

## Venue Tables

### venue

```sql
id UUID PRIMARY KEY
jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id)
parent_venue_id UUID REFERENCES venue(id)
name VARCHAR(255) NOT NULL
description TEXT
venue_identifier VARCHAR(100)  -- jurisdiction-specific identifier
capacity INTEGER
physical_address_id UUID REFERENCES address(id)
primary_contact_id UUID REFERENCES contact(id)
venue_permit_id UUID REFERENCES license(id)
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

## Organization Tables

### organization_setting

```sql
id UUID PRIMARY KEY
default_revenue_calculation_method VARCHAR(50) DEFAULT 'gross_revenue'
allows_online_ticket_sales BOOLEAN NOT NULL DEFAULT true
max_event_duration_hours INTEGER
max_concurrent_events INTEGER DEFAULT 5
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

### organization

```sql
id UUID PRIMARY KEY
primary_venue_id UUID REFERENCES venue(id)
name VARCHAR(255) NOT NULL
description TEXT
organization_type VARCHAR(50) NOT NULL  -- nonprofit, charity, club
tax_id_number VARCHAR(50)
mailing_address_id UUID REFERENCES address(id)
primary_contact_id UUID REFERENCES contact(id)
gaming_license_id UUID REFERENCES license(id)
current_settings_id UUID REFERENCES organization_setting(id)
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
```

## Relationship Tables

### organization_venue

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id) ON DELETE CASCADE
venue_id UUID NOT NULL REFERENCES venue(id) ON DELETE CASCADE
relationship_type VARCHAR(50) NOT NULL DEFAULT 'authorized'  -- primary, authorized
authorized_date TIMESTAMPTZ NOT NULL DEFAULT NOW()
authorized_by UUID
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
UNIQUE(organization_id, venue_id)
```

## Extension Tables

### entity_attribute

```sql
id UUID PRIMARY KEY
entity_type VARCHAR(50) NOT NULL  -- jurisdiction, venue, organization
entity_id UUID NOT NULL
attribute_name VARCHAR(100) NOT NULL
attribute_value TEXT
data_type VARCHAR(20) DEFAULT 'string'  -- string, number, boolean, json
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID
UNIQUE(entity_type, entity_id, attribute_name)
```

## Entity Relationships

```
jurisdiction (1) ←→ (1) jurisdiction_regulation
jurisdiction (1) ←→ (1) contact [primary_contact_id]
jurisdiction (1) ←→ (*) venue
jurisdiction (1) ←→ (*) jurisdiction [parent_jurisdiction_id]

venue (1) ←→ (1) address [physical_address_id]
venue (1) ←→ (1) contact [primary_contact_id]
venue (1) ←→ (1) license [venue_permit_id]
venue (1) ←→ (*) venue [parent_venue_id]

organization (1) ←→ (1) organization_setting
organization (1) ←→ (1) address [mailing_address_id]
organization (1) ←→ (1) contact [primary_contact_id]
organization (1) ←→ (1) license [gaming_license_id]
organization (1) ←→ (1) venue [primary_venue_id]

organization (*) ←→ (*) venue [via organization_venue]

entity_attribute (*) ←→ (1) [any entity via entity_type + entity_id]

Ticket State Transitions

available → Created when raffle configured
allocated → Assigned to an RSU for sale
sold → Purchased by customer (linked to order)
voided → Refunded/cancelled
winner → Selected in draw
claimed → Prize collected

Order-Ticket Relationship

One order has one validation number
One order can contain multiple tickets
All tickets in an order share the same validation number
Validation lookup returns all tickets in the order

RSU Allocation Flow

RSU enrolled in raffle
Number range allocated (e.g., 1001-1100)
Tickets in range updated to 'allocated' state
RSU can only sell from allocated range
Online sales use 'available' (non-allocated) tickets
```

## Key Indexes

```sql
-- Jurisdiction
CREATE INDEX idx_jurisdiction_code ON jurisdiction(code);
CREATE INDEX idx_jurisdiction_parent ON jurisdiction(parent_jurisdiction_id);
CREATE INDEX idx_jurisdiction_active ON jurisdiction(is_active);

-- Venue
CREATE INDEX idx_venue_jurisdiction ON venue(jurisdiction_id);
CREATE INDEX idx_venue_parent ON venue(parent_venue_id);
CREATE INDEX idx_venue_active ON venue(is_active);

-- Organization
CREATE INDEX idx_organization_primary_venue ON organization(primary_venue_id);
CREATE INDEX idx_organization_type ON organization(organization_type);
CREATE INDEX idx_organization_active ON organization(is_active);
CREATE INDEX idx_organization_tax_id ON organization(tax_id_number);

-- Organization-Venue Relationship
CREATE INDEX idx_organization_venue_org ON organization_venue(organization_id);
CREATE INDEX idx_organization_venue_venue ON organization_venue(venue_id);
CREATE INDEX idx_organization_venue_active ON organization_venue(is_active);

-- Entity Attributes
CREATE INDEX idx_entity_attribute_entity ON entity_attribute(entity_type, entity_id);
CREATE INDEX idx_entity_attribute_name ON entity_attribute(attribute_name);

-- Raffle Event
CREATE INDEX idx_raffle_event_state ON raffle_event(state);
CREATE INDEX idx_raffle_event_org ON raffle_event(organization_id);
CREATE INDEX idx_raffle_event_venue ON raffle_event(venue_id);
CREATE INDEX idx_raffle_event_dates ON raffle_event(sales_start_time, sales_end_time);

-- Ticket
CREATE INDEX idx_ticket_state ON ticket(state);
CREATE INDEX idx_ticket_raffle ON ticket(raffle_event_id);
CREATE INDEX idx_ticket_order ON ticket(order_id);
CREATE INDEX idx_ticket_allocation ON ticket(allocation_id);
CREATE INDEX idx_ticket_draw_number ON ticket(raffle_event_id, draw_number);

-- Order
CREATE INDEX idx_order_raffle ON order(raffle_event_id);
CREATE INDEX idx_order_validation ON order(validation_number);
CREATE INDEX idx_order_payment_status ON order(payment_status);
CREATE INDEX idx_order_customer ON order(customer_id);

-- RSU
CREATE INDEX idx_rsu_state ON rsu(state);
CREATE INDEX idx_rsu_org ON rsu(organization_id);

-- Draw
CREATE INDEX idx_draw_raffle ON draw(raffle_event_id);
CREATE INDEX idx_draw_state ON draw(state);

-- Draw Result
CREATE INDEX idx_draw_result_draw ON draw_result(draw_id);
CREATE INDEX idx_draw_result_ticket ON draw_result(winning_ticket_id);
CREATE INDEX idx_draw_result_claim_status ON draw_result(claim_status);

-- Shared Components
CREATE INDEX idx_address_type ON address(address_type);
CREATE INDEX idx_contact_type ON contact(contact_type);
CREATE INDEX idx_license_type ON license(license_type);
CREATE INDEX idx_license_status ON license(status);
```

## Extensibility Pattern

Use `entity_attribute` for future features:

```sql
-- Example: Add venue amenities
INSERT INTO entity_attribute (entity_type, entity_id, attribute_name, attribute_value, data_type)
VALUES ('venue', 'venue-uuid', 'parking_spaces', '100', 'number');

-- Example: Add organization banking
INSERT INTO entity_attribute (entity_type, entity_id, attribute_name, attribute_value, data_type)
VALUES ('organization', 'org-uuid', 'bank_account', '{"bank": "Wells Fargo", "masked": "****1234"}', 'json');
```

### Data Integrity Rules

- Ticket Generation: All tickets created when raffle state changes to ‘configured’
- Unique Draw Numbers: Database enforces one ticket per draw number per raffle
- Allocation Boundaries: RSUs can only sell tickets within their allocated range
- Validation Uniqueness: Each order has a globally unique validation number
- State Consistency: Ticket state changes follow defined transitions
- Reconciliation: Cannot draw until raffle is reconciled (confirmed by admin)
- User References: All user operations require valid user_reference record
- Organization/Venue Active: Raffle can only be activated if both are active

### Extensions for Future Phases

The schema supports future additions without breaking changes:

- Perks/Benefits: Add via entity_attribute or dedicated tables
- Scheduled Pricing: Add time-based pricing to ticket_package
- Customer Loyalty: Add customer preference tracking
- Multi-draw: Extend draw/draw_result for multiple drawings
- Reporting: Add materialized views for performance