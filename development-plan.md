# Last-Mile Delivery Platform — Phased Development Plan

> Project: 212-last-mile-delivery-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (API + frontend) + Kotlin (Android) + Swift (iOS) | TypeScript for API and dispatcher dashboard; native mobile for driver app performance (GPS, camera, barcode scanning) |
| API framework | Fastify | High-throughput for concurrent driver GPS updates; WebSocket support for live tracking; auto-generates OpenAPI 3.1 |
| Database | PostgreSQL 16 + PostGIS | Normalised schema from data-model-suggestion-1; PostGIS for geospatial route queries, geofencing, and delivery zone management |
| ORM | Drizzle ORM | Type-safe; PostGIS integration; complex joins across order→route→stop→driver→ePOD |
| Task queue | BullMQ + Redis | Async route optimisation, SMS/email notifications, ePOD processing, carrier API calls |
| Route optimisation | Google OR-Tools (via Python microservice) or VROOM (C++ via REST) | Vehicle Routing Problem with Time Windows (VRPTW) solver; OR-Tools is Apache 2.0, VROOM is BSD-2 — both proven open-source VRP solvers |
| Real-time | WebSocket (Fastify WebSocket) + Redis Pub/Sub | Live driver GPS → dispatcher map; delivery status updates → customer tracking page |
| Maps | Mapbox GL JS (dispatcher) + Mapbox Navigation SDK (driver) | Dispatcher map with driver pins; driver turn-by-turn navigation |
| SMS/Email | Twilio (SMS) + Resend (email) | Customer delivery notifications with tracking links |
| LLM integration | Anthropic TypeScript SDK (Claude) | ETA prediction context, dispatcher-driver exception messaging, ePOD fraud analysis |
| Barcode scanning | ML Kit (Android/iOS) | GS1 barcode scanning in driver app for package verification |
| Mobile | React Native (Expo) | Cross-platform driver app with camera, GPS, and barcode scanning; faster iteration than fully native for MVP |
| Auth | Better Auth + JWT | Multi-tenant; driver JWT for mobile app; dispatcher session auth; API keys for integrations |
| Containerisation | Docker + docker-compose | PostgreSQL+PostGIS, Redis, Fastify API, VRP solver, Next.js dashboard |
| Frontend | Next.js 15 (App Router) + Tailwind CSS | Dispatcher dashboard with live map, route management, analytics |
| Testing | Vitest + Playwright + Detox (mobile) | API/frontend (Vitest/Playwright); Mobile (Detox for E2E) |
| Linting | Biome | Fast linter + formatter |
| Type checking | TypeScript strict mode | Enforced in CI |
| Package manager | pnpm | Workspace support |

### Project Structure

