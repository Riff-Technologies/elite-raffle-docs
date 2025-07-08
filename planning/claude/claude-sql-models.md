# Electronic Raffle System - Database Schema

Flow:

1. Raffle Created → Define max_tickets
2. Raffle Configured → Generate all ticket records (state: 'available')
3. RSU Allocation → Update tickets to 'allocated' state
4. Sale → Create order with validation number, update tickets to 'sold'
5. Draw → Select winners from sold tickets

## Service Layer Responsibilities

The following logic should be implemented in the service layer, not database triggers:

- State transition validation
- Counter updates for allocations
- Cascade state changes (e.g., order → tickets)
- Validation number generation
- Critical event logging
- User reference synchronization with Cognito
- Ticket generation when raffle moves to 'configured' state

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
jackpot_starting_cents BIGINT DEFAULT 0
revenue_calculation_method VARCHAR(50) NOT NULL DEFAULT 'gross_revenue'
max_tickets INTEGER NOT NULL
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

### ticket_allocation_range

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
range_type VARCHAR(20) NOT NULL  -- rsu, online, reserve
range_start INTEGER NOT NULL
range_end INTEGER NOT NULL
default_allocation_size INTEGER NOT NULL DEFAULT 1
allocated INTEGER NOT NULL DEFAULT 0  -- running counter
available INTEGER NOT NULL  -- computed as (range_end - range_start + 1 - allocated)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

UNIQUE(raffle_event_id, range_type)
CONSTRAINT check_range_order CHECK (range_end >= range_start)
CONSTRAINT check_allocation CHECK (allocated <= (range_end - range_start + 1))
```

### prize_template

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id)
name VARCHAR(255) NOT NULL
description TEXT
prize_type VARCHAR(50) NOT NULL  -- fixed, percentage
value_cents BIGINT  -- for fixed prizes
percentage DECIMAL(5,2)  -- for percentage prizes
currency VARCHAR(3) NOT NULL DEFAULT 'USD'
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_prize_template_value CHECK (
    (prize_type = 'fixed' AND value_cents IS NOT NULL AND percentage IS NULL) OR
    (prize_type = 'percentage' AND percentage IS NOT NULL AND value_cents IS NULL)
)
```

### prize

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
name VARCHAR(255) NOT NULL
description TEXT
prize_type VARCHAR(50) NOT NULL  -- fixed, percentage
value_cents BIGINT  -- for fixed prizes
percentage DECIMAL(5,2)  -- for percentage prizes
currency VARCHAR(3) NOT NULL DEFAULT 'USD'
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
UNIQUE(raffle_event_id, position)
```

### ticket_package_template

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id)
name VARCHAR(255) NOT NULL
description TEXT
ticket_count INTEGER NOT NULL CHECK (ticket_count > 0)
price_cents BIGINT NOT NULL CHECK (price_cents >= 0)
currency VARCHAR(3) NOT NULL DEFAULT 'USD'
display_order INTEGER NOT NULL DEFAULT 0
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)
```

### ticket_package

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
name VARCHAR(255) NOT NULL
description TEXT
ticket_count INTEGER NOT NULL CHECK (ticket_count > 0)
price_cents BIGINT NOT NULL CHECK (price_cents >= 0)
currency VARCHAR(3) NOT NULL DEFAULT 'USD'
is_active BOOLEAN NOT NULL DEFAULT true
display_order INTEGER NOT NULL DEFAULT 0
active_from TIMESTAMPTZ  -- null if no dynamic pricing
active_to TIMESTAMPTZ    -- null if no dynamic pricing
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)
```

### ticket_package_perk

```sql
id UUID PRIMARY KEY
ticket_package_id UUID NOT NULL REFERENCES ticket_package(id)
perk_id UUID NOT NULL REFERENCES perk(id)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
UNIQUE(ticket_package_id, perk_id)
```

### perk

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id)
name VARCHAR(255) NOT NULL
description TEXT
perk_type VARCHAR(50) NOT NULL  -- benefit, physical_item, digital_item, experience
category VARCHAR(50)
requires_inventory BOOLEAN NOT NULL DEFAULT false
total_inventory INTEGER  -- null if unlimited
allocated_inventory INTEGER NOT NULL DEFAULT 0
claimed_inventory INTEGER NOT NULL DEFAULT 0
available_inventory INTEGER  -- computed as (total_inventory - allocated_inventory) if total_inventory is not null
is_active BOOLEAN NOT NULL DEFAULT true
metadata JSONB
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)
```

