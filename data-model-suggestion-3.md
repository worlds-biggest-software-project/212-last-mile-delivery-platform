# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Last-Mile Delivery Platform · Created: 2026-05-20

## Philosophy

This model uses a lean relational core for the entities and relationships that are stable across all deployments, combined with JSONB columns for data that varies by tenant, carrier, jurisdiction, or delivery vertical. The key insight is that a last-mile delivery platform serving grocery chains, furniture retailers, pharmacy deliveries, and courier companies will encounter significant variation in order attributes, proof-of-delivery requirements, and compliance fields -- but the core routing and dispatch workflow remains the same.

Rather than creating dozens of nullable columns or entity-attribute-value tables to handle this variation, the hybrid approach puts stable, frequently-queried fields in typed columns (with proper indexes and constraints) and puts variable, domain-specific fields in JSONB columns (with GIN indexes for containment queries). This gives the best of both worlds: relational integrity where it matters, and schema-on-write flexibility where the domain demands it.

This pattern is widely used in modern SaaS platforms. Stripe stores payment metadata in JSONB-style structures alongside relational core data. Shopify's order model has fixed fields plus a flexible `note_attributes` structure. The approach lets a single codebase serve verticals from restaurant delivery to big-and-bulky furniture without schema changes per customer.

**Best for:** Multi-vertical platforms that must serve diverse customer types (grocery, pharmacy, furniture, courier) without per-tenant schema customization; teams targeting rapid MVP delivery with iterative schema hardening.

**Trade-offs:**
- (+) Fastest path to MVP -- new fields can be added to JSONB without migrations
- (+) Single schema serves multiple delivery verticals without customization
- (+) JSONB GIN indexes enable efficient containment queries on flexible fields
- (+) Carrier-specific response data stored without schema proliferation
- (+) Tenant-specific configuration without separate config tables
- (-) JSONB fields lack database-level type enforcement -- validation must happen in application code
- (-) JSONB fields are invisible to database-level foreign key constraints
- (-) Over-reliance on JSONB can lead to a "schema-less mess" if not disciplined
- (-) Complex JSONB queries can be slower than equivalent relational queries
- (-) ORMs may not fully support JSONB querying patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 SSCC | `packages.sscc` as a typed column for the standard identifier |
| GS1 EPCIS 2.0 | `delivery_events.epcis_data` JSONB column stores full EPCIS event payloads for export |
| GS1 CBV | `delivery_events.business_step` typed column with CBV vocabulary |
| ISO 3166-1/2 | `addresses.country_code` typed column, subdivision in JSONB for jurisdictions that need it |
| GeoJSON (RFC 7946) | Delivery zones stored as PostGIS GEOMETRY, convertible to GeoJSON |
| OpenAPI 3.1 | Core relational fields map to fixed API schema; JSONB fields exposed as `additionalProperties` |

---

## Multi-Tenancy & Configuration

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'standard',
    -- Flexible tenant configuration avoids a separate settings table
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "branding": { "color": "#FF5722", "logo_url": "..." },
    --   "notifications": { "sms_enabled": true, "email_enabled": true, "provider": "twilio" },
    --   "pod_requirements": { "photo_required": true, "signature_required": false, "barcode_scan": true },
    --   "compliance": { "gdpr_enabled": true, "location_retention_days": 90 },
    --   "verticals": ["grocery", "pharmacy"],
    --   "co2_reporting": true,
    --   "carrier_selection_mode": "manual"  -- manual | rules | ai
    -- }
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
    -- User-specific preferences and permissions beyond the role
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "language": "es",
    --   "timezone": "America/Los_Angeles",
    --   "notification_preferences": { "sms": true, "email": false, "push": true }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

---

## Teams, Drivers & Vehicles

