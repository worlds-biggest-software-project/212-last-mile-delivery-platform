# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Last-Mile Delivery Platform · Created: 2026-05-20

## Philosophy

This model follows classical relational database design: every real-world concept gets its own table, relationships are expressed through foreign keys, and data integrity is enforced at the database level with constraints, unique indexes, and referential actions. The schema is designed for PostgreSQL with PostGIS for geospatial operations.

The normalized approach maps directly to the domain language used by dispatchers, drivers, and operations managers. An order is an order, a stop is a stop, a route is a route. There are no clever indirections or event replays required to answer basic operational questions. This makes the system easy to reason about, easy to query ad hoc, and easy to onboard new developers.

Real-world delivery platforms like Onfleet structure their API around exactly these entities (organizations, teams, workers, tasks, hubs). This model codifies that structure at the database level, adding proper normalization for carrier integrations, proof-of-delivery capture, and EPCIS-aligned event recording.

**Best for:** Teams that value data integrity, need complex cross-entity reporting, and want a schema that maps cleanly to the business domain without architectural indirection.

**Trade-offs:**
- (+) Strong referential integrity enforced at the database level
- (+) Straightforward SQL queries for reporting and analytics
- (+) Well-understood by most developers; low learning curve
- (+) Easy to build REST APIs that mirror the table structure
- (-) Higher table count increases migration complexity
- (-) Schema changes require migrations; less flexible for rapid iteration on new entity types
- (-) Junction tables for many-to-many relationships add query complexity
- (-) Audit trail requires separate trigger-based or application-level implementation

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 SSCC (Serial Shipping Container Code) | `packages.sscc` field stores the 18-digit GS1 identifier for each logistics unit |
| GS1 EPCIS 2.0 | `delivery_events` table structure mirrors EPCIS ObjectEvent fields (what/when/where/why/how) |
| GS1 CBV (Core Business Vocabulary) | `delivery_events.business_step` uses CBV-defined values (shipping, in_transit, delivered) |
| ISO 3166-1/2 | `addresses.country_code` and `addresses.subdivision_code` use ISO 3166 codes |
| GeoJSON (RFC 7946) | `delivery_zones.boundary` stored as PostGIS GEOMETRY, exposed as GeoJSON via `ST_AsGeoJSON` |
| OAuth 2.0 (RFC 6749) | `api_keys` and `oauth_clients` tables support both API key and OAuth client credentials flows |
| OpenAPI 3.1 | Schema designed to map 1:1 to OpenAPI resource definitions |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'standard',  -- standard, professional, enterprise
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
    role            VARCHAR(50) NOT NULL,  -- admin, dispatcher, manager, driver, api_client
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, suspended, deactivated
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_role ON users(tenant_id, role);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    key_hash        VARCHAR(255) NOT NULL UNIQUE,
    key_prefix      VARCHAR(12) NOT NULL,  -- first 8 chars for identification
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Teams, Drivers & Vehicles

```sql
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    hub_id          UUID,  -- FK added after hubs table
    timezone        VARCHAR(100) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_teams_tenant ON teams(tenant_id);

CREATE TABLE hubs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    address_id      UUID NOT NULL,  -- FK to addresses
    location        GEOMETRY(Point, 4326) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE teams ADD CONSTRAINT fk_teams_hub FOREIGN KEY (hub_id) REFERENCES hubs(id);

CREATE TABLE drivers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID REFERENCES teams(id),
    vehicle_id      UUID,  -- FK to vehicles
    license_number  VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'off_duty',  -- on_duty, on_route, off_duty, suspended
    current_location GEOMETRY(Point, 4326),
    location_updated_at TIMESTAMPTZ,
    capacity_units  INTEGER NOT NULL DEFAULT 100,  -- available capacity in abstract units
    skills          TEXT[] NOT NULL DEFAULT '{}',  -- e.g., 'fragile', 'refrigerated', 'heavy_lift'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drivers_tenant ON drivers(tenant_id);
CREATE INDEX idx_drivers_team ON drivers(team_id);
CREATE INDEX idx_drivers_status ON drivers(tenant_id, status);
CREATE INDEX idx_drivers_location ON drivers USING GIST(current_location);

CREATE TABLE vehicles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    registration    VARCHAR(50) NOT NULL,
    type            VARCHAR(50) NOT NULL,  -- van, truck, bike, car, cargo_bike
    capacity_weight_kg NUMERIC(10,2),
    capacity_volume_m3 NUMERIC(10,2),
    capacity_units  INTEGER,
    fuel_type       VARCHAR(30),  -- petrol, diesel, electric, hybrid
    co2_per_km      NUMERIC(8,4),  -- grams CO2 per km
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE drivers ADD CONSTRAINT fk_drivers_vehicle FOREIGN KEY (vehicle_id) REFERENCES vehicles(id);

CREATE INDEX idx_vehicles_tenant ON vehicles(tenant_id);
```