### perk_claim

```sql
id UUID PRIMARY KEY
perk_id UUID NOT NULL REFERENCES perk(id)
order_id UUID NOT NULL REFERENCES "order"(id)
claimed_at TIMESTAMPTZ
claimed_by UUID REFERENCES user_reference(id)
status VARCHAR(20) NOT NULL DEFAULT 'pending'  -- pending, claimed, expired
claim_code VARCHAR(50)
metadata JSONB
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

CONSTRAINT check_claim_status CHECK (status IN ('pending', 'claimed', 'expired'))
```

### rsu

```sql
id UUID PRIMARY KEY
organization_id UUID REFERENCES organization(id)  -- can be null for admin RSU
venue_id UUID REFERENCES venue(id)  -- can be null
device_identifier VARCHAR(255) UNIQUE NOT NULL  -- MAC address or unique ID
name VARCHAR(100) NOT NULL
description TEXT
state VARCHAR(50) NOT NULL DEFAULT 'registered'  -- registered, enrolled, active, closing, closed, reconciled, suspended, error
max_offline_tickets INTEGER NOT NULL DEFAULT 100
sync_interval_minutes INTEGER NOT NULL DEFAULT 5
last_sync_at TIMESTAMPTZ
last_validation_at TIMESTAMPTZ
validation_status VARCHAR(20) DEFAULT 'pending'  -- valid, invalid, pending
is_active BOOLEAN NOT NULL DEFAULT true
current_operator_id UUID REFERENCES user_reference(id)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_rsu_state CHECK (state IN ('registered', 'enrolled', 'active', 'closing', 'closed', 'reconciled', 'suspended', 'error'))
CONSTRAINT check_validation_status CHECK (validation_status IN ('valid', 'invalid', 'pending'))
```

### rsu_configuration

```sql
id UUID PRIMARY KEY
rsu_id UUID NOT NULL REFERENCES rsu(id)
printer_paper_size VARCHAR(20) DEFAULT '58mm'
printer_speed VARCHAR(20) DEFAULT 'normal'
configuration JSONB  -- additional settings
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### rsu_allocation

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
rsu_id UUID NOT NULL REFERENCES rsu(id)
range_type VARCHAR(20) NOT NULL DEFAULT 'rsu'  -- rsu, online, reserve
entity_type VARCHAR(20) NOT NULL DEFAULT 'rsu'  -- rsu, system, partner
start_number INTEGER NOT NULL
end_number INTEGER NOT NULL
total_tickets INTEGER NOT NULL  -- computed as (end_number - start_number + 1)
state VARCHAR(20) NOT NULL DEFAULT 'pending'  -- pending, active, exhausted, cancelled
tickets_sold INTEGER NOT NULL DEFAULT 0  -- counter for tracking (updated by service layer)
tickets_voided INTEGER NOT NULL DEFAULT 0  -- counter for tracking (updated by service layer)
tickets_available INTEGER NOT NULL  -- computed as (total_tickets - tickets_sold - tickets_voided)
allocated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
allocated_by UUID REFERENCES user_reference(id)
exhausted_at TIMESTAMPTZ
metadata JSONB
is_active BOOLEAN NOT NULL DEFAULT true

UNIQUE(raffle_event_id, start_number, end_number)
CONSTRAINT check_number_range CHECK (end_number >= start_number)
CONSTRAINT check_allocation_counts CHECK (tickets_sold + tickets_voided <= total_tickets)
CONSTRAINT check_allocation_state CHECK (state IN ('pending', 'active', 'exhausted', 'cancelled'))
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

### "order"  -- using quotes because 'order' is a reserved word

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
customer_id UUID REFERENCES customer(id)
ticket_package_id UUID NOT NULL REFERENCES ticket_package(id)
ticket_count INTEGER NOT NULL
total_amount_cents BIGINT NOT NULL
currency VARCHAR(3) NOT NULL DEFAULT 'USD'
validation_number VARCHAR(100) UNIQUE NOT NULL  -- Generated by service layer on completion
payment_status VARCHAR(50) NOT NULL DEFAULT 'pending'  -- pending, completed, failed, refunded
payment_reference VARCHAR(255)  -- Stripe payment ID, etc.
source VARCHAR(50) NOT NULL  -- online, rsu, admin
rsu_id UUID REFERENCES rsu(id)  -- if sold via RSU
operator_id UUID REFERENCES user_reference(id)  -- who processed if RSU sale
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
order_id UUID REFERENCES "order"(id)
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
CONSTRAINT check_allocated_data CHECK (
    (state != 'allocated' OR (allocated_to_rsu_id IS NOT NULL AND allocation_id IS NOT NULL))
)
```

