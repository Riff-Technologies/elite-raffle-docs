# Elite Raffle System – MVP Database Schema

> **Database:** PostgreSQL 15 (or higher)
>
> **Encoding / Collation:** `UTF8` / `en_US.UTF8`
>
> This schema is designed to cover the MVP functionality described in `models.md` and the OpenAPI specification. It emphasises:
>
> 1. Data integrity with **foreign-key constraints** & `CHECK` clauses
> 2. **Extensibility** via lookup tables and `JSONB` columns
> 3. Full **auditability** to satisfy GLI-31 compliance
> 4. Indices tuned for the access patterns implied by the API endpoints.

---

## 1 – Extensions

```sql
-- Required to generate UUID PKs without external libraries
CREATE EXTENSION IF NOT EXISTS "pgcrypto"; -- Provides gen_random_uuid()
```

---

## 2 – Lookup / Reference Tables

Lookup tables are preferred over native `ENUM`s for easier modification at runtime.

```sql
-- Event states (created ➜ active ➜ suspended ➜ closed ➜ cancelled)
CREATE TABLE event_states (
    key              TEXT PRIMARY KEY,
    description      TEXT NOT NULL
);

-- Ticket states
CREATE TABLE ticket_states (
    key              TEXT PRIMARY KEY,
    description      TEXT NOT NULL
);

-- Order payment statuses
CREATE TABLE payment_statuses (
    key              TEXT PRIMARY KEY,
    description      TEXT NOT NULL
);

-- Order sources (online | rsu)
CREATE TABLE order_sources (
    key              TEXT PRIMARY KEY,
    description      TEXT NOT NULL
);

-- RSU states
CREATE TABLE rsu_states (
    key              TEXT PRIMARY KEY,
    description      TEXT NOT NULL
);

-- Offline-sync statuses (pending | synced | failed)
CREATE TABLE offline_sync_statuses (
    key              TEXT PRIMARY KEY,
    description      TEXT NOT NULL
);
```

Populate these with seed data during migrations.

---

## 3 – Core Reference Entities

```sql
-- Organizations -------------------------------------------------------
CREATE TABLE organizations (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name             TEXT    NOT NULL,
    description      TEXT,
    license_number   TEXT,
    primary_currency CHAR(3) NOT NULL,
    settings         JSONB   DEFAULT '{}'::JSONB, -- RSU & tax settings, etc.
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);

-- Users ---------------------------------------------------------------
CREATE TABLE users (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cognito_sub      TEXT UNIQUE NOT NULL,
    display_name     TEXT NOT NULL,
    last_login       TIMESTAMPTZ,
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- RBAC Tables ---------------------------------------------------------
-- Roles
CREATE TABLE roles (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name             TEXT NOT NULL,
    description      TEXT,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Permissions
CREATE TABLE permissions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name             TEXT NOT NULL,
    description      TEXT,
    resource_type    TEXT NOT NULL,  -- raffle, rsu, organization, etc.
    action           TEXT NOT NULL,  -- create, read, update, delete, void, reconcile, etc.
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(resource_type, action)
);

-- Role-Permission mapping
CREATE TABLE role_permissions (
    role_id          UUID NOT NULL REFERENCES roles(id),
    permission_id    UUID NOT NULL REFERENCES permissions(id),
    PRIMARY KEY (role_id, permission_id),
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- User ↔ Organization (many-to-many with role)
CREATE TABLE organization_users (
    organization_id  UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id          UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id          UUID NOT NULL REFERENCES roles(id),
    is_default       BOOLEAN DEFAULT FALSE,
    joined_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (organization_id, user_id)
);

-- Addresses -----------------------------------------------------------
CREATE TABLE addresses (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    street     TEXT,
    city       TEXT,
    state      TEXT,
    zip_code   TEXT,
    country    CHAR(2)
);

-- Venues --------------------------------------------------------------
CREATE TABLE venues (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id  UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name             TEXT NOT NULL,
    description      TEXT,
    address_id       UUID REFERENCES addresses(id),
    capacity         INTEGER,
    timezone         TEXT,
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);
```