---

## Addresses & Geospatial

```sql
CREATE TABLE addresses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    line1           VARCHAR(255) NOT NULL,
    line2           VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    subdivision_code VARCHAR(10),  -- ISO 3166-2 (e.g., US-CA, GB-LND)
    country_code    CHAR(2) NOT NULL,       -- ISO 3166-1 alpha-2
    postal_code     VARCHAR(20),
    location        GEOMETRY(Point, 4326),
    geocode_quality VARCHAR(20),  -- rooftop, parcel_centroid, street_segment, approximate
    address_hash    VARCHAR(64),  -- SHA-256 for deduplication
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_addresses_tenant ON addresses(tenant_id);
CREATE INDEX idx_addresses_location ON addresses USING GIST(location);
CREATE INDEX idx_addresses_hash ON addresses(tenant_id, address_hash);

ALTER TABLE hubs ADD CONSTRAINT fk_hubs_address FOREIGN KEY (address_id) REFERENCES addresses(id);

CREATE TABLE delivery_zones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    team_id         UUID REFERENCES teams(id),
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,  -- exposed as GeoJSON via ST_AsGeoJSON
    color           VARCHAR(7),  -- hex color for map rendering
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_delivery_zones_tenant ON delivery_zones(tenant_id);
CREATE INDEX idx_delivery_zones_boundary ON delivery_zones USING GIST(boundary);
```

---

## Orders, Stops & Packages

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    external_id     VARCHAR(255),  -- caller's order ID from OMS/WMS
    customer_name   VARCHAR(255) NOT NULL,
    customer_phone  VARCHAR(50),
    customer_email  VARCHAR(255),
    pickup_address_id  UUID REFERENCES addresses(id),
    dropoff_address_id UUID NOT NULL REFERENCES addresses(id),
    priority        SMALLINT NOT NULL DEFAULT 0,  -- 0=normal, 1=high, 2=urgent
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, assigned, picked_up, in_transit, delivered, failed, cancelled
    required_skills TEXT[] NOT NULL DEFAULT '{}',
    time_window_start TIMESTAMPTZ,
    time_window_end   TIMESTAMPTZ,
    service_duration_minutes INTEGER NOT NULL DEFAULT 5,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_tenant ON orders(tenant_id);
CREATE INDEX idx_orders_status ON orders(tenant_id, status);
CREATE INDEX idx_orders_external ON orders(tenant_id, external_id);
CREATE INDEX idx_orders_created ON orders(tenant_id, created_at DESC);

CREATE TABLE packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    sscc            VARCHAR(18),  -- GS1 Serial Shipping Container Code
    barcode         VARCHAR(255),
    description     VARCHAR(255),
    weight_kg       NUMERIC(10,2),
    length_cm       NUMERIC(8,2),
    width_cm        NUMERIC(8,2),
    height_cm       NUMERIC(8,2),
    is_fragile      BOOLEAN NOT NULL DEFAULT false,
    is_refrigerated BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_packages_order ON packages(order_id);
CREATE INDEX idx_packages_sscc ON packages(sscc) WHERE sscc IS NOT NULL;
CREATE INDEX idx_packages_barcode ON packages(barcode) WHERE barcode IS NOT NULL;
```

---

## Routes & Stops

```sql
CREATE TABLE routes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    driver_id       UUID NOT NULL REFERENCES drivers(id),
    vehicle_id      UUID REFERENCES vehicles(id),
    hub_id          UUID REFERENCES hubs(id),
    date            DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',
        -- planned, optimised, in_progress, completed, cancelled
    optimised_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    total_distance_m INTEGER,  -- metres
    total_duration_s INTEGER,  -- seconds
    total_co2_g     NUMERIC(12,2),  -- grams CO2
    polyline        GEOMETRY(LineString, 4326),
    stop_count      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routes_tenant ON routes(tenant_id);
CREATE INDEX idx_routes_driver_date ON routes(driver_id, date);
CREATE INDEX idx_routes_status ON routes(tenant_id, status);