### reconciliation

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id) UNIQUE
state VARCHAR(20) NOT NULL DEFAULT 'in_progress'  -- in_progress, completed, failed
total_tickets_sold INTEGER
total_revenue_cents BIGINT
currency VARCHAR(3) NOT NULL DEFAULT 'USD'
started_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
confirmed_at TIMESTAMPTZ
confirmed_by UUID REFERENCES user_reference(id)
notes TEXT
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

CONSTRAINT check_reconciliation_state CHECK (state IN ('in_progress', 'completed', 'failed'))
```

### reconciliation_rsu_status

```sql
id UUID PRIMARY KEY
reconciliation_id UUID NOT NULL REFERENCES reconciliation(id)
rsu_id UUID NOT NULL REFERENCES rsu(id)
status VARCHAR(20) NOT NULL DEFAULT 'pending'  -- pending, syncing, synced, error
tickets_sold INTEGER
tickets_voided INTEGER
last_sync_at TIMESTAMPTZ
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

CONSTRAINT check_rsu_status CHECK (status IN ('pending', 'syncing', 'synced', 'error'))
```

### reconciliation_discrepancy

```sql
id UUID PRIMARY KEY
reconciliation_id UUID NOT NULL REFERENCES reconciliation(id)
discrepancy_type VARCHAR(50) NOT NULL  -- ticket_count_mismatch, revenue_mismatch, etc.
rsu_id UUID REFERENCES rsu(id)
expected_value VARCHAR(255)
actual_value VARCHAR(255)
resolution VARCHAR(50)  -- manual_adjustment, system_correction, ignored
resolved_by UUID REFERENCES user_reference(id)
resolved_at TIMESTAMPTZ
notes TEXT
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### draw_execution

```sql
id UUID PRIMARY KEY
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id) UNIQUE
state VARCHAR(50) NOT NULL DEFAULT 'pending'  -- pending, in_progress, completed, failed
started_at TIMESTAMPTZ
rng_seed VARCHAR(255)  -- for audit trail
conducted_by UUID REFERENCES user_reference(id)
system_checksum VARCHAR(255)  -- System state at draw time
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_draw_state CHECK (state IN ('pending', 'in_progress', 'completed', 'failed'))
```

### draw_result