```sql
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    hub_location    GEOMETRY(Point, 4326),
    hub_address     JSONB,  -- full address object, avoids separate address table for hubs
    timezone        VARCHAR(100) NOT NULL DEFAULT 'UTC',
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "auto_dispatch": true,
    --   "max_stops_per_route": 50,
    --   "default_service_time_minutes": 5,
    --   "operating_hours": { "start": "07:00", "end": "21:00" }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_teams_tenant ON teams(tenant_id);

CREATE TABLE drivers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID REFERENCES teams(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'off_duty',
    current_location GEOMETRY(Point, 4326),
    location_updated_at TIMESTAMPTZ,
    -- Relational fields for core dispatch queries
    capacity_units  INTEGER NOT NULL DEFAULT 100,
    skills          TEXT[] NOT NULL DEFAULT '{}',
    -- Flexible driver attributes that vary by vertical
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example attributes:
    -- {
    --   "license_type": "CDL-B",
    --   "vehicle_id": "uuid",
    --   "vehicle_type": "refrigerated_van",
    --   "certifications": ["food_safety", "hazmat"],
    --   "languages": ["en", "es"],
    --   "max_weight_kg": 500,
    --   "rest_period_compliance": { "jurisdiction": "EU", "max_drive_hours": 9 }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drivers_tenant ON drivers(tenant_id);
CREATE INDEX idx_drivers_status ON drivers(tenant_id, status);
CREATE INDEX idx_drivers_location ON drivers USING GIST(current_location);
-- GIN index for skill-based dispatch queries
CREATE INDEX idx_drivers_skills ON drivers USING GIN(skills);
-- GIN index for JSONB attribute queries (e.g., find all drivers with food_safety cert)
CREATE INDEX idx_drivers_attributes ON drivers USING GIN(attributes jsonb_path_ops);

CREATE TABLE vehicles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    registration    VARCHAR(50) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    capacity_weight_kg NUMERIC(10,2),
    capacity_volume_m3 NUMERIC(10,2),
    fuel_type       VARCHAR(30),
    co2_per_km      NUMERIC(8,4),
    -- Vehicle-specific attributes that vary by fleet type
    specs           JSONB NOT NULL DEFAULT '{}',
    -- Example specs:
    -- {
    --   "make": "Ford", "model": "Transit", "year": 2024,
    --   "refrigeration": { "min_temp_c": -20, "max_temp_c": 8 },
    --   "lift_gate": true,
    --   "telematics": { "provider": "geotab", "device_id": "GT-12345" }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vehicles_tenant ON vehicles(tenant_id);
```

---

## Orders (The Heart of the Hybrid Model)

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    external_id     VARCHAR(255),

    -- Core relational fields: queried constantly, must be indexed and typed
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    priority        SMALLINT NOT NULL DEFAULT 0,
    time_window_start TIMESTAMPTZ,
    time_window_end   TIMESTAMPTZ,
    service_duration_minutes INTEGER NOT NULL DEFAULT 5,

    -- Customer info: relational for notification queries
    customer_name   VARCHAR(255) NOT NULL,
    customer_phone  VARCHAR(50),
    customer_email  VARCHAR(255),

    -- Locations: PostGIS for spatial queries
    pickup_location  GEOMETRY(Point, 4326),
    dropoff_location GEOMETRY(Point, 4326) NOT NULL,

    -- Addresses: JSONB because address formats vary by country
    pickup_address  JSONB,
    dropoff_address JSONB NOT NULL,
    -- Example address:
    -- {
    --   "line1": "123 Main St", "line2": "Apt 4B",
    --   "city": "San Francisco", "subdivision": "US-CA",
    --   "country": "US", "postal_code": "94105",
    --   "geocode_quality": "rooftop",
    --   "delivery_instructions": "Ring bell twice, leave at side door",
    --   "access_code": "4521",
    --   "floor": 4,
    --   "building_type": "apartment"
    -- }

    -- Vertical-specific order data in JSONB
    order_data      JSONB NOT NULL DEFAULT '{}',
    -- GROCERY example:
    -- {
    --   "vertical": "grocery",
    --   "bags_count": 5,
    --   "contains_alcohol": true,
    --   "age_verification_required": true,
    --   "temperature_requirements": ["ambient", "chilled"],
    --   "substitution_preferences": "contact_customer"
    -- }
    --
    -- FURNITURE example:
    -- {
    --   "vertical": "furniture",
    --   "items": [{ "name": "Sofa", "weight_kg": 85, "requires_assembly": true }],
    --   "crew_size_required": 2,
    --   "elevator_available": false,
    --   "stairs_flights": 3,
    --   "removal_of_old_item": true
    -- }
    --
    -- PHARMACY example:
    -- {
    --   "vertical": "pharmacy",
    --   "prescription_id": "RX-789456",
    --   "controlled_substance": false,
    --   "id_verification_required": true,
    --   "cold_chain_required": true,
    --   "hipaa_consent_obtained": true
    -- }

    -- Assignment tracking
    assigned_driver_id UUID REFERENCES drivers(id),
    assigned_route_id  UUID,  -- FK added after routes table
    carrier_id      UUID,     -- FK added after carriers table

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_tenant ON orders(tenant_id);
CREATE INDEX idx_orders_status ON orders(tenant_id, status);
CREATE INDEX idx_orders_external ON orders(tenant_id, external_id);
CREATE INDEX idx_orders_time_window ON orders(tenant_id, time_window_start, time_window_end)
    WHERE status IN ('pending', 'assigned');