### Indexes

```sql
CREATE INDEX idx_venues_org ON venues(organization_id);
CREATE INDEX idx_users_last_login ON users(last_login);
CREATE INDEX idx_role_permissions_role ON role_permissions(role_id);
CREATE INDEX idx_org_users_role ON organization_users(organization_id, role_id);
```

---

## 4 – Events & Related Configuration

```sql
-- Events -------------------------------------------------------
CREATE TABLE events (
    id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                     TEXT NOT NULL,
    description              TEXT,
    organization_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    venue_id                 UUID REFERENCES venues(id),
    state                    TEXT NOT NULL REFERENCES event_states(key),
    configuration            JSONB NOT NULL,   -- sales timings, jackpot, etc.
    ticket_allocation        JSONB NOT NULL,   -- ranges & counters
    max_tickets              INTEGER NOT NULL, -- maximum number of tickets that can be sold
    reconciliation_confirmed_at TIMESTAMPTZ,
    reconciliation_confirmed_by UUID,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by               UUID,
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by               UUID
);

-- Ticket Package Templates -------------------------------------------
CREATE TABLE ticket_package_templates (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id  UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name             TEXT NOT NULL,
    description      TEXT,
    ticket_count     INTEGER NOT NULL CHECK (ticket_count > 0),
    price            INTEGER NOT NULL CHECK (price >= 0), -- lowest currency unit
    currency         CHAR(3) NOT NULL,
    display_order    INTEGER,
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);

-- Ticket Packages (per event) ----------------------------------------
CREATE TABLE ticket_packages (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id         UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    name             TEXT NOT NULL,
    description      TEXT,
    ticket_count     INTEGER NOT NULL CHECK (ticket_count > 0),
    price            INTEGER NOT NULL CHECK (price >= 0),
    currency         CHAR(3) NOT NULL,
    display_order    INTEGER,
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    active_from      TIMESTAMPTZ,
    active_to        TIMESTAMPTZ,
    perk_ids         UUID[] DEFAULT '{}', -- Array of perk UUIDs (future-proof)
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);

-- Prize Templates ----------------------------------------------------
CREATE TABLE prize_templates (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id  UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name             TEXT NOT NULL,
    description      TEXT,
    type             TEXT NOT NULL,   -- fixed | percentage | …
    value            NUMERIC(14,2),   -- nullable for percentage
    percentage       NUMERIC(5,2),    -- nullable for fixed
    currency         CHAR(3),
    metadata         JSONB DEFAULT '{}'::JSONB,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);

-- Prizes (per event) -------------------------------------------------
CREATE TABLE prizes (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id         UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    name             TEXT NOT NULL,
    description      TEXT,
    type             TEXT NOT NULL,
    value            NUMERIC(14,2),   -- nullable for percentage
    percentage       NUMERIC(5,2) CHECK (percentage IS NULL OR (percentage >= 0 AND percentage <= 100)),
    currency         CHAR(3),
    position         SMALLINT NOT NULL,
    winner_count     SMALLINT NOT NULL DEFAULT 1,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);
```

### Indexes

```sql
CREATE INDEX idx_events_org ON events(organization_id);
CREATE INDEX idx_events_state ON events(state);
CREATE INDEX idx_prizes_event_pos ON prizes(event_id, position);
CREATE INDEX idx_event_config_gin ON events USING gin (configuration jsonb_path_ops);
```

---

## 5 – Sales & Ticketing

