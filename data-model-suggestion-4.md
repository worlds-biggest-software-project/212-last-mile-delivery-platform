# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Last-Mile Delivery Platform · Created: 2026-05-20

## Philosophy

This model combines a relational core for operational CRUD with a property graph layer for relationship-heavy queries. The key observation is that a last-mile delivery platform is fundamentally a network problem: drivers belong to teams, teams serve zones, zones contain addresses, orders flow through routes, routes visit stops in sequence, carriers connect to orders, and all of these entities have temporal relationships that change throughout the day.

The relational layer handles the operational workload: creating orders, assigning drivers, recording proof of delivery. The graph layer (implemented as `graph_nodes` and `graph_edges` tables in PostgreSQL, or optionally backed by Apache AGE or Neo4j) handles queries that are awkward or slow in relational SQL: "which drivers have delivered to this address before?", "what is the shortest path from the current driver position through all remaining stops?", "which carriers have the best on-time rate for this delivery zone?", "show me all entities connected to this failed delivery for root cause analysis."

This pattern is used by logistics platforms that need to optimize across interconnected entities. Route optimization is itself a graph traversal problem (Travelling Salesman / Vehicle Routing Problem variants). Carrier selection involves traversing a network of zone-carrier-SLA relationships. Fraud detection benefits from graph pattern matching (same address, different recipients, same time window).

**Best for:** Platforms where route optimization, carrier network analysis, and relationship-based queries (driver-address history, zone-carrier performance, delivery pattern detection) are first-class requirements, not afterthoughts.

**Trade-offs:**
- (+) Graph queries for route optimization, carrier selection, and pattern detection are natural and fast
- (+) Relationship discovery queries that would require multiple JOINs in relational SQL become single-hop traversals
- (+) Fraud detection via graph pattern matching (suspicious delivery clusters, repeated failures at same address)
- (+) Driver-address affinity scoring for intelligent dispatch ("this driver has delivered here 12 times")
- (+) Flexible relationship types: new edge types can be added without schema changes
- (-) Dual-model complexity: developers must understand both relational and graph paradigms
- (-) Data synchronization between relational and graph layers adds operational overhead
- (-) Graph query languages (Cypher, GQL, or recursive CTEs) have a steeper learning curve
- (-) Fewer off-the-shelf ORM tools support graph patterns natively
- (-) PostgreSQL-native graph (Apache AGE, recursive CTEs) is less mature than dedicated graph databases

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 SSCC | Package nodes carry SSCC identifiers as properties |
| GS1 EPCIS 2.0 | Delivery events modeled as timestamped edges between entity nodes |
| ISO 3166-1/2 | Zone and address nodes carry ISO jurisdiction codes as properties |
| GeoJSON (RFC 7946) | Zone boundaries stored as PostGIS GEOMETRY on the relational layer |
| GS1 CBV | Event edge properties include CBV business_step and disposition |
| W3C RDF / JSON-LD | Graph structure is exportable as RDF triples for linked data interoperability |

---

## Relational Core (Operational CRUD)