```sql
id UUID PRIMARY KEY
draw_execution_id UUID NOT NULL REFERENCES draw_execution(id)
prize_id UUID NOT NULL REFERENCES prize(id)
position INTEGER NOT NULL
winning_ticket_id UUID NOT NULL REFERENCES ticket(id)
winning_number INTEGER NOT NULL
prize_value_cents BIGINT NOT NULL  -- actual value at time of draw (calculated for percentage prizes)
selected_at TIMESTAMPTZ NOT NULL
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### prize_claim

```sql
id UUID PRIMARY KEY
draw_result_id UUID NOT NULL REFERENCES draw_result(id) UNIQUE
claimed_at TIMESTAMPTZ NOT NULL
claimed_by UUID REFERENCES user_reference(id)
verification_method VARCHAR(50) NOT NULL  -- validation_number, id_verification
verified_by UUID NOT NULL REFERENCES user_reference(id)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### critical_event

```sql
id UUID PRIMARY KEY
event_type VARCHAR(100) NOT NULL
raffle_event_id UUID REFERENCES raffle_event(id)
rsu_id UUID REFERENCES rsu(id)
user_id UUID REFERENCES user_reference(id)
severity VARCHAR(20) NOT NULL DEFAULT 'INFO'  -- INFO, WARNING, ERROR, CRITICAL
description TEXT NOT NULL
metadata JSONB
timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
ttl INTEGER  -- for DynamoDB TTL if synced

CONSTRAINT check_severity CHECK (severity IN ('INFO', 'WARNING', 'ERROR', 'CRITICAL'))
```

### verification

```sql
id UUID PRIMARY KEY
target_type VARCHAR(20) NOT NULL  -- system, raffle, rsu
target_id UUID  -- null for system-wide
verification_type VARCHAR(20) NOT NULL  -- scheduled, manual, pre_operation
checksums JSONB NOT NULL
status VARCHAR(20) NOT NULL  -- valid, invalid
performed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
performed_by VARCHAR(50) NOT NULL  -- 'system' or user ID
details TEXT
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

CONSTRAINT check_target_type CHECK (target_type IN ('system', 'raffle', 'rsu'))
CONSTRAINT check_verification_type CHECK (verification_type IN ('scheduled', 'manual', 'pre_operation'))
CONSTRAINT check_status CHECK (status IN ('valid', 'invalid'))
```

### system_configuration

```sql
id UUID PRIMARY KEY
version VARCHAR(20) NOT NULL
effective_from TIMESTAMPTZ NOT NULL
effective_to TIMESTAMPTZ
settings JSONB NOT NULL
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
```

### organization_configuration

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id)
version VARCHAR(20) NOT NULL
effective_from TIMESTAMPTZ NOT NULL
effective_to TIMESTAMPTZ
settings JSONB NOT NULL
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
```

### address

```sql
id UUID PRIMARY KEY
street VARCHAR(500) NOT NULL
city VARCHAR(255) NOT NULL
state VARCHAR(100) NOT NULL
zip_code VARCHAR(20) NOT NULL
country VARCHAR(2) NOT NULL
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### contact

```sql
id UUID PRIMARY KEY
name VARCHAR(255) NOT NULL
email VARCHAR(255)
phone VARCHAR(50)
address_id UUID REFERENCES address(id)
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
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
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)
```

## Jurisdiction Tables

### jurisdiction_regulation

```sql
id UUID PRIMARY KEY
jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id)
name VARCHAR(255) NOT NULL
regulation_type VARCHAR(50) NOT NULL  -- prize_limit, ticket_price_limit, event_duration, sales_method, reporting_requirement, data_retention
value JSONB NOT NULL
effective_date TIMESTAMPTZ NOT NULL
expiration_date TIMESTAMPTZ
overridable BOOLEAN NOT NULL DEFAULT false
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

CONSTRAINT check_regulation_type CHECK (regulation_type IN ('prize_limit', 'ticket_price_limit', 'event_duration', 'sales_method', 'reporting_requirement', 'data_retention'))
```

### jurisdiction

