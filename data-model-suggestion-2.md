# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Last-Mile Delivery Platform · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The current state of any entity (order, route, driver) is derived by replaying its event stream. Read-optimized projections (materialized views) serve operational queries, while the event store serves as the authoritative audit trail and the foundation for temporal queries ("what was the state of this order at 14:32?").

Event sourcing is particularly well-suited to last-mile delivery because the domain is inherently event-driven: orders are created, assigned, picked up, attempted, delivered, or failed. Regulators and enterprise customers increasingly demand full audit trails. The EPCIS 2.0 standard itself is an event-based data model. By making events the source of truth rather than an afterthought, the platform gains native audit compliance, temporal querying, and the ability to replay events for AI model training.

The model uses CQRS (Command Query Responsibility Segregation): write operations append events to the event store, while read operations query denormalized projection tables. This separation allows independent optimization of write throughput (append-only inserts) and read performance (pre-joined, indexed projections).

**Best for:** Platforms where full audit trail is a procurement requirement, temporal queries are needed for dispute resolution, and event data will feed AI/ML pipelines for route optimization and fraud detection.

**Trade-offs:**
- (+) Complete, immutable audit trail from day one — no separate audit logging needed
- (+) Temporal queries are native: "show me the state of this route at 2pm" is a simple event replay
- (+) Event streams are ideal training data for AI models (route optimization, ETA prediction, fraud detection)
- (+) EPCIS 2.0 compliance is architecturally native, not bolted on
- (+) Write path is append-only, enabling high throughput
- (-) Read queries require projection tables that must be kept in sync — eventual consistency
- (-) Higher storage requirements (events are never deleted, only compacted)
- (-) More complex to implement than straightforward CRUD
- (-) Schema evolution of events requires careful versioning
- (-) Debugging requires understanding event replay, not just reading a row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 | The event store IS the EPCIS event log — each delivery event maps directly to an EPCIS ObjectEvent |
| GS1 CBV | Event `business_step` and `disposition` fields use CBV vocabulary values |
| GS1 SSCC | Package identifiers stored as `epc_list` entries within events, per EPCIS spec |
| ISO 3166-1/2 | Location context in events uses ISO country and subdivision codes |
| GeoJSON (RFC 7946) | Event location data stored as GeoJSON-compatible coordinates |
| CloudEvents 1.0 | Event envelope structure follows CloudEvents spec for interoperability |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth. All state is derived from this table.
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,  -- order, route, driver, carrier_assignment
    stream_id       UUID NOT NULL,         -- the aggregate root ID
    sequence_num    BIGINT NOT NULL,       -- monotonically increasing per stream
    event_type      VARCHAR(100) NOT NULL, -- e.g., OrderCreated, StopArrived, DeliveryCompleted
    event_version   SMALLINT NOT NULL DEFAULT 1,  -- schema version for this event type
    -- EPCIS dimensions embedded
    event_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    business_step   VARCHAR(50),           -- GS1 CBV: shipping, in_transit, delivering, delivered
    disposition     VARCHAR(50),           -- GS1 CBV: in_progress, completeness_verified
    -- Payload
    data            JSONB NOT NULL,        -- event-specific payload
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- { "source": "driver_app", "user_id": "...", "ip": "...", "device": "..." }
    -- CloudEvents-compatible envelope fields
    ce_source       VARCHAR(255) NOT NULL DEFAULT '/delivery-platform',
    ce_type         VARCHAR(255) NOT NULL,  -- mirrors event_type with namespace
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
);

-- Primary query: replay a single aggregate's event stream
CREATE INDEX idx_events_stream ON events(stream_id, sequence_num);

-- Cross-aggregate queries by tenant and time
CREATE INDEX idx_events_tenant_time ON events(tenant_id, event_time DESC);

-- Query by event type for projections and consumers
CREATE INDEX idx_events_type ON events(tenant_id, event_type, event_time DESC);

-- Partition by month for retention management
-- In production, this table would be range-partitioned on event_time:
-- CREATE TABLE events (...) PARTITION BY RANGE (event_time);
```

### Event Type Examples

```sql
-- OrderCreated event data payload:
-- {
--   "external_id": "SHOP-12345",
--   "customer": { "name": "Jane Smith", "phone": "+1555123456", "email": "jane@example.com" },
--   "pickup": { "address_id": "uuid", "location": [lng, lat] },
--   "dropoff": { "address_id": "uuid", "location": [lng, lat] },
--   "packages": [{ "sscc": "003412345678901234", "weight_kg": 2.5, "barcode": "PKG-001" }],
--   "time_window": { "start": "2026-05-20T14:00:00Z", "end": "2026-05-20T16:00:00Z" },
--   "priority": 1,
--   "required_skills": ["refrigerated"]
-- }