CREATE TABLE stops (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_id        UUID NOT NULL REFERENCES routes(id),
    order_id        UUID NOT NULL REFERENCES orders(id),
    sequence        INTEGER NOT NULL,  -- 1-based stop order
    address_id      UUID NOT NULL REFERENCES addresses(id),
    type            VARCHAR(20) NOT NULL DEFAULT 'dropoff',  -- pickup, dropoff
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, arrived, completed, failed, skipped
    eta             TIMESTAMPTZ,
    arrival_time    TIMESTAMPTZ,
    departure_time  TIMESTAMPTZ,
    distance_from_prev_m INTEGER,
    duration_from_prev_s INTEGER,
    failure_reason  VARCHAR(255),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stops_route ON stops(route_id, sequence);
CREATE INDEX idx_stops_order ON stops(order_id);
CREATE INDEX idx_stops_status ON stops(route_id, status);
```

---

## Proof of Delivery

```sql
CREATE TABLE proof_of_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stop_id         UUID NOT NULL REFERENCES stops(id),
    order_id        UUID NOT NULL REFERENCES orders(id),
    driver_id       UUID NOT NULL REFERENCES drivers(id),
    recipient_name  VARCHAR(255),
    signature_url   VARCHAR(500),  -- S3/storage URL
    location        GEOMETRY(Point, 4326),
    location_accuracy_m NUMERIC(8,2),
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_contactless  BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pod_stop ON proof_of_delivery(stop_id);
CREATE INDEX idx_pod_order ON proof_of_delivery(order_id);

CREATE TABLE pod_photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pod_id          UUID NOT NULL REFERENCES proof_of_delivery(id),
    photo_url       VARCHAR(500) NOT NULL,
    photo_type      VARCHAR(30) NOT NULL,  -- package, doorstep, signature, id_verification
    ai_fraud_score  NUMERIC(5,4),  -- 0.0000 to 1.0000, null if not yet analysed
    ai_analysis     JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pod_photos_pod ON pod_photos(pod_id);

CREATE TABLE pod_scans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pod_id          UUID NOT NULL REFERENCES proof_of_delivery(id),
    package_id      UUID REFERENCES packages(id),
    scan_type       VARCHAR(20) NOT NULL,  -- barcode, qr, rfid
    scan_value      VARCHAR(255) NOT NULL,
    scanned_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Delivery Events (EPCIS-Aligned)

```sql
CREATE TABLE delivery_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    order_id        UUID REFERENCES orders(id),
    stop_id         UUID REFERENCES stops(id),
    route_id        UUID REFERENCES routes(id),
    driver_id       UUID REFERENCES drivers(id),
    -- EPCIS "what"
    event_type      VARCHAR(50) NOT NULL,
        -- order_created, order_assigned, route_started, stop_arrived,
        -- delivery_attempted, delivery_completed, delivery_failed,
        -- proof_captured, route_completed
    -- EPCIS "when"
    event_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- EPCIS "where"
    location        GEOMETRY(Point, 4326),
    location_name   VARCHAR(255),
    -- EPCIS "why"
    business_step   VARCHAR(50),  -- GS1 CBV: shipping, in_transit, delivering, delivered
    disposition     VARCHAR(50),  -- GS1 CBV: in_progress, completeness_verified, damaged
    -- EPCIS "how"
    source          VARCHAR(50),  -- driver_app, dispatcher_ui, api, system
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_delivery_events_tenant ON delivery_events(tenant_id, event_time DESC);
CREATE INDEX idx_delivery_events_order ON delivery_events(order_id, event_time);
CREATE INDEX idx_delivery_events_route ON delivery_events(route_id, event_time);
CREATE INDEX idx_delivery_events_type ON delivery_events(tenant_id, event_type);
```

---

## Carrier Integration

```sql
CREATE TABLE carriers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,  -- fedex, ups, dhl, custom
    type            VARCHAR(30) NOT NULL,  -- parcel_carrier, 3pl, gig_platform
    api_endpoint    VARCHAR(500),
    credentials     JSONB,  -- encrypted at application level
    is_active       BOOLEAN NOT NULL DEFAULT true,
    rate_card       JSONB,  -- pricing rules
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE carrier_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    carrier_id      UUID NOT NULL REFERENCES carriers(id),
    tracking_number VARCHAR(255),
    label_url       VARCHAR(500),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, label_created, picked_up, in_transit, delivered, failed
    cost            NUMERIC(10,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    carrier_response JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carrier_assignments_order ON carrier_assignments(order_id);
CREATE INDEX idx_carrier_assignments_tracking ON carrier_assignments(tracking_number);
```

---

## Customer Notifications & Tracking

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    order_id        UUID NOT NULL REFERENCES orders(id),
    channel         VARCHAR(20) NOT NULL,  -- sms, email, push
    template        VARCHAR(100) NOT NULL,  -- eta_update, out_for_delivery, delivered, failed
    recipient       VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, sent, delivered, failed
    sent_at         TIMESTAMPTZ,
    external_id     VARCHAR(255),  -- provider message ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_order ON notifications(order_id);
CREATE INDEX idx_notifications_status ON notifications(tenant_id, status);

CREATE TABLE tracking_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    token           VARCHAR(64) NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    view_count      INTEGER NOT NULL DEFAULT 0,
    last_viewed_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tracking_links_token ON tracking_links(token);
```

---

## Analytics & Reporting

```sql
CREATE TABLE driver_daily_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    driver_id       UUID NOT NULL REFERENCES drivers(id),
    date            DATE NOT NULL,
    stops_completed INTEGER NOT NULL DEFAULT 0,
    stops_failed    INTEGER NOT NULL DEFAULT 0,
    distance_m      INTEGER NOT NULL DEFAULT 0,
    duration_s      INTEGER NOT NULL DEFAULT 0,
    on_time_count   INTEGER NOT NULL DEFAULT 0,
    late_count      INTEGER NOT NULL DEFAULT 0,
    co2_g           NUMERIC(12,2) NOT NULL DEFAULT 0,
    avg_stop_duration_s INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (driver_id, date)
);