```sql
id UUID PRIMARY KEY
name VARCHAR(255) NOT NULL
code VARCHAR(50) NOT NULL UNIQUE  -- e.g., "CA-US", "SF-CA-US"
jurisdiction_type VARCHAR(50) NOT NULL  -- state, province, country, local
parent_jurisdiction_id UUID REFERENCES jurisdiction(id)
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)

CONSTRAINT check_jurisdiction_type CHECK (jurisdiction_type IN ('state', 'province', 'country', 'local'))
```

## Venue Tables

### venue

```sql
id UUID PRIMARY KEY
jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id)
name VARCHAR(255) NOT NULL
description TEXT
address_id UUID REFERENCES address(id)
capacity INTEGER
amenities TEXT[]  -- Array of amenities
timezone VARCHAR(50) NOT NULL  -- e.g., 'America/Los_Angeles'
accepted_currencies VARCHAR(3)[] NOT NULL DEFAULT ARRAY['USD']
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)
```

## Organization Tables

### organization

```sql
id UUID PRIMARY KEY
name VARCHAR(255) NOT NULL
description TEXT
license_number VARCHAR(100)
primary_currency VARCHAR(3) NOT NULL DEFAULT 'USD'
contact_email VARCHAR(255)
contact_phone VARCHAR(50)
address_id UUID REFERENCES address(id)
default_revenue_calculation_method VARCHAR(50) DEFAULT 'gross_revenue'
tax_rate DECIMAL(5,2)
tax_included BOOLEAN NOT NULL DEFAULT false
default_venue_id UUID REFERENCES venue(id)
is_active BOOLEAN NOT NULL DEFAULT true
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_by UUID REFERENCES user_reference(id)
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_by UUID REFERENCES user_reference(id)
```

### organization_venue

```sql
id UUID PRIMARY KEY
organization_id UUID NOT NULL REFERENCES organization(id) ON DELETE CASCADE
venue_id UUID NOT NULL REFERENCES venue(id) ON DELETE CASCADE
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
UNIQUE(organization_id, venue_id)
```

### user_organization

```sql
id UUID PRIMARY KEY
user_id UUID NOT NULL REFERENCES user_reference(id)
organization_id UUID NOT NULL REFERENCES organization(id)
role VARCHAR(50) NOT NULL  -- admin, operator
is_default BOOLEAN NOT NULL DEFAULT false
joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

UNIQUE(user_id, organization_id)
CONSTRAINT check_role CHECK (role IN ('admin', 'operator'))
```

## Join Tables

### raffle_prize

```sql
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
prize_id UUID NOT NULL REFERENCES prize(id)
PRIMARY KEY (raffle_event_id, prize_id)
```

### raffle_ticket_package

```sql
raffle_event_id UUID NOT NULL REFERENCES raffle_event(id)
ticket_package_id UUID NOT NULL REFERENCES ticket_package(id)
PRIMARY KEY (raffle_event_id, ticket_package_id)
```

## Entity Relationships

```
Ticket Generation & Allocation:
- When raffle moves to 'configured', all tickets are pre-generated
- Tickets are created with sequential draw numbers (1 to max_tickets)
- Initial state is 'available' for all tickets
- ticket_allocation_range tracks the configured ranges for RSU/online/reserve

RSU Allocation Process:
- RSU requests allocation from their designated range
- Service updates tickets in that range from 'available' to 'allocated'
- Tickets are linked to the RSU via allocated_to_rsu_id and allocation_id

Online Sales:
- Pull from online range where state = 'available'
- Direct transition from 'available' to 'sold'

State Transitions:
- Raffle: created → configured → active → sales_closed → reconciling → reconciled → drawing_ready → completed
- RSU: registered → enrolled → active → closing → closed → reconciled
- Ticket: available → allocated → sold → voided | sold → winner → claimed
- Order: pending → completed → refunded

Key Relationships:
- One order → multiple tickets (all share same validation number)
- RSU → multiple allocations across different raffles
- Raffle → multiple ticket allocation ranges
- Organization → multiple venues (many-to-many)
- User → multiple organizations with roles
```