-- StopArrived event data payload:
-- {
--   "stop_id": "uuid",
--   "route_id": "uuid",
--   "location": [lng, lat],
--   "location_accuracy_m": 8.5,
--   "eta_vs_actual_s": -120
-- }

-- DeliveryCompleted event data payload:
-- {
--   "stop_id": "uuid",
--   "pod": {
--     "recipient_name": "Jane Smith",
--     "signature_url": "s3://bucket/sig-uuid.png",
--     "photos": [{ "url": "s3://bucket/photo-uuid.jpg", "type": "doorstep" }],
--     "scans": [{ "type": "barcode", "value": "PKG-001" }],
--     "location": [lng, lat],
--     "is_contactless": false
--   }
-- }

-- DeliveryFailed event data payload:
-- {
--   "stop_id": "uuid",
--   "reason": "customer_not_home",
--   "location": [lng, lat],
--   "photo_url": "s3://bucket/failed-uuid.jpg",
--   "reschedule_requested": true
-- }
```

---

## Event Stream Snapshots

```sql
-- Periodic snapshots to avoid replaying long event streams
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    sequence_num    BIGINT NOT NULL,       -- snapshot taken at this sequence number
    state           JSONB NOT NULL,        -- the full aggregate state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_num)
);

-- To reconstruct state: load latest snapshot, then replay events after snapshot.sequence_num
```

---

## Read Projections (Materialized from Events)

### Orders Projection

```sql
CREATE TABLE proj_orders (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    external_id     VARCHAR(255),
    customer_name   VARCHAR(255) NOT NULL,
    customer_phone  VARCHAR(50),
    customer_email  VARCHAR(255),
    pickup_location GEOMETRY(Point, 4326),
    dropoff_location GEOMETRY(Point, 4326),
    pickup_address  JSONB,
    dropoff_address JSONB,
    priority        SMALLINT NOT NULL DEFAULT 0,
    status          VARCHAR(30) NOT NULL,
    required_skills TEXT[],
    time_window_start TIMESTAMPTZ,
    time_window_end   TIMESTAMPTZ,
    assigned_driver_id UUID,
    assigned_route_id  UUID,
    carrier_id      UUID,
    tracking_number VARCHAR(255),
    delivered_at    TIMESTAMPTZ,
    failed_at       TIMESTAMPTZ,
    failure_reason  VARCHAR(255),
    last_event_seq  BIGINT NOT NULL,      -- watermark: last event applied to this projection
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_orders_tenant ON proj_orders(tenant_id);
CREATE INDEX idx_proj_orders_status ON proj_orders(tenant_id, status);
CREATE INDEX idx_proj_orders_external ON proj_orders(tenant_id, external_id);
CREATE INDEX idx_proj_orders_driver ON proj_orders(assigned_driver_id) WHERE assigned_driver_id IS NOT NULL;
```

### Routes Projection

```sql
CREATE TABLE proj_routes (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    driver_id       UUID NOT NULL,
    vehicle_id      UUID,
    hub_id          UUID,
    date            DATE NOT NULL,
    status          VARCHAR(20) NOT NULL,
    stop_count      INTEGER NOT NULL DEFAULT 0,
    stops_completed INTEGER NOT NULL DEFAULT 0,
    stops_failed    INTEGER NOT NULL DEFAULT 0,
    total_distance_m INTEGER,
    total_duration_s INTEGER,
    total_co2_g     NUMERIC(12,2),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_routes_tenant ON proj_routes(tenant_id);
CREATE INDEX idx_proj_routes_driver_date ON proj_routes(driver_id, date);
CREATE INDEX idx_proj_routes_status ON proj_routes(tenant_id, status);
```

### Stops Projection

```sql
CREATE TABLE proj_stops (
    id              UUID PRIMARY KEY,
    route_id        UUID NOT NULL,
    order_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    sequence        INTEGER NOT NULL,
    type            VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    address         JSONB NOT NULL,
    location        GEOMETRY(Point, 4326),
    eta             TIMESTAMPTZ,
    arrival_time    TIMESTAMPTZ,
    departure_time  TIMESTAMPTZ,
    pod_captured    BOOLEAN NOT NULL DEFAULT false,
    pod_photo_count SMALLINT NOT NULL DEFAULT 0,
    pod_signature   BOOLEAN NOT NULL DEFAULT false,
    failure_reason  VARCHAR(255),
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_stops_route ON proj_stops(route_id, sequence);
CREATE INDEX idx_proj_stops_order ON proj_stops(order_id);
```

### Drivers Projection

```sql
CREATE TABLE proj_drivers (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    team_id         UUID,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    status          VARCHAR(20) NOT NULL,
    current_location GEOMETRY(Point, 4326),
    location_updated_at TIMESTAMPTZ,
    current_route_id UUID,
    vehicle_id      UUID,
    capacity_units  INTEGER,
    skills          TEXT[],
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_drivers_tenant ON proj_drivers(tenant_id);
CREATE INDEX idx_proj_drivers_status ON proj_drivers(tenant_id, status);
CREATE INDEX idx_proj_drivers_location ON proj_drivers USING GIST(current_location);
```

---

## Supporting Operational Tables

These tables are not event-sourced because they represent slowly-changing reference data, not domain event streams.

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    role            VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    hub_location    GEOMETRY(Point, 4326),
    timezone        VARCHAR(100) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vehicles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    registration    VARCHAR(50) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    capacity_weight_kg NUMERIC(10,2),
    capacity_volume_m3 NUMERIC(10,2),
    fuel_type       VARCHAR(30),
    co2_per_km      NUMERIC(8,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE carriers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    type            VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE delivery_zones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    team_id         UUID REFERENCES teams(id),
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_delivery_zones_boundary ON delivery_zones USING GIST(boundary);
```

---

## Event Consumer Infrastructure

```sql
-- Tracks which events each consumer/projection has processed
CREATE TABLE consumer_checkpoints (
    consumer_name   VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed projection processing
CREATE TABLE consumer_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_name   VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dead_letters_consumer ON consumer_dead_letters(consumer_name, created_at);

-- Webhook subscriptions (webhooks are just another event consumer)
CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             VARCHAR(500) NOT NULL,
    event_types     TEXT[] NOT NULL,
    secret          VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Temporal Query: Order state at a specific time

```sql
-- Reconstruct order state as of 2026-05-20 14:32:00 UTC
SELECT data
FROM events
WHERE stream_type = 'order'
  AND stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND event_time <= '2026-05-20T14:32:00Z'
ORDER BY sequence_num ASC;

-- Application code replays these events to reconstruct the aggregate state
```

### EPCIS 2.0 Export: Generate EPCIS events for a shipment

```sql
SELECT
    event_id AS "eventID",
    event_type AS "type",
    event_time AS "eventTime",
    business_step AS "bizStep",
    disposition,
    data->'location' AS "readPoint",
    data->'packages' AS "epcList"
FROM events
WHERE stream_type = 'order'
  AND stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND business_step IS NOT NULL
ORDER BY event_time ASC;
```

### Analytics: Failed delivery patterns by hour

```sql
SELECT
    EXTRACT(HOUR FROM event_time) AS hour_of_day,
    data->>'reason' AS failure_reason,
    COUNT(*) AS failure_count
FROM events
WHERE tenant_id = '...'
  AND event_type = 'DeliveryFailed'
  AND event_time >= now() - INTERVAL '30 days'
GROUP BY 1, 2
ORDER BY 1, 3 DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, snapshots |
| Read Projections | 4 | proj_orders, proj_routes, proj_stops, proj_drivers |
| Reference Data | 6 | tenants, users, teams, vehicles, carriers, delivery_zones |
| Infrastructure | 3 | consumer_checkpoints, consumer_dead_letters, webhook_subscriptions |
| **Total** | **15** | Fewer tables, but events table is the workhorse |

---

## Key Design Decisions

1. **Events are the single source of truth.** The `events` table is append-only and immutable. Projection tables are derived and can always be rebuilt by replaying events. This provides a complete audit trail without any additional logging infrastructure.

2. **Stream-based aggregation.** Each event belongs to a stream (identified by `stream_type` + `stream_id`). An order's full history is its stream. A route's full history is its stream. This maps directly to DDD aggregate roots and makes event replay straightforward.

3. **Snapshots for performance.** Long-running aggregates (orders with many status changes) get periodic snapshots to avoid replaying hundreds of events on every read. The snapshot + subsequent events pattern keeps reconstruction fast.

4. **Projections carry watermarks.** Every projection row has a `last_event_seq` field that records the last event sequence number applied. This enables idempotent projection updates and makes it easy to detect stale projections.

5. **Reference data is NOT event-sourced.** Tenants, users, teams, vehicles, and carriers are standard CRUD tables. Event sourcing is reserved for domain aggregates with significant state transitions (orders, routes, drivers). This avoids over-engineering slowly-changing reference data.

6. **CloudEvents-compatible envelope.** The `ce_source` and `ce_type` fields follow the CloudEvents 1.0 specification, enabling direct forwarding of events to external systems (Kafka, webhooks, EPCIS receivers) without transformation.

7. **Partition-ready design.** The events table should be range-partitioned by `event_time` in production. Old partitions can be moved to cold storage while keeping recent events on fast disks. Events are never deleted, but partitioning makes retention management practical.

8. **Dead letter queue for projection failures.** When a projection consumer fails to process an event, it goes to the dead letter queue rather than blocking the entire consumer pipeline. This is critical for operational reliability when running dozens of projection consumers.