```sql
-- RSUs (Retail Sales Units) ------------------------------------------
CREATE TABLE rsus (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id  UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    venue_id         UUID REFERENCES venues(id),
    name             TEXT NOT NULL,
    description      TEXT,
    device_identifier TEXT UNIQUE NOT NULL,   -- e.g., MAC address
    state            TEXT NOT NULL REFERENCES rsu_states(key),
    configuration    JSONB NOT NULL DEFAULT '{}'::JSONB,
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    current_operator_id UUID,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);

-- RSU Allocations ----------------------------------------------------
CREATE TABLE rsu_allocations (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id         UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    rsu_id           UUID NOT NULL REFERENCES rsus(id) ON DELETE CASCADE,
    start_number     INTEGER NOT NULL,
    end_number       INTEGER NOT NULL,
    next_draw_number INTEGER NOT NULL,
    total_tickets    INTEGER NOT NULL,
    state            TEXT NOT NULL, -- active | exhausted | released
    tickets_sold     INTEGER NOT NULL DEFAULT 0,
    tickets_voided   INTEGER NOT NULL DEFAULT 0,
    tickets_available INTEGER NOT NULL,
    allocated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    allocated_by     UUID,
    exhausted_at     TIMESTAMPTZ,
    metadata         JSONB DEFAULT '{}'::JSONB
);


-- Orders -------------------------------------------------------------
CREATE TABLE orders (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    validation_number TEXT UNIQUE NOT NULL,
    event_id         UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    package_id       UUID REFERENCES ticket_packages(id),
    rsu_id           UUID REFERENCES rsus(id),
    rsu_allocation_id UUID REFERENCES rsu_allocations(id),
    total_amount     INTEGER NOT NULL CHECK (total_amount >= 0),
    currency         CHAR(3) NOT NULL,
    payment_status   TEXT NOT NULL REFERENCES payment_statuses(key),
    payment_reference TEXT,
    ticket_count     INTEGER NOT NULL CHECK (ticket_count > 0),
    customer_info    JSONB,
    source           TEXT NOT NULL REFERENCES order_sources(key),
    operator_id      UUID, -- nullable for online sales
    is_offline_sale  BOOLEAN NOT NULL DEFAULT FALSE,
    sold_at          TIMESTAMPTZ NOT NULL, -- timestamp when the order was sold
    sold_by          UUID,  -- null for online sales
    voided_at        TIMESTAMPTZ, -- timestamp when the order was voided, if applicable
    voided_by        UUID, -- user who voided the order, if applicable
    voided_reason    TEXT, -- reason for voiding the order, if applicable
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);

-- Tickets ------------------------------------------------------------
CREATE TABLE tickets (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id         UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    draw_number      INTEGER NOT NULL,
    state            TEXT NOT NULL REFERENCES ticket_states(key),
    order_id         UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(event_id, draw_number) -- ensures unique draw numbers per event
);
```

### Indexes

```sql
CREATE INDEX idx_orders_event ON orders(event_id);
CREATE INDEX idx_orders_payment_status ON orders(payment_status);
CREATE INDEX idx_tickets_event_state ON tickets(event_id, state);
CREATE INDEX idx_tickets_order ON tickets(order_id);
CREATE INDEX idx_rsu_alloc_event ON rsu_allocations(event_id);
CREATE INDEX idx_rsu_allocation_rsu ON rsu_allocations(rsu_id);
CREATE INDEX idx_rsu_allocation_state ON rsu_allocations(state);
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_orders_event_status ON orders(event_id, payment_status);
CREATE INDEX idx_orders_sold_at ON orders(sold_at);
```

---

## 6 – Draws & Claims