## Key Indexes

```sql
-- Performance indexes for common queries
CREATE INDEX idx_ticket_raffle_draw ON ticket(raffle_event_id, draw_number);
CREATE INDEX idx_ticket_raffle_state ON ticket(raffle_event_id, state);
CREATE INDEX idx_ticket_order ON ticket(order_id);
CREATE INDEX idx_ticket_allocation ON ticket(allocation_id);
CREATE INDEX idx_ticket_available ON ticket(raffle_event_id, state) WHERE state = 'available';

CREATE INDEX idx_order_validation ON "order"(validation_number);
CREATE INDEX idx_order_raffle ON "order"(raffle_event_id);
CREATE INDEX idx_order_payment_status ON "order"(payment_status);

CREATE INDEX idx_rsu_allocation_raffle ON rsu_allocation(raffle_event_id);
CREATE INDEX idx_rsu_allocation_active ON rsu_allocation(is_active) WHERE is_active = true;

CREATE INDEX idx_critical_event_type ON critical_event(event_type);
CREATE INDEX idx_critical_event_timestamp ON critical_event(timestamp);

-- Jurisdiction
CREATE INDEX idx_jurisdiction_code ON jurisdiction(code);
CREATE INDEX idx_jurisdiction_parent ON jurisdiction(parent_jurisdiction_id);
CREATE INDEX idx_jurisdiction_active ON jurisdiction(is_active);

-- Venue
CREATE INDEX idx_venue_jurisdiction ON venue(jurisdiction_id);
CREATE INDEX idx_venue_active ON venue(is_active);

-- Organization
CREATE INDEX idx_organization_active ON organization(is_active);

-- Draw
CREATE INDEX idx_draw_result_draw ON draw_result(draw_execution_id);
CREATE INDEX idx_draw_result_ticket ON draw_result(winning_ticket_id);
```

## Data Integrity Rules

- Ticket Generation: All tickets created when raffle state changes to 'configured'
- Unique Draw Numbers: Database enforces one ticket per draw number per raffle
- Allocation Boundaries: RSUs can only sell tickets within their allocated range
- Validation Uniqueness: Each order has a globally unique validation number
- State Consistency: Ticket state changes follow defined transitions
- Reconciliation: Cannot draw until raffle is reconciled (confirmed by admin)
- User References: All user operations require valid user_reference record
- Organization/Venue Active: Raffle can only be activated if both are active

## Ticket Generation Process

When a raffle transitions from 'created' to 'configured':

```sql
-- Example: Generate 200,000 tickets
INSERT INTO ticket (id, raffle_event_id, draw_number, state)
SELECT
    gen_random_uuid(),
    '123e4567-e89b-12d3-a456-426614174000',
    generate_series(1, 200000),
    'available'
;

-- This creates tickets numbered 1-200,000 all in 'available' state
```

## RSU Allocation Process

When RSU is allocated a range:

```sql
-- Update tickets in the allocated range
UPDATE ticket
SET state = 'allocated',
    allocated_to_rsu_id = 'rsu-456f7890-b12c-34d5-e678-901234567abc',
    allocation_id = 'range-123e4567-e89b-12d3-a456-426614174000',
    allocated_at = NOW()
WHERE raffle_event_id = '123e4567-e89b-12d3-a456-426614174000'
  AND draw_number BETWEEN 1001 AND 1100
  AND state = 'available';
```

## Notes on Pre-generation Strategy

1. **Performance Consideration**: Pre-generating tickets creates a large number of records upfront
2. **Benefits**: Guarantees no duplicate tickets, simplifies allocation logic
3. **Storage**: For 200,000 tickets, expect ~20MB of data per raffle
4. **Batch Operations**: Use batch inserts and updates for efficiency
5. **Indexing**: Critical to have proper indexes on raffle_event_id + state for allocation queries
