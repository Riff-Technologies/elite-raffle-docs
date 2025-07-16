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
-- Raffle event states (created ➜ active ➜ suspended ➜ closed ➜ cancelled)
CREATE TABLE raffle_event_states (
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

-- User ↔ Organization (many-to-many with role)
CREATE TABLE organization_users (
    organization_id  UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id          UUID NOT NULL REFERENCES users(id)         ON DELETE CASCADE,
    role             TEXT  NOT NULL,           -- admin | operator | …
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
CREATE INDEX idx_venues_org   ON venues(organization_id);
CREATE INDEX idx_users_last_login ON users(last_login);
```

---

## 4 – Raffle Events & Related Configuration

```sql
-- Raffle Events -------------------------------------------------------
CREATE TABLE raffle_events (
    id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                     TEXT NOT NULL,
    description              TEXT,
    organization_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    venue_id                 UUID REFERENCES venues(id),
    event_identifier         TEXT UNIQUE NOT NULL,
    state                    TEXT NOT NULL REFERENCES raffle_event_states(key),
    configuration            JSONB NOT NULL,   -- sales timings, jackpot, etc.
    ticket_allocation        JSONB NOT NULL,   -- ranges & counters
    total_sold               INTEGER NOT NULL DEFAULT 0,
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
    event_id         UUID NOT NULL REFERENCES raffle_events(id) ON DELETE CASCADE,
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
    event_id         UUID NOT NULL REFERENCES raffle_events(id) ON DELETE CASCADE,
    name             TEXT NOT NULL,
    description      TEXT,
    type             TEXT NOT NULL,
    value            NUMERIC(14,2),   -- nullable for percentage
    percentage       NUMERIC(5,2),
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
CREATE INDEX idx_raffle_events_org   ON raffle_events(organization_id);
CREATE INDEX idx_raffle_events_state ON raffle_events(state);
CREATE INDEX idx_prizes_event_pos    ON prizes(event_id, position);
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
    raffle_event_id  UUID NOT NULL REFERENCES raffle_events(id) ON DELETE CASCADE,
    rsu_id           UUID NOT NULL REFERENCES rsus(id)          ON DELETE CASCADE,
    start_number     INTEGER NOT NULL,
    end_number       INTEGER NOT NULL,
    next_draw_number INTEGER NOT NULL,
    total_tickets    INTEGER NOT NULL,
    state            TEXT NOT NULL, -- active | exhausted | released
    tickets_sold     INTEGER NOT NULL DEFAULT 0,
    tickets_voided   INTEGER NOT NULL DEFAULT 0,
    tickets_available INTEGER NOT NULL,
    validation_number_block JSONB NOT NULL, -- holds block info (future split)
    allocated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    allocated_by     UUID,
    exhausted_at     TIMESTAMPTZ,
    metadata         JSONB DEFAULT '{}'::JSONB
);

-- Validation Number Blocks ------------------------------------------
CREATE TABLE validation_number_blocks (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rsu_allocation_id UUID NOT NULL REFERENCES rsu_allocations(id) ON DELETE CASCADE,
    rsu_id           UUID NOT NULL REFERENCES rsus(id)          ON DELETE CASCADE,
    raffle_event_id  UUID NOT NULL REFERENCES raffle_events(id) ON DELETE CASCADE,
    block_start      TEXT NOT NULL,
    block_end        TEXT NOT NULL,
    block_size       INTEGER NOT NULL,
    numbers_used     INTEGER NOT NULL DEFAULT 0,
    next_sequence    INTEGER NOT NULL DEFAULT 1,
    expires_at       TIMESTAMPTZ,
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Orders -------------------------------------------------------------
CREATE TABLE orders (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    validation_number TEXT UNIQUE NOT NULL,
    event_id         UUID NOT NULL REFERENCES raffle_events(id) ON DELETE CASCADE,
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
    offline_sync_status TEXT REFERENCES offline_sync_statuses(key),
    offline_created_at   TIMESTAMPTZ,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by       UUID
);

-- Tickets ------------------------------------------------------------
CREATE TABLE tickets (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id         UUID NOT NULL REFERENCES raffle_events(id) ON DELETE CASCADE,
    draw_number      INTEGER NOT NULL,
    state            TEXT NOT NULL REFERENCES ticket_states(key),
    order_id         UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    sold_at          TIMESTAMPTZ NOT NULL,
    sold_by          UUID,  -- null for online
    voided_at        TIMESTAMPTZ,
    voided_by        UUID,
    voided_reason    TEXT,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(event_id, draw_number) -- ensures unique draw numbers per event
);
```

### Indexes

```sql
CREATE INDEX idx_orders_event          ON orders(event_id);
CREATE INDEX idx_orders_payment_status ON orders(payment_status);
CREATE INDEX idx_tickets_event_state   ON tickets(event_id, state);
CREATE INDEX idx_tickets_order         ON tickets(order_id);
CREATE INDEX idx_rsu_alloc_event       ON rsu_allocations(raffle_event_id);
```

---

## 6 – Draws & Claims

```sql
-- Draw Executions ----------------------------------------------------
CREATE TABLE draw_executions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    raffle_event_id  UUID NOT NULL REFERENCES raffle_events(id) ON DELETE CASCADE,
    state            TEXT NOT NULL, -- pending | in_progress | completed | failed
    started_at       TIMESTAMPTZ NOT NULL,
    rng_seed         TEXT NOT NULL,
    conducted_by     UUID NOT NULL,
    system_checksum  TEXT NOT NULL,
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
CREATE INDEX idx_prize_claims_result    ON prize_claims(draw_result_id);
```

---

## 7 – Critical Event Logging

```sql
-- Critical events per GLI-31 ----------------------------------------
CREATE TABLE critical_events (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type       TEXT NOT NULL,
    event_id         UUID,   -- nullable, depends on context
    rsu_id           UUID,
    user_id          UUID,
    severity         TEXT NOT NULL, -- INFO | WARN | ERROR
    description      TEXT,
    metadata         JSONB DEFAULT '{}'::JSONB,
    timestamp        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ttl              BIGINT  -- epoch when record will expire (if using Dynamo in prod)
);

CREATE INDEX idx_critical_events_type_time ON critical_events(event_type, timestamp DESC);
```

---

## 8 – System Configuration (Versioned)

```sql
CREATE TABLE system_configurations (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version          TEXT NOT NULL,
    effective_from   TIMESTAMPTZ NOT NULL,
    effective_to     TIMESTAMPTZ,
    settings         JSONB NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by       UUID
);
```

---

## 9 – Audit Trail

Critical tables employ lightweight `created_* / updated_*` columns, **plus** a centralized audit log for immutable history.

```sql
CREATE TABLE audit_log (
    id               BIGSERIAL PRIMARY KEY,
    table_name       TEXT NOT NULL,
    operation        TEXT NOT NULL CHECK (operation IN ('INSERT','UPDATE','DELETE')),
    record_id        UUID NOT NULL,
    old_data         JSONB,
    new_data         JSONB,
    changed_by       UUID,
    changed_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_table_time ON audit_log(table_name, changed_at DESC);
```

> **Trigger strategy (not shown):** For each critical table (`raffle_events`, `orders`, `tickets`, `prizes`, etc.) create `FOR EACH ROW` triggers that write detailed before/after rows into `audit_log`.

---

## 10 – Seed Data Examples (optional)

Below is **illustrative** seed data for lookup tables. Load via migrations, _not_ in production scripts.

```sql
-- Raffle event states
INSERT INTO raffle_event_states(key, description) VALUES
  ('created',   'Initial state after creation'),
  ('active',    'Sales are open'),
  ('suspended', 'Sales temporarily paused'),
  ('closed',    'Sales closed, awaiting draw'),
  ('cancelled', 'Event cancelled');
```

_(Add similar inserts for the other lookup tables.)_

---

### Future-Proofing Notes

1. Use `JSONB` columns (`configuration`, `settings`, `metadata`) to add optional fields without migrations.
2. Reference tables avoid the rigidity of SQL `ENUM`s.
3. All monetary amounts are `INTEGER` in lowest denomination – rename to `*_cents` if desired for clarity.
4. Time-series data (e.g., ticket sales history) can be extracted into append-only tables later to manage OLTP vs. analytics workloads.
5. Partitioning (e.g., by `event_id` for `tickets`) can be introduced when volume dictates, without affecting the logical model.

---

> **End of schema.**