CREATE INDEX idx_orders_dropoff ON orders USING GIST(dropoff_location);
-- GIN index for vertical-specific queries
CREATE INDEX idx_orders_data ON orders USING GIN(order_data jsonb_path_ops);
```

### Example JSONB Queries

```sql
-- Find all grocery orders requiring age verification
SELECT * FROM orders
WHERE tenant_id = '...'
  AND order_data @> '{"vertical": "grocery", "age_verification_required": true}';

-- Find all furniture orders requiring 2+ crew
SELECT * FROM orders
WHERE tenant_id = '...'
  AND order_data @> '{"vertical": "furniture"}'
  AND (order_data->>'crew_size_required')::int >= 2;

-- Find orders with cold chain requirements across verticals
SELECT * FROM orders
WHERE tenant_id = '...'
  AND (order_data @> '{"cold_chain_required": true}'
       OR order_data @> '{"temperature_requirements": ["chilled"]}');
```

---

## Packages

```sql
CREATE TABLE packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    sscc            VARCHAR(18),
    barcode         VARCHAR(255),
    description     VARCHAR(255),
    weight_kg       NUMERIC(10,2),
    dimensions      JSONB,  -- { "length_cm": 40, "width_cm": 30, "height_cm": 20 }
    handling        JSONB NOT NULL DEFAULT '{}',
    -- Example handling:
    -- { "fragile": true, "refrigerated": true, "hazmat_class": null, "stackable": false }
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
    team_id         UUID REFERENCES teams(id),
    date            DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',
    stop_count      INTEGER NOT NULL DEFAULT 0,
    -- Metrics populated during/after route execution
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- Example metrics:
    -- {
    --   "total_distance_m": 45200,
    --   "total_duration_s": 28800,
    --   "total_co2_g": 8540.5,
    --   "stops_completed": 23,
    --   "stops_failed": 2,
    --   "on_time_count": 20,
    --   "late_count": 3,
    --   "avg_stop_duration_s": 312,
    --   "optimisation_score": 0.87
    -- }
    polyline        GEOMETRY(LineString, 4326),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE orders ADD CONSTRAINT fk_orders_route FOREIGN KEY (assigned_route_id) REFERENCES routes(id);

CREATE INDEX idx_routes_tenant ON routes(tenant_id);
CREATE INDEX idx_routes_driver_date ON routes(driver_id, date);
CREATE INDEX idx_routes_status ON routes(tenant_id, status);