```sql
-- Draw Executions ----------------------------------------------------
CREATE TABLE draw_executions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id         UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    state            TEXT NOT NULL, -- pending | in_progress | completed | failed
    started_at       TIMESTAMPTZ NOT NULL,
    rng_seed         TEXT NOT NULL,
    conducted_by     UUID NOT NULL,
    system_checksum  TEXT,  -- Nullable to accommodate external logging systems
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Draw Results -------------------------------------------------------
CREATE TABLE draw_results (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    draw_execution_id UUID NOT NULL REFERENCES draw_executions(id) ON DELETE CASCADE,
    prize_id         UUID NOT NULL REFERENCES prizes(id) ON DELETE CASCADE,
    position         SMALLINT NOT NULL,
    winning_ticket_id UUID NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    winning_number   INTEGER NOT NULL,
    prize_value_at_draw INTEGER NOT NULL, -- stored in lowest currency unit
    selected_at      TIMESTAMPTZ NOT NULL
);

-- Prize Claims -------------------------------------------------------
CREATE TABLE prize_claims (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    draw_result_id   UUID NOT NULL REFERENCES draw_results(id) ON DELETE CASCADE,
    claimed_at       TIMESTAMPTZ NOT NULL,
    claimed_by       UUID NOT NULL,      -- winner id / user reference
    verification_method TEXT NOT NULL,   -- validation_number | manual | …
    verified_by      UUID,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Indexes

```sql
CREATE INDEX idx_draw_results_execution ON draw_results(draw_execution_id);
CREATE INDEX idx_prize_claims_result ON prize_claims(draw_result_id);
CREATE INDEX idx_draw_result_ticket ON draw_results(winning_ticket_id);
```

---

## 8 – Configuration Management

> Note: System and organization configurations are now managed outside the database through AWS SSM Parameter Store. This provides better integration with AWS services, simplified access control, and version history management without requiring database tables.

### Indexes

```sql
CREATE INDEX idx_organization_settings_gin ON organizations USING gin (settings jsonb_path_ops);
```

---

## 9 – Audit Trail

Critical tables employ lightweight `created_* / updated_*` columns for basic change tracking. Comprehensive audit logging is now handled by the API applications and stored in CloudWatch for better integration with AWS services and improved separation of concerns.

---

## 10 – Verification

> Note: Verification tracking is now handled by the API applications rather than being stored in the database. This provides better separation of concerns, leverages AWS services for verification tracking, and allows for more sophisticated monitoring and alerting without requiring a dedicated database table.

---

## 11 – Seed Data Examples (optional)

Below is **illustrative** seed data for lookup tables. Load via migrations, _not_ in production scripts.

```sql
-- Event states
INSERT INTO event_states(key, description) VALUES
  ('created',   'Initial state after creation'),
  ('configured', 'All required parameters set'),
  ('active',    'Sales are open'),
  ('suspended', 'Sales temporarily paused'),
  ('sales_closed', 'Sales closed'),
  ('reconciling', 'Awaiting RSU synchronization'),
  ('reconciled', 'Admin confirmed all sales accounted for'),
  ('drawing_ready', 'Ready for draw to begin'),
  ('completed', 'Draw completed, winners determined'),
  ('cancelled', 'Event cancelled');

-- Default roles
INSERT INTO roles(id, name, description) VALUES
  (gen_random_uuid(), 'System Administrator', 'Full access to all system functions'),
  (gen_random_uuid(), 'Organization Administrator', 'Full access within their organization'),
  (gen_random_uuid(), 'Raffle Manager', 'Can create and manage raffles, but not system settings'),
  (gen_random_uuid(), 'RSU Operator', 'Can sell tickets and manage assigned RSUs'),
  (gen_random_uuid(), 'Viewer', 'Read-only access to reports and raffle information');
