# Electronic Raffle System - Database Schema

## Core Tables

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

## Table Creation Order
1. address
2. contact
3. license
4. jurisdiction_regulation
5. jurisdiction
6. venue
7. organization_setting
8. organization
9. organization_venue
10. entity_attribute

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