CREATE TABLE stops (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_id        UUID NOT NULL REFERENCES routes(id),
    order_id        UUID NOT NULL REFERENCES orders(id),
    sequence        INTEGER NOT NULL,
    type            VARCHAR(20) NOT NULL DEFAULT 'dropoff',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    location        GEOMETRY(Point, 4326) NOT NULL,
    eta             TIMESTAMPTZ,
    arrival_time    TIMESTAMPTZ,
    departure_time  TIMESTAMPTZ,
    -- Stop-level details in JSONB to avoid column proliferation
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "distance_from_prev_m": 3200,
    --   "duration_from_prev_s": 480,
    --   "customer_instructions": "Leave at back door",
    --   "access_code": "1234",
    --   "failure_reason": "customer_not_home",
    --   "attempt_number": 1,
    --   "re_optimised": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stops_route ON stops(route_id, sequence);
CREATE INDEX idx_stops_order ON stops(order_id);
```

---

## Proof of Delivery (Flexible per Vertical)

```sql
CREATE TABLE proof_of_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stop_id         UUID NOT NULL REFERENCES stops(id),
    order_id        UUID NOT NULL REFERENCES orders(id),
    driver_id       UUID NOT NULL REFERENCES drivers(id),
    -- Core ePOD fields that every vertical needs
    location        GEOMETRY(Point, 4326),
    location_accuracy_m NUMERIC(8,2),
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Flexible evidence collection varies by vertical and tenant config
    evidence        JSONB NOT NULL DEFAULT '{}',
    -- STANDARD delivery:
    -- {
    --   "signature_url": "s3://...",
    --   "photos": [
    --     { "url": "s3://...", "type": "doorstep", "ai_fraud_score": 0.02 },
    --     { "url": "s3://...", "type": "package", "ai_fraud_score": 0.01 }
    --   ],
    --   "scans": [{ "type": "barcode", "value": "PKG-001", "package_id": "uuid" }],
    --   "recipient_name": "Jane Smith",
    --   "is_contactless": false
    -- }
    --
    -- GROCERY with age verification:
    -- {
    --   "signature_url": "s3://...",
    --   "photos": [{ "url": "s3://...", "type": "id_document" }],
    --   "age_verified": true,
    --   "id_type": "drivers_license",
    --   "id_dob_confirmed": true,
    --   "bags_delivered": 5,
    --   "temperature_check": { "chilled_temp_c": 4.2, "compliant": true }
    -- }
    --
    -- FURNITURE:
    -- {
    --   "photos": [
    --     { "url": "s3://...", "type": "assembled_furniture" },
    --     { "url": "s3://...", "type": "old_item_removed" }
    --   ],
    --   "assembly_completed": true,
    --   "old_item_removed": true,
    --   "damage_noted": false,
    --   "customer_satisfaction_rating": 5
    -- }
    ai_fraud_score  NUMERIC(5,4),  -- aggregate fraud score across all evidence
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pod_stop ON proof_of_delivery(stop_id);
CREATE INDEX idx_pod_order ON proof_of_delivery(order_id);
CREATE INDEX idx_pod_evidence ON proof_of_delivery USING GIN(evidence jsonb_path_ops);
```

---

## Delivery Events

```sql
CREATE TABLE delivery_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    order_id        UUID,
    stop_id         UUID,
    route_id        UUID,
    driver_id       UUID,
    event_type      VARCHAR(50) NOT NULL,
    event_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    business_step   VARCHAR(50),  -- GS1 CBV
    disposition     VARCHAR(50),  -- GS1 CBV
    location        GEOMETRY(Point, 4326),
    source          VARCHAR(50),
    -- Full EPCIS payload for standards-compliant export
    epcis_data      JSONB,
    -- Event-specific details
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition by month in production
CREATE INDEX idx_events_tenant_time ON delivery_events(tenant_id, event_time DESC);
CREATE INDEX idx_events_order ON delivery_events(order_id, event_time);
CREATE INDEX idx_events_type ON delivery_events(tenant_id, event_type);
```

---

## Carriers (with JSONB Config per Carrier Type)

```sql
CREATE TABLE carriers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    type            VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Carrier-specific configuration varies dramatically
    config          JSONB NOT NULL DEFAULT '{}',
    -- FedEx example:
    -- {
    --   "api_endpoint": "https://apis.fedex.com",
    --   "account_number": "123456789",
    --   "service_types": ["GROUND", "EXPRESS_SAVER", "OVERNIGHT"],
    --   "label_format": "PDF",
    --   "rate_card": { "ground_base": 8.50, "per_kg": 0.45 }
    -- }
    --
    -- Gig platform example:
    -- {
    --   "api_endpoint": "https://api.doordash.com/drive/v2",
    --   "developer_id": "...",
    --   "max_distance_km": 15,
    --   "supported_verticals": ["restaurant", "grocery"],
    --   "sla_minutes": 45
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

ALTER TABLE orders ADD CONSTRAINT fk_orders_carrier FOREIGN KEY (carrier_id) REFERENCES carriers(id);

CREATE TABLE carrier_shipments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    carrier_id      UUID NOT NULL REFERENCES carriers(id),
    tracking_number VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    cost            NUMERIC(10,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- Carrier-specific response data stored as-is
    carrier_response JSONB NOT NULL DEFAULT '{}',
    -- { "label_url": "...", "estimated_delivery": "2026-05-21", "service_type": "GROUND" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carrier_shipments_order ON carrier_shipments(order_id);
CREATE INDEX idx_carrier_shipments_tracking ON carrier_shipments(tracking_number)
    WHERE tracking_number IS NOT NULL;
```

---

## Delivery Zones

```sql
CREATE TABLE delivery_zones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    team_id         UUID REFERENCES teams(id),
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Zone-specific rules vary by tenant
    rules           JSONB NOT NULL DEFAULT '{}',
    -- Example rules:
    -- {
    --   "surcharge": 5.00,
    --   "max_stops_per_route": 30,
    --   "restricted_hours": { "start": "22:00", "end": "07:00" },
    --   "required_vehicle_type": "cargo_bike",
    --   "carbon_zone": true
    -- }
    color           VARCHAR(7),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_zones_tenant ON delivery_zones(tenant_id);
CREATE INDEX idx_zones_boundary ON delivery_zones USING GIST(boundary);
```

---

## Notifications & Webhooks

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    order_id        UUID NOT NULL REFERENCES orders(id),
    channel         VARCHAR(20) NOT NULL,
    template        VARCHAR(100) NOT NULL,
    recipient       VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    sent_at         TIMESTAMPTZ,
    -- Channel-specific delivery details
    delivery_info   JSONB NOT NULL DEFAULT '{}',
    -- SMS: { "provider": "twilio", "message_sid": "SM...", "segments": 1 }
    -- Email: { "provider": "sendgrid", "message_id": "...", "opened": true }
    -- Push: { "provider": "firebase", "device_token": "...", "clicked": false }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_order ON notifications(order_id);

CREATE TABLE tracking_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    token           VARCHAR(64) NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- { "show_driver_name": true, "show_eta": true, "branding": { "color": "#FF5722" } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             VARCHAR(500) NOT NULL,
    events          TEXT[] NOT NULL,
    secret          VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Bringg-style field selection per subscription
    field_filter    JSONB,
    -- { "include": ["id", "status", "customer_name", "eta"], "exclude": ["order_data"] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Users | 2 | tenants, users |
| Teams, Drivers & Vehicles | 3 | teams, drivers, vehicles |
| Orders & Packages | 2 | orders, packages |
| Routes & Stops | 2 | routes, stops |
| Proof of Delivery | 1 | Single table with JSONB evidence (vs. 3 tables in Model 1) |
| Delivery Events | 1 | EPCIS-aligned with JSONB details |
| Carriers | 2 | carriers, carrier_shipments |
| Delivery Zones | 1 | With JSONB rules |
| Communication | 3 | notifications, tracking_links, webhook_subscriptions |
| **Total** | **17** | 32% fewer tables than Model 1 |

---

## Key Design Decisions

1. **JSONB for variation, columns for invariants.** Fields that every tenant, every vertical, and every query needs (status, priority, time_window, customer_name, location) are typed columns. Fields that vary by vertical (grocery age verification, furniture assembly confirmation, pharmacy HIPAA consent) live in JSONB. The boundary is drawn by query frequency and type safety needs.

2. **Single proof_of_delivery table.** Model 1 uses three tables (proof_of_delivery, pod_photos, pod_scans). This model uses one table with a JSONB `evidence` column that accommodates any combination of photos, signatures, scans, and vertical-specific confirmations. This dramatically simplifies the ePOD write path from the driver app.

3. **Addresses in JSONB, locations in PostGIS.** The address string representation (line1, city, postal code) lives in a JSONB column on the order, avoiding a separate addresses table. The geocoded point lives in a PostGIS GEOMETRY column for spatial indexing. This reflects the reality that address formats vary by country and most queries care about the point, not the string.

4. **Route metrics in JSONB.** Rather than individual columns for every metric, the `routes.metrics` JSONB column accumulates performance data (distance, duration, CO2, on-time counts) during route execution. New metrics can be added without migrations.

5. **Carrier config is fully JSONB.** Each carrier type (FedEx, UPS, DoorDash, custom 3PL) has completely different configuration requirements. JSONB avoids a carrier-type-specific config table per integration. The application layer validates config against carrier-specific schemas.

6. **Delivery zone rules in JSONB.** Zone-level business rules (surcharges, restricted hours, vehicle type requirements) vary per tenant and per zone. JSONB rules avoid a separate zone_rules table and make it easy to add new rule types.

7. **GIN indexes on key JSONB columns.** The `jsonb_path_ops` GIN index class on `orders.order_data`, `drivers.attributes`, and `proof_of_delivery.evidence` enables efficient containment queries (`@>` operator) without full table scans.

8. **Fewer tables, same information density.** This model stores the same information as Model 1 in 17 tables instead of 25. The reduction comes from collapsing normalized child tables (pod_photos, pod_scans, addresses, hubs, api_keys, driver_daily_stats, tenant_daily_stats, webhook_deliveries) into JSONB fields or parent tables. The trade-off is less database-level enforcement, compensated by application-level validation.