```
last-mile-delivery/
├── package.json
├── pnpm-workspace.yaml
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.frontend
├── Dockerfile.vrp                   # VRP solver microservice
├── packages/
│   ├── shared/                      # Shared types
│   │   └── src/types/
│   │       ├── order.ts
│   │       ├── route.ts
│   │       ├── stop.ts
│   │       ├── driver.ts
│   │       ├── epod.ts
│   │       └── tracking.ts
│   ├── api/                         # Fastify API
│   │   ├── drizzle.config.ts
│   │   ├── drizzle/migrations/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── config.ts
│   │   │   ├── db/schema.ts
│   │   │   ├── routes/
│   │   │   │   ├── orders.ts
│   │   │   │   ├── routes.ts
│   │   │   │   ├── drivers.ts
│   │   │   │   ├── stops.ts
│   │   │   │   ├── epod.ts
│   │   │   │   ├── tracking.ts
│   │   │   │   ├── carriers.ts
│   │   │   │   ├── zones.ts
│   │   │   │   ├── analytics.ts
│   │   │   │   └── webhooks.ts
│   │   │   ├── services/
│   │   │   │   ├── route-optimizer.ts
│   │   │   │   ├── dispatch-engine.ts
│   │   │   │   ├── driver-tracker.ts
│   │   │   │   ├── notification-service.ts
│   │   │   │   ├── epod-processor.ts
│   │   │   │   ├── eta-predictor.ts          # AI ETA (v1.1)
│   │   │   │   ├── carrier-selector.ts       # Multi-carrier (v1.1)
│   │   │   │   └── epod-fraud-detector.ts    # AI fraud (backlog)
│   │   │   ├── workers/
│   │   │   │   ├── optimisation.worker.ts
│   │   │   │   ├── notification.worker.ts
│   │   │   │   ├── tracking.worker.ts
│   │   │   │   └── eta.worker.ts
│   │   │   ├── websocket/
│   │   │   │   ├── driver-location.ts
│   │   │   │   └── tracking-updates.ts
│   │   │   └── lib/
│   │   │       ├── auth.ts
│   │   │       ├── geo.ts                    # PostGIS helpers
│   │   │       └── vrp-client.ts             # VRP solver client
│   │   └── tests/
│   ├── frontend/                    # Next.js dispatcher dashboard
│   │   ├── package.json
│   │   └── src/app/
│   │       ├── dashboard/
│   │       ├── routes/
│   │       ├── orders/
│   │       ├── drivers/
│   │       ├── zones/
│   │       ├── analytics/
│   │       ├── track/[trackingId]/ # Customer tracking page
│   │       └── settings/
│   └── vrp-solver/                  # Python VRP microservice
│       ├── pyproject.toml
│       └── src/
│           ├── main.py              # FastAPI
│           └── solver.py            # OR-Tools VRPTW
├── mobile/                          # React Native driver app
│   ├── package.json
│   └── src/
│       ├── screens/
│       │   ├── RouteScreen.tsx
│       │   ├── StopDetailScreen.tsx
│       │   ├── NavigationScreen.tsx
│       │   ├── EPODScreen.tsx
│       │   └── SettingsScreen.tsx
│       └── components/
├── tests/e2e/
└── docs/
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema from data-model-suggestion-1 (tenants, drivers, orders, routes, stops, delivery_zones), authentication, PostGIS, and Docker environment.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create pnpm workspace, Fastify API, Next.js frontend, VRP solver microservice, Docker with PostgreSQL+PostGIS and Redis.

**Design**:

```typescript
const envSchema = z.object({
  DATABASE_URL: z.string().default("postgresql://delivery:delivery@localhost:5432/delivery"),
  REDIS_URL: z.string().default("redis://localhost:6379"),
  VRP_SOLVER_URL: z.string().default("http://localhost:8001"),
  MAPBOX_TOKEN: z.string().optional(),
  TWILIO_ACCOUNT_SID: z.string().optional(),
  TWILIO_AUTH_TOKEN: z.string().optional(),
  TWILIO_FROM_NUMBER: z.string().optional(),
  RESEND_API_KEY: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  SITE_URL: z.string().default("http://localhost:3000"),
  API_PORT: z.coerce.number().default(3001),
  WS_PORT: z.coerce.number().default(3002),
});
```

**Testing**:
- Unit: config parses env
- Integration: `docker-compose up -d` → PostgreSQL+PostGIS, Redis, API, VRP solver healthy

#### 1.2 — Database Schema — Core Tables

**What**: Implement tenants, drivers, orders, routes, stops, delivery_zones, packages from data-model-suggestion-1.

**Design**:

Key entities from data-model-suggestion-1:

```typescript
export const orders = pgTable("orders", {
  id: uuid("id").defaultRandom().primaryKey(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  orderNumber: varchar("order_number", { length: 50 }).notNull(),
  externalOrderId: varchar("external_order_id", { length: 255 }),
  status: varchar("status", { length: 30 }).notNull().default("pending"),
  // pending → assigned → in_transit → at_stop → completed | failed | returned
  recipientName: varchar("recipient_name", { length: 255 }).notNull(),
  recipientPhone: varchar("recipient_phone", { length: 30 }),
  recipientEmail: varchar("recipient_email", { length: 255 }),
  deliveryAddressId: uuid("delivery_address_id").notNull().references(() => addresses.id),
  timeWindowStart: timestamp("time_window_start", { withTimezone: true }),
  timeWindowEnd: timestamp("time_window_end", { withTimezone: true }),
  packageCount: integer("package_count").notNull().default(1),
  totalWeightKg: numeric("total_weight_kg", { precision: 8, scale: 2 }),
  totalVolumeCm3: numeric("total_volume_cm3", { precision: 12, scale: 2 }),
  notes: text("notes"),
  trackingId: varchar("tracking_id", { length: 50 }).notNull().unique(),
  // auto-generated: 8-char alphanumeric for customer tracking
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
});

export const routes = pgTable("routes", {
  id: uuid("id").defaultRandom().primaryKey(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  driverId: uuid("driver_id").references(() => drivers.id),
  routeDate: date("route_date").notNull(),
  status: varchar("status", { length: 30 }).notNull().default("planned"),
  // planned → dispatched → in_progress → completed
  estimatedDistanceKm: numeric("estimated_distance_km", { precision: 8, scale: 2 }),
  estimatedDurationMinutes: integer("estimated_duration_minutes"),
  actualDistanceKm: numeric("actual_distance_km", { precision: 8, scale: 2 }),
  co2Kg: numeric("co2_kg", { precision: 8, scale: 3 }),
  optimisedAt: timestamp("optimised_at", { withTimezone: true }),
});

export const stops = pgTable("stops", {
  id: uuid("id").defaultRandom().primaryKey(),
  routeId: uuid("route_id").notNull().references(() => routes.id),
  orderId: uuid("order_id").notNull().references(() => orders.id),
  sequenceNumber: integer("sequence_number").notNull(),
  status: varchar("status", { length: 30 }).notNull().default("pending"),
  // pending → arrived → completed | failed | skipped
  estimatedArrivalAt: timestamp("estimated_arrival_at", { withTimezone: true }),
  actualArrivalAt: timestamp("actual_arrival_at", { withTimezone: true }),
  completedAt: timestamp("completed_at", { withTimezone: true }),
  durationSeconds: integer("duration_seconds"),
  failureReason: text("failure_reason"),
});

// Delivery zones: PostGIS polygon
export const deliveryZones = pgTable("delivery_zones", {
  id: uuid("id").defaultRandom().primaryKey(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  name: varchar("name", { length: 255 }).notNull(),
  // boundary stored as PostGIS GEOMETRY(POLYGON, 4326)
  // exposed as GeoJSON via ST_AsGeoJSON
});
```

Addresses with PostGIS POINT for geocoding (latitude/longitude).

**Testing**:
- Integration: migrations apply with PostGIS extension
- Integration: create order → route → 5 stops → FK chain holds
- Unit: delivery zone contains address → PostGIS ST_Contains query works
- Unit: trackingId auto-generated as 8-char alphanumeric

#### 1.3 — Authentication and Roles

**What**: Multi-tenant auth; driver JWT for mobile app; dispatcher session auth.

**Design**:

Roles: `admin` (full access), `dispatcher` (routes, orders, drivers, analytics), `driver` (own route, stop completion, ePOD), `api_service` (order ingestion, webhooks).

Driver auth: login with phone number + OTP → JWT with driver_id and tenant_id. JWT refreshed daily.

**Testing**:
- Unit: driver JWT contains correct driver_id and tenant_id
- Unit: driver can only access own route → other routes return 403
- Unit: dispatcher can access all routes for their tenant

---

## Phase 2: Route Optimisation

### Purpose
Build the core VRP solver that creates optimised multi-stop routes with time windows and vehicle capacity constraints. This is the primary value proposition.

### Tasks

#### 2.1 — VRP Solver Microservice

**What**: Python microservice wrapping Google OR-Tools for VRPTW (Vehicle Routing Problem with Time Windows).

**Design**:

```python
# packages/vrp-solver/src/solver.py
class VRPSolver:
    def solve(self, request: VRPRequest) -> VRPSolution:
        """
        Input:
        - depot: { lat, lng }
        - stops: [{ id, lat, lng, time_window_start, time_window_end, service_time_minutes, weight_kg, volume_cm3 }]
        - vehicles: [{ id, capacity_kg, capacity_cm3, max_stops, start_time, end_time }]
        - distance_matrix: computed from PostGIS or Mapbox Directions API

        OR-Tools setup:
        1. Create RoutingIndexManager with len(stops) + 1 (depot)
        2. Add distance/time dimension with transit callbacks
        3. Add capacity dimension (weight, volume)
        4. Add time-window constraints per stop
        5. Set search parameters: AUTOMATIC first solution, GUIDED_LOCAL_SEARCH metaheuristic
        6. Solve with time limit (30 seconds default)

        Output: { routes: [{ vehicle_id, stops: [{ stop_id, sequence, eta }], distance_km, duration_minutes }] }
        """

# POST /solve
class VRPRequest(BaseModel):
    depot: Location
    stops: list[StopInput]
    vehicles: list[VehicleInput]
    max_solve_time_seconds: int = 30

class VRPSolution(BaseModel):
    routes: list[OptimisedRoute]
    total_distance_km: float
    total_duration_minutes: int
    unassigned_stops: list[str]  # stops that couldn't be assigned
```

**Testing**:
- Unit: 10 stops, 2 vehicles → 2 routes with all stops assigned, sequence optimised
- Unit: stop with tight time window → assigned to route that can reach it in time
- Unit: vehicle capacity exceeded → stops split across vehicles
- Unit: infeasible stop (unreachable within time window) → in unassigned_stops
- Integration: API call with 50 stops → solution within 30 seconds

#### 2.2 — Route Planning API and UI

**What**: API and dispatcher UI for creating, optimising, and dispatching routes.

**Design**:

```typescript
// POST /api/v1/routes/optimise
interface RouteOptimiseRequest {
  routeDate: string;           // ISO 8601 date
  orderIds: string[];          // orders to include
  driverIds: string[];         // available drivers
  depotAddress?: AddressInput; // starting point (default: tenant depot)
}

// Response: { routes: OptimisedRoute[], unassignedOrders: string[] }
// Dispatcher reviews → POST /api/v1/routes/{id}/dispatch → sends to driver app
```

Dispatcher UI: `/routes` page with map showing planned routes (colour-coded per driver), stop sequence list, drag-to-reorder for manual adjustments, "Optimise" button to run VRP solver, "Dispatch" button to send to drivers.

**Testing**:
- E2E: select 20 orders + 3 drivers → click Optimise → routes appear on map
- E2E: drag stop to different position → sequence updates
- E2E: dispatch route → driver receives route in mobile app
- Unit: distance matrix computed from PostGIS → correct travel times

---

## Phase 3: Driver Mobile App

### Purpose
Build the driver-facing mobile app for navigation, stop completion, barcode scanning, and proof of delivery capture.

### Tasks

#### 3.1 — Driver App — Route View and Navigation

**What**: Driver sees assigned route with stop list; taps stop for turn-by-turn navigation.

**Design**:

Screens:
- **Route List**: today's stops in sequence with address, time window, package count. Pull-to-refresh.
- **Stop Detail**: recipient name, address, notes, packages to deliver. "Navigate" button launches Mapbox Navigation.
- **Navigation**: Mapbox turn-by-turn directions to next stop.

GPS tracking: driver's location sent to API every 15 seconds via WebSocket. Stored in Redis for real-time dispatcher map; periodically persisted to `driver_locations` table.

**Testing**:
- E2E: driver logs in → sees assigned route with correct stops
- E2E: tap "Navigate" → turn-by-turn navigation starts
- Unit: GPS updates sent every 15 seconds → dispatcher sees position on map

#### 3.2 — Driver App — Stop Completion and ePOD

**What**: Driver marks stops as completed with barcode scan, photo, signature, and notes.

**Design**:

Stop completion flow:
1. Arrive at stop → tap "Arrived" (geofence verification: driver within 200m of address)
2. Scan package barcode (GS1 SSCC or order barcode) → verify package matches order
3. Deliver package → capture:
   - **Photo**: camera capture of package at delivery location
   - **Signature**: on-screen signature pad
   - **Notes**: optional text ("left at door", "handed to doorman")
4. Tap "Complete" → stop status = `completed`, ePOD record created

Failed delivery: tap "Failed" → select reason (not home, refused, wrong address, damaged) → capture photo if applicable.

```typescript
// POST /api/v1/stops/{stopId}/complete
interface StopCompleteRequest {
  photoUrl: string;           // uploaded to S3
  signatureUrl: string;       // uploaded to S3
  barcodeScanned: string;
  notes: string | null;
  latitude: number;
  longitude: number;
}

// POST /api/v1/stops/{stopId}/fail
interface StopFailRequest {
  reason: "not_home" | "refused" | "wrong_address" | "damaged" | "access_issue" | "other";
  notes: string | null;
  photoUrl: string | null;
  latitude: number;
  longitude: number;
}
```

**Testing**:
- E2E: arrive at stop → scan barcode → take photo → sign → complete → status = completed
- E2E: wrong barcode scanned → error "Package does not match this stop"
- E2E: driver > 500m from address → warning "You appear to be far from the delivery address"
- Unit: ePOD record created with photo, signature, barcode, GPS coordinates, timestamp

---

## Phase 4: Live Tracking and Customer Notifications

### Purpose
Real-time delivery tracking for dispatchers and customers. Automated SMS/email notifications with tracking links. This is a table-stakes customer experience requirement.

### Tasks

#### 4.1 — Live Driver Tracking Map

**What**: Dispatcher map showing all active drivers with real-time GPS positions.

**Design**:

`/dashboard` page: Mapbox GL JS map with driver pins (coloured by status: green=on-route, yellow=at-stop, grey=idle). Click driver → sidebar with route progress, current stop, ETA to next stop. Stop pins on map (green=completed, yellow=current, grey=pending, red=failed).

WebSocket: API pushes driver location updates to connected dispatcher clients via Redis Pub/Sub → WebSocket.

**Testing**:
- E2E: 3 active drivers → 3 pins on map, positions update in real time
- E2E: click driver → route progress sidebar shows correct stop status
- Unit: WebSocket delivers location update within 2 seconds of GPS event

#### 4.2 — Customer Tracking Page and Notifications

**What**: Customer receives SMS/email with tracking link; tracking page shows live ETA and driver position.

**Design**:

Notifications triggered at:
1. Route dispatched → "Your delivery is on its way" with tracking link
2. Driver 3 stops away → "Your delivery is nearby, ETA: 15 minutes"
3. Driver at stop → "Your driver has arrived"
4. Delivery complete → "Your delivery has been completed" with ePOD photo

Tracking page: `/track/[trackingId]` (public, no auth) → map with driver position, ETA countdown, delivery status timeline.

```typescript
// GET /api/v1/tracking/{trackingId}
interface TrackingResponse {
  orderNumber: string;
  status: string;
  driverName: string;
  driverLocation: { lat: number; lng: number } | null;
  estimatedArrival: string | null;
  stopsRemaining: number;
  timeline: { event: string; timestamp: string }[];
  epod: { photoUrl: string; signedAt: string } | null;
}
```

**Testing**:
- E2E: route dispatched → customer receives SMS with tracking link
- E2E: open tracking page → map shows driver position, ETA updates
- E2E: delivery completed → tracking page shows "Delivered" with ePOD photo
- Integration (mocked Twilio): SMS sent with correct message and tracking URL

---

## Phase 5: Analytics and Reporting

### Purpose
Operational analytics for dispatchers and operations managers: on-time rate, completion rate, average stop time, CO2 per delivery.

### Tasks

#### 5.1 — Delivery Analytics Dashboard

**What**: Dashboard with KPIs, trend charts, and driver performance comparison.

**Design**:

`/analytics` page:
- **KPI Cards**: On-time rate %, completion rate %, failed delivery rate %, avg stops per route, avg stop duration (minutes)
- **OTIF Compliance**: On Time In Full % (orders delivered within time window with all packages)
- **CO2 Reporting**: total CO2 per delivery (estimated from route distance × emission factor per vehicle type)
- **Driver Comparison**: bar chart comparing drivers by stops completed, on-time %, avg stop time
- **Trend Charts**: daily/weekly/monthly trends for on-time rate and completion rate
- Date range picker, zone filter, driver filter

CO2 calculation: `co2_kg = distance_km × emission_factor_kg_per_km` (configurable per vehicle type; default 0.21 kg/km for van).

**Testing**:
- Unit: 100 deliveries, 85 on-time → on-time rate = 85%
- Unit: CO2 for 50km route with van → 10.5 kg
- E2E: analytics page → KPI cards show correct values
- E2E: filter by driver → metrics scoped to that driver

---

## Phase 6: Continuous Re-Optimisation (v1.1)

### Purpose
Move from static pre-shift batch optimisation to continuous mid-route re-optimisation triggered by failed stops, new orders, and traffic. This is the key AI differentiator.

### Tasks

#### 6.1 — Mid-Route Re-Optimisation

**What**: Re-optimise remaining stops when conditions change during delivery.

**Design**:

Re-optimisation triggers:
1. **Failed stop**: driver marks stop as failed → remaining stops re-sequenced
2. **New order added**: dispatcher adds urgent order mid-route → inserted optimally
3. **Traffic delay**: ETA slips beyond time window for upcoming stop → re-sequence

```typescript
class ContinuousOptimiser {
  async reoptimise(routeId: string, trigger: string): Promise<void> {
    // 1. Get current driver location from Redis
    // 2. Get remaining unvisited stops
    // 3. Call VRP solver with driver's current position as depot
    // 4. Update stop sequences with new solution
    // 5. Push updated route to driver app via WebSocket
    // 6. Recalculate ETAs for affected customers
    // 7. Send updated ETA notifications if window changed
  }
}
```

**Testing**:
- Unit: failed stop removed → remaining stops re-sequenced optimally
- Unit: new order inserted → placed at optimal position minimising total distance
- Unit: traffic delay detected → stops re-ordered to meet time windows
- E2E: driver fails stop → route updates on driver app within 30 seconds

---

## Phase 7: Multi-Carrier Integration (v1.1)

### Purpose
Support hybrid fleet: route some stops to internal drivers, others to 3PLs or parcel carriers (FedEx, UPS, DHL) based on cost, SLA, and capacity.

### Tasks

#### 7.1 — Carrier Selection and Label Generation

**What**: For each order, decide whether to fulfil via own driver or external carrier; generate shipping labels for carrier stops.

**Design**:

```typescript
class CarrierSelector {
  async select(order: Order): Promise<CarrierDecision> {
    // Decision logic:
    // 1. If order in a delivery zone with available driver capacity → own driver
    // 2. If order outside zones OR driver capacity full → evaluate carriers
    // 3. Rate-shop across connected carriers (FedEx, UPS, DHL) for cost + transit time
    // 4. Select cheapest option meeting SLA
    // Return: { method: "own_driver" | "carrier", carrierId?, estimatedCostCents, transitDays }
  }
}

// Carrier connectors: POST label request → return tracking number + label PDF
class FedExConnector { async createShipment(order: Order): Promise<ShipmentResult> { ... } }
class UPSConnector { async createShipment(order: Order): Promise<ShipmentResult> { ... } }
```

**Testing**:
- Unit: order in zone with capacity → own_driver selected
- Unit: order outside all zones → cheapest carrier selected
- Integration (mocked): FedEx API → label PDF and tracking number returned
- E2E: order assigned to carrier → tracking number visible in dispatcher dashboard

---

## Phase 8: AI-Enhanced Features (v1.1)

### Purpose
Add AI differentiators: predictive ETA, ePOD fraud detection, and natural-language dispatcher-driver communication.

### Tasks

#### 8.1 — AI-Powered ETA Prediction

**What**: Predict delivery ETAs using historical driver performance, address difficulty, traffic, and weather.

**Design**:

```typescript
class ETAPredictor {
  async predict(stop: Stop, driver: Driver): Promise<ETAPrediction> {
    // Features: distance_to_stop_km, time_of_day, day_of_week,
    //           driver_avg_stop_duration_seconds, address_difficulty_score (apartment vs house),
    //           traffic_level (from Mapbox Traffic API), weather_condition
    // Model: XGBoost regressor trained on historical stop completion data
    // Output: predicted_arrival_time, confidence_interval, delay_risk_score
  }
}
```

**Testing**:
- Unit: apartment address + rush hour → longer ETA than house in suburbs
- Unit: driver with historically fast stops → tighter ETA
- Integration: ETA displayed on customer tracking page → updates as driver progresses

#### 8.2 — ePOD Fraud Detection

**What**: Flag suspicious proof-of-delivery submissions using image analysis and geofence validation.

**Design**:

```typescript
class EPODFraudDetector {
  async analyse(epod: EPODRecord): Promise<FraudAnalysis> {
    // Checks:
    // 1. Geofence: driver GPS > 500m from delivery address → flag
    // 2. Photo analysis (GPT-4o Vision): does photo show a package at a door/location?
    //    Flag if: photo is blank, shows wrong location, appears reused
    // 3. Timing: stop completed in < 30 seconds → flag (unrealistically fast)
    // 4. Pattern: same photo hash used for multiple deliveries → flag
    // Return: { fraud_score: 0-1, flags: string[], recommendation: "review" | "clear" }
  }
}
```

Flagged ePODs surfaced in dispatcher dashboard for manual review.

**Testing**:
- Unit: driver 1km from address → geofence flag
- Unit: stop completed in 10 seconds → timing flag
- Unit (mocked Vision): blank photo → photo_suspicious flag
- Unit: duplicate photo hash → reused_photo flag

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                      ─── required by everything
    │
Phase 2: Route Optimisation              ─── requires Phase 1
    │
Phase 3: Driver Mobile App               ─── requires Phase 2
    │
Phase 4: Live Tracking & Notifications   ─── requires Phase 3
    │
Phase 5: Analytics & Reporting           ─── requires Phase 3 (parallel with Phase 4)
    │
Phase 6: Continuous Re-Optimisation      ─── requires Phase 2 + Phase 3
    │
Phase 7: Multi-Carrier Integration       ─── requires Phase 1 (parallel with Phases 2-6)
    │
Phase 8: AI-Enhanced Features           ─── requires Phase 3 + Phase 4
```

Parallelism opportunities:
- Phase 5 and Phase 4 can be developed concurrently after Phase 3
- Phase 7 can be developed concurrently with Phases 2-6 (independent carrier integration)
- Phase 8 tasks can be developed independently after their dependencies

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`vitest run`).
3. Biome linting passes with zero warnings.
4. TypeScript compiles in strict mode with zero errors.
5. Docker build succeeds for API, VRP solver, and frontend.
6. `docker-compose up` brings all services to healthy state.
7. Feature works end-to-end.
8. PostGIS geospatial queries return correct results (zone containment, distance).
9. VRP solver produces valid routes within time limit.
10. WebSocket delivers driver location updates within 2 seconds.
11. New API endpoints appear in auto-generated OpenAPI spec.
12. Database migrations created and tested.
13. Mobile app builds for iOS and Android (where applicable).
14. New environment variables documented in config.ts.