```sql
-- Standard multi-tenant identity
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

CREATE TABLE drivers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID REFERENCES teams(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'off_duty',
    current_location GEOMETRY(Point, 4326),
    location_updated_at TIMESTAMPTZ,
    capacity_units  INTEGER NOT NULL DEFAULT 100,
    skills          TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drivers_tenant ON drivers(tenant_id);
CREATE INDEX idx_drivers_status ON drivers(tenant_id, status);
CREATE INDEX idx_drivers_location ON drivers USING GIST(current_location);

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

CREATE INDEX idx_zones_boundary ON delivery_zones USING GIST(boundary);

CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    external_id     VARCHAR(255),
    customer_name   VARCHAR(255) NOT NULL,
    customer_phone  VARCHAR(50),
    customer_email  VARCHAR(255),
    pickup_location  GEOMETRY(Point, 4326),
    dropoff_location GEOMETRY(Point, 4326) NOT NULL,
    dropoff_address JSONB NOT NULL,
    pickup_address  JSONB,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    priority        SMALLINT NOT NULL DEFAULT 0,
    time_window_start TIMESTAMPTZ,
    time_window_end   TIMESTAMPTZ,
    service_duration_minutes INTEGER NOT NULL DEFAULT 5,
    required_skills TEXT[] NOT NULL DEFAULT '{}',
    order_data      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_tenant ON orders(tenant_id);
CREATE INDEX idx_orders_status ON orders(tenant_id, status);
CREATE INDEX idx_orders_dropoff ON orders USING GIST(dropoff_location);

CREATE TABLE routes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    driver_id       UUID NOT NULL REFERENCES drivers(id),
    vehicle_id      UUID REFERENCES vehicles(id),
    date            DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',
    stop_count      INTEGER NOT NULL DEFAULT 0,
    polyline        GEOMETRY(LineString, 4326),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    metrics         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routes_driver_date ON routes(driver_id, date);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stops_route ON stops(route_id, sequence);

CREATE TABLE packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    sscc            VARCHAR(18),
    barcode         VARCHAR(255),
    weight_kg       NUMERIC(10,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proof_of_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stop_id         UUID NOT NULL REFERENCES stops(id),
    order_id        UUID NOT NULL REFERENCES orders(id),
    driver_id       UUID NOT NULL REFERENCES drivers(id),
    location        GEOMETRY(Point, 4326),
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    evidence        JSONB NOT NULL DEFAULT '{}',
    ai_fraud_score  NUMERIC(5,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
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

CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    order_id        UUID NOT NULL REFERENCES orders(id),
    channel         VARCHAR(20) NOT NULL,
    template        VARCHAR(100) NOT NULL,
    recipient       VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             VARCHAR(500) NOT NULL,
    events          TEXT[] NOT NULL,
    secret          VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Property Graph Layer

```sql
-- Generic graph node table: every domain entity that participates in
-- relationship queries gets a node. The entity_table + entity_id
-- fields point back to the relational source of truth.
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
        -- driver, order, stop, route, address, zone, carrier, team, vehicle, customer
    entity_table    VARCHAR(100) NOT NULL,  -- relational table name
    entity_id       UUID NOT NULL,          -- PK in the relational table
    label           VARCHAR(255),           -- human-readable label
    location        GEOMETRY(Point, 4326),  -- for spatial graph queries
    properties      JSONB NOT NULL DEFAULT '{}',
        -- Cached properties from the relational table for graph-only queries
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entity_table, entity_id)
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(tenant_id, node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_table, entity_id);
CREATE INDEX idx_graph_nodes_location ON graph_nodes USING GIST(location);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN(properties jsonb_path_ops);

-- Generic graph edge table: typed, directed, temporal relationships
-- between any two nodes.
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    edge_type       VARCHAR(50) NOT NULL,
        -- ASSIGNED_TO, BELONGS_TO, DELIVERED_TO, SERVES_ZONE, DRIVES_VEHICLE,
        -- CONTAINS_STOP, CARRIED_BY, FAILED_AT, PICKED_UP_FROM, LOCATED_IN
    weight          NUMERIC(12,4),          -- cost, distance, duration, score
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Temporal validity: when was/is this relationship active?
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,            -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_type ON graph_edges(tenant_id, edge_type);
CREATE INDEX idx_graph_edges_temporal ON graph_edges(valid_from, valid_to)
    WHERE valid_to IS NULL;  -- active edges
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN(properties jsonb_path_ops);
```

---

## Graph Edge Type Catalogue

| Edge Type | Source Node | Target Node | Weight | Properties |
|-----------|------------|-------------|--------|------------|
| `ASSIGNED_TO` | order | driver | | `{ "assigned_at": "..." }` |
| `BELONGS_TO` | driver | team | | `{ "role": "lead" }` |
| `SERVES_ZONE` | team | zone | | `{ "priority": 1 }` |
| `DRIVES_VEHICLE` | driver | vehicle | | `{ "assigned_date": "..." }` |
| `CONTAINS_STOP` | route | stop | sequence | `{ "eta": "..." }` |
| `DELIVERED_TO` | driver | address | delivery_count | `{ "avg_duration_s": 285, "success_rate": 0.96 }` |
| `CARRIED_BY` | order | carrier | cost | `{ "tracking": "1Z...", "service": "GROUND" }` |
| `LOCATED_IN` | address | zone | | `{ "confirmed": true }` |
| `FAILED_AT` | order | address | attempt_count | `{ "reasons": ["not_home", "access_denied"] }` |
| `PICKED_UP_FROM` | order | address | | `{ "pickup_time": "..." }` |
| `NEXT_STOP` | stop | stop | distance_m | `{ "duration_s": 480 }` |
| `CARRIER_SERVES` | carrier | zone | on_time_rate | `{ "avg_cost": 8.50, "avg_delivery_hours": 4.2 }` |

---

## Example Graph Queries

### Driver-Address Affinity (for intelligent dispatch)

```sql
-- Find drivers who have successfully delivered to this address before,
-- ranked by number of deliveries and success rate
WITH target_address AS (
    SELECT id FROM graph_nodes
    WHERE node_type = 'address'
      AND properties->>'address_hash' = 'abc123hash'
)
SELECT
    gn_driver.entity_id AS driver_id,
    gn_driver.label AS driver_name,
    ge.weight AS delivery_count,
    (ge.properties->>'success_rate')::numeric AS success_rate,
    (ge.properties->>'avg_duration_s')::int AS avg_stop_duration_s