CREATE INDEX idx_driver_daily_stats_tenant_date ON driver_daily_stats(tenant_id, date DESC);

CREATE TABLE tenant_daily_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    date            DATE NOT NULL,
    orders_created  INTEGER NOT NULL DEFAULT 0,
    orders_delivered INTEGER NOT NULL DEFAULT 0,
    orders_failed   INTEGER NOT NULL DEFAULT 0,
    otif_rate       NUMERIC(5,4),  -- On Time In Full rate (0.0000 to 1.0000)
    avg_delivery_time_s INTEGER,
    total_distance_m BIGINT NOT NULL DEFAULT 0,
    total_co2_g     NUMERIC(14,2) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, date)
);
```

---

## Webhooks

```sql
CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             VARCHAR(500) NOT NULL,
    events          TEXT[] NOT NULL,  -- e.g., {'order.created', 'order.delivered', 'route.completed'}
    secret          VARCHAR(255) NOT NULL,  -- HMAC signing secret
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id),
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, sent, failed, retrying
    attempts        SMALLINT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    response_code   SMALLINT,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_status ON webhook_deliveries(status, next_retry_at)
    WHERE status IN ('pending', 'retrying');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | tenants, users, api_keys |
| Teams, Drivers & Vehicles | 4 | teams, hubs, drivers, vehicles |
| Addresses & Geospatial | 2 | addresses, delivery_zones |
| Orders & Packages | 2 | orders, packages |
| Routes & Stops | 2 | routes, stops |
| Proof of Delivery | 3 | proof_of_delivery, pod_photos, pod_scans |
| Delivery Events | 1 | EPCIS-aligned event log |
| Carrier Integration | 2 | carriers, carrier_assignments |
| Customer Communication | 2 | notifications, tracking_links |
| Analytics | 2 | driver_daily_stats, tenant_daily_stats |
| Webhooks | 2 | webhook_subscriptions, webhook_deliveries |
| **Total** | **25** | |

---

## Key Design Decisions

1. **Tenant-scoped everything.** Every table with business data carries a `tenant_id` column with a foreign key to `tenants`. Row-level security (RLS) can be layered on top, but the FK ensures referential integrity even without RLS.

2. **Addresses as a first-class entity.** Addresses are normalized into their own table with geocode quality tracking. The `address_hash` column enables deduplication across orders. This supports the research finding that geocoding accuracy is a hidden dependency for route quality.

3. **PostGIS for all geospatial data.** Driver locations, hub positions, delivery zone boundaries, and delivery event locations all use PostGIS GEOMETRY columns with SRID 4326. GiST indexes enable efficient spatial queries like "which delivery zone does this address fall in?"

4. **EPCIS-aligned delivery events.** The `delivery_events` table follows the EPCIS five-dimension model (what/when/where/why/how) with GS1 CBV-compatible business step and disposition values. This enables EPCIS 2.0 JSON-LD event emission without data transformation.

5. **Proof of delivery as a composite entity.** ePOD is modeled as a parent record with child photo and scan records, supporting the multi-evidence pattern (photo + signature + barcode scan) that the feature survey identified as table stakes.

6. **Pre-aggregated analytics tables.** `driver_daily_stats` and `tenant_daily_stats` are materialized summaries populated by background jobs, avoiding expensive real-time aggregation queries against the events and stops tables.

7. **Carrier integration as a pluggable entity.** Carriers are tenant-scoped with JSONB credentials and rate cards, allowing each tenant to configure their own carrier mix without schema changes. The `carrier_assignments` junction table tracks per-order carrier handoffs.

8. **Webhook delivery with retry state.** Webhook deliveries track attempt count, response codes, and next retry time, enabling exponential backoff without a separate queue system for simple deployments.