```

_(Add similar inserts for the other lookup tables.)_

---

### Future-Proofing Notes

1. Use `JSONB` columns (`configuration`, `settings`, `metadata`) to add optional fields without migrations.
2. Reference tables avoid the rigidity of SQL `ENUM`s.
3. All monetary amounts are `INTEGER` in lowest denomination – rename to `*_cents` if desired for clarity.
4. Time-series data (e.g., ticket sales history) can be extracted into append-only tables later to manage OLTP vs. analytics workloads.
5. Partitioning (e.g., by `event_id` for `tickets`) can be introduced when volume dictates, without affecting the logical model.
6. The RBAC system provides fine-grained permission management through roles and permissions.
7. **Table Naming Consistency**: The schema uses consistent naming conventions, with the generic `events` table replacing the more specific `raffle_events`. This approach allows for future expansion to other types of events without schema changes.
8. **Ticket Status at Order Level**: Ticket status tracking fields (`sold_at`, `sold_by`, `voided_at`, `voided_by`, `voided_reason`) are stored at the order level rather than the ticket level to better reflect the business process where entire orders are sold or voided as a unit. This design change improves data consistency and reduces redundancy since these fields apply to the entire order transaction rather than individual tickets.
9. **Nullable System Checksum**: The `system_checksum` field in the `draw_executions` table is nullable to accommodate external logging systems like CloudWatch. This allows for flexibility in how system integrity is verified, with checksums either stored in the database or managed by external AWS services.
10. **On-Device Validation Number Generation**: Validation numbers are now generated directly by RSU devices without requiring pre-allocated blocks. Each RSU creates globally unique validation numbers on-device, eliminating the need for the previously used `validation_number_blocks` table. This approach simplifies the system architecture and reduces database complexity while maintaining the uniqueness guarantees required for validation numbers.
11. **External Event Logging**: Critical events are now logged to CloudWatch instead of being stored in the database. This provides better separation of concerns, leverages AWS services for logging, and allows for more sophisticated monitoring, alerting, and retention policies.
12. **AWS Parameter Store for Configuration**: System and organization configurations are now managed through AWS SSM Parameter Store instead of database tables. This approach provides better integration with AWS services, simplified access control, automatic encryption, and built-in version history management without requiring custom database tables and versioning logic.
13. **CloudWatch for Audit Logging**: Audit logging is now handled by the API applications and stored in CloudWatch rather than using database triggers and an audit_log table. This approach provides better separation of concerns, leverages AWS services for comprehensive logging, and allows for more sophisticated monitoring, alerting, and retention policies without adding overhead to database operations.
14. **External Verification Tracking**: Verification tracking is now handled by the API applications rather than being stored in the database. This provides better separation of concerns and leverages AWS services for verification tracking.
15. **Dynamic Ticket Count Calculation**: Total tickets sold is calculated dynamically rather than stored as a redundant counter in the events table. This calculation can be performed using a simple SQL query that counts tickets with a valid state for a given event:

    ```sql
    SELECT COUNT(*) FROM tickets
    WHERE event_id = :event_id
    AND state = 'active';
    ```

    This approach ensures data consistency by deriving the count from the actual ticket records rather than maintaining a separate counter that could become out of sync.

16. **Dynamic Jackpot Calculation**: Jackpot amounts are calculated dynamically based on ticket sales and prize configurations rather than being stored as a static value. For percentage-based prizes, the jackpot can be calculated as:

    ```sql
    SELECT
      p.id AS prize_id,
      p.name AS prize_name,
      p.percentage,
      (SELECT COUNT(*) FROM tickets t WHERE t.event_id = e.id AND t.state = 'active') AS tickets_sold,
      (SELECT COUNT(*) FROM tickets t WHERE t.event_id = e.id AND t.state = 'active') *
        (SELECT price FROM ticket_packages WHERE id = o.package_id) * (p.percentage / 100.0) AS jackpot_amount
    FROM
      events e
      JOIN prizes p ON p.event_id = e.id
      JOIN orders o ON o.event_id = e.id
    WHERE
      e.id = :event_id
      AND p.type = 'percentage'
    GROUP BY
      p.id, p.name, p.percentage, e.id;
    ```

    This dynamic calculation ensures that the jackpot amount always reflects the current ticket sales and prize configuration, eliminating the risk of inconsistencies that could occur with a stored value.

17. **Maximum Tickets per Event**: The `max_tickets` field in the `events` table defines the total number of tickets that can be sold for an event, providing a clear upper bound for ticket allocation and sales. This replaces the need for ticket count fields in package tables, as the system now uses the event-level maximum to control ticket allocation.

18. **Simplified Ticket Package Structure**: The ticket package structure has been simplified by removing the `ticket_count` field from both `ticket_package_templates` and `ticket_packages` tables. This change aligns with how ticket allocation actually works in the system, where the maximum number of tickets is defined at the event level rather than the package level.

---

> **End of schema.**