FROM graph_edges ge
JOIN graph_nodes gn_driver ON ge.source_node_id = gn_driver.id
JOIN target_address ta ON ge.target_node_id = ta.id
WHERE ge.edge_type = 'DELIVERED_TO'
  AND ge.valid_to IS NULL
  AND gn_driver.node_type = 'driver'
ORDER BY ge.weight DESC, success_rate DESC
LIMIT 5;
```

### Carrier Performance by Zone (for automated carrier selection)

```sql
-- For a given delivery zone, find the best carrier by on-time rate
WITH target_zone AS (
    SELECT id FROM graph_nodes
    WHERE node_type = 'zone'
      AND entity_id = '...'  -- zone UUID
)
SELECT
    gn_carrier.label AS carrier_name,
    gn_carrier.entity_id AS carrier_id,
    ge.weight AS on_time_rate,
    (ge.properties->>'avg_cost')::numeric AS avg_cost,
    (ge.properties->>'avg_delivery_hours')::numeric AS avg_hours
FROM graph_edges ge
JOIN graph_nodes gn_carrier ON ge.source_node_id = gn_carrier.id
JOIN target_zone tz ON ge.target_node_id = tz.id
WHERE ge.edge_type = 'CARRIER_SERVES'
  AND ge.valid_to IS NULL
ORDER BY ge.weight DESC;
```

### Route as a Graph Traversal

```sql
-- Reconstruct a route as a linked list of stops via NEXT_STOP edges
WITH RECURSIVE route_path AS (
    -- Start: find the first stop (no incoming NEXT_STOP edge)
    SELECT
        gn.entity_id AS stop_id,
        gn.location,
        gn.properties,
        1 AS hop,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    JOIN graph_edges ge_contains ON ge_contains.target_node_id = gn.id
    WHERE ge_contains.edge_type = 'CONTAINS_STOP'
      AND ge_contains.source_node_id = (
          SELECT id FROM graph_nodes WHERE entity_table = 'routes' AND entity_id = '...'
      )
      AND NOT EXISTS (
          SELECT 1 FROM graph_edges ge_prev
          WHERE ge_prev.target_node_id = gn.id AND ge_prev.edge_type = 'NEXT_STOP'
      )

    UNION ALL

    -- Traverse NEXT_STOP edges
    SELECT
        gn_next.entity_id,
        gn_next.location,
        gn_next.properties,
        rp.hop + 1,
        rp.path || gn_next.id
    FROM route_path rp
    JOIN graph_edges ge_next ON ge_next.source_node_id = rp.path[array_length(rp.path, 1)]
    JOIN graph_nodes gn_next ON ge_next.target_node_id = gn_next.id
    WHERE ge_next.edge_type = 'NEXT_STOP'
      AND NOT (gn_next.id = ANY(rp.path))  -- cycle prevention
)
SELECT stop_id, hop, properties->>'eta' AS eta, ST_AsGeoJSON(location) AS geojson
FROM route_path
ORDER BY hop;
```

### Fraud Detection: Suspicious Delivery Clusters

```sql
-- Find addresses with high failure rates from multiple different drivers
-- (potential indicator of fraudulent delivery claims)
SELECT
    gn_addr.label AS address,
    gn_addr.properties->>'address_hash' AS address_hash,
    COUNT(DISTINCT ge.source_node_id) AS unique_drivers,
    SUM(ge.weight) AS total_failures,
    array_agg(DISTINCT ge.properties->>'reasons') AS failure_reasons
FROM graph_edges ge
JOIN graph_nodes gn_addr ON ge.target_node_id = gn_addr.id
WHERE ge.edge_type = 'FAILED_AT'
  AND ge.tenant_id = '...'
  AND ge.created_at >= now() - INTERVAL '30 days'
  AND gn_addr.node_type = 'address'
GROUP BY gn_addr.id, gn_addr.label, gn_addr.properties->>'address_hash'
HAVING COUNT(DISTINCT ge.source_node_id) >= 3
   AND SUM(ge.weight) >= 5
ORDER BY total_failures DESC;
```

---

## Graph Synchronization

The graph layer is maintained by application-level event handlers that create/update nodes and edges when relational data changes.

```sql
-- Track synchronization state between relational and graph layers
CREATE TABLE graph_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_table    VARCHAR(100) NOT NULL,
    source_id       UUID NOT NULL,
    operation       VARCHAR(20) NOT NULL,  -- insert, update, delete
    node_id         UUID REFERENCES graph_nodes(id),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'success',  -- success, failed, pending
    error_message   TEXT
);

CREATE INDEX idx_graph_sync_status ON graph_sync_log(sync_status)
    WHERE sync_status != 'success';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenants, users |
| Teams, Drivers & Vehicles | 3 | teams, drivers, vehicles |
| Orders & Packages | 2 | orders, packages |
| Routes & Stops | 2 | routes, stops |
| Proof of Delivery | 1 | With JSONB evidence |
| Carriers & Zones | 2 | carriers, delivery_zones |
| Communication | 2 | notifications, webhook_subscriptions |
| Graph Layer | 2 | graph_nodes, graph_edges |
| Graph Infrastructure | 1 | graph_sync_log |
| **Total** | **17** | Plus 2 graph tables that mirror relational data |

---

## Key Design Decisions

1. **Graph as a secondary index, not the source of truth.** The relational tables are the authoritative data store. The graph layer is a derived, queryable index optimized for relationship traversal. If the graph layer is lost, it can be fully reconstructed from the relational tables. This avoids the operational risk of a graph-only architecture.

2. **Temporal edges for historical analysis.** Every graph edge has `valid_from` and `valid_to` timestamps. When a driver is reassigned from one team to another, the old BELONGS_TO edge gets a `valid_to` timestamp and a new one is created. This enables temporal graph queries without an event store.

3. **Driver-address affinity as a first-class edge.** The DELIVERED_TO edge between driver and address nodes accumulates delivery count and success rate. This enables the AI dispatch system to prefer drivers who have successfully delivered to an address before, reducing failed delivery attempts.

4. **Carrier-zone performance edges.** CARRIER_SERVES edges store aggregated performance metrics (on-time rate, average cost, average delivery time) between carrier and zone nodes. The automated carrier selection engine traverses these edges to make real-time allocation decisions.

5. **Route modeled as a linked list in the graph.** NEXT_STOP edges between stop nodes form a traversable linked list. Re-optimization (inserting, removing, or reordering stops) is a graph edge manipulation, which is more natural than updating sequence integers in a relational table.

6. **Graph sync via event handlers.** Relational writes trigger graph synchronization through application-level event handlers (or PostgreSQL triggers). The `graph_sync_log` table tracks sync status and enables detection of graph drift.

7. **PostgreSQL-native implementation.** The graph is implemented as standard PostgreSQL tables with recursive CTEs for traversal. This avoids introducing a separate graph database (Neo4j, Neptune) and keeps the operational footprint simple. If graph query performance becomes a bottleneck, Apache AGE (a PostgreSQL extension providing Cypher query language) can be added without schema changes.

8. **JSONB properties on nodes and edges.** Rather than creating separate property tables per node type or edge type, cached properties live in JSONB columns. This keeps the graph schema at exactly two tables regardless of how many entity types and relationship types are added.
