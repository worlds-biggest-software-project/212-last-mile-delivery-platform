# Standards & API Reference

> Project: Last-Mile Delivery Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 19987 — EPCIS (Electronic Product Code Information Services)**
- URL: https://www.gs1.org/standards/epcis
- GS1's flagship data-sharing standard (also published as an ISO/IEC standard) for capturing and sharing supply-chain visibility events. Defines the "what, when, where, why, and how" of package movements. EPCIS 2.0 supports JSON and modern REST API communication, making it the canonical standard for recording scan events, custody transfers, and proof-of-delivery confirmations in last-mile workflows.

**ISO 8000 — Data Quality**
- URL: https://www.iso.org/standard/50798.html
- Defines quality requirements for master data, including address and location data exchanged between OMS, WMS, and delivery platforms. Address accuracy directly affects route optimisation quality and delivery success rates.

**GS1 SSCC — Serial Shipping Container Code**
- URL: https://www.gs1.org/standards/id-keys/sscc
- 18-digit GS1 standard identifier for logistics units (parcels, cases, pallets). The SSCC enables unambiguous package identification across all stakeholders in last-mile delivery, typically encoded in a GS1-128 barcode or RFID tag. Required by major retailers (Amazon, Walmart, Target) for inbound logistics and return workflows.

**GS1 Logistic Label Guideline**
- URL: https://www.gs1.org/standards/gs1-logistic-label-guideline/1-3
- Specifies how barcodes, SSCC codes, and human-readable data must be formatted on shipping labels to ensure compatibility with automated scanning systems at hubs and during last-mile delivery confirmation.

**GS1 Logistics Interoperability Model (LIM)**
- URL: https://www.gs1.org/docs/EDI/GS1_Logistics_Interoperability_Model_Application_Standard.pdf
- Application standard for electronic data interchange between logistics stakeholders. Provides the data model for transport instructions, freight status updates, and proof of delivery that connects shippers, carriers, and recipients.

**GS1 EPCIS & CBV (Core Business Vocabulary)**
- URL: https://www.gs1.org/standards/epcis-and-cbv-implementation-guideline/current-standardd
- CBV is the companion vocabulary standard to EPCIS, defining the permitted values for business steps (e.g., "shipping", "in transit", "delivered") and dispositions (e.g., "in_progress", "completeness_inferred"). Required for interoperable delivery-event sharing across carrier networks.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Defines HTTP methods (GET, POST, PUT, PATCH, DELETE), status codes, and content negotiation used by all REST APIs in this domain. The delivery platform API must comply with RFC 7231 semantics for webhook delivery confirmations, order creation, and status updates.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Standard for compact, URL-safe token claims. JWTs are used for authenticating API clients and as short-lived access tokens for driver mobile app sessions. Platforms typically use JWTs for internal service-to-service trust and opaque tokens for mobile clients (per OAuth best practices).

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authorisation framework used across all major platforms (Onfleet, Bringg, Route4Me) for third-party integrations. OAuth 2.0 Client Credentials flow is standard for server-to-server API access (OMS → delivery platform); Authorization Code flow for merchant-facing integrations.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the `Link` header used in paginated REST API responses. Relevant for the delivery platform's API when returning large collections (all stops for a route, historical delivery records).

**RFC 6455 — WebSocket Protocol**
- URL: https://www.rfc-editor.org/rfc/rfc6455
- The WebSocket protocol standard underlying real-time driver location streaming and dispatcher map updates. Fleetbase uses WebSocket as a first-class real-time channel; other platforms use server-sent events or polling for similar functionality.

**W3C Geolocation API**
- URL: https://www.w3.org/TR/geolocation/
- Browser and mobile Geolocation API standard used in driver apps to capture GPS coordinates for tracking, geo-fenced delivery confirmation, and ePOD timestamp/location anchoring.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.0**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The de-facto standard for documenting REST APIs. Onfleet, Route4Me, and Routific all publish OpenAPI-compatible documentation. An AI-native delivery platform should provide a machine-readable OpenAPI 3.1 spec to ease third-party integration and SDK generation.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/draft/2020-12/schema
- Underpins OpenAPI 3.1 schema definitions. Used to validate request payloads (order creation, driver assignment, ePOD submission) and enforce data quality at API boundaries.

**GeoJSON (RFC 7946)**
- URL: https://www.rfc-editor.org/rfc/rfc7946
- Standard JSON format for encoding geographic data structures: Point (driver location), LineString (route polyline), Polygon (delivery zone / territory boundary). Should be the canonical format for all geospatial data in the delivery platform API.

**Protocol Buffers (Protobuf) — Google**
- URL: https://protobuf.dev/
- Binary serialisation format used in high-throughput internal microservices (e.g., real-time location event ingestion). Not required for public API but relevant for the internal event pipeline processing millions of GPS pings per day.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
- URL: https://www.rfc-editor.org/rfc/rfc7636
- PKCE (Proof Key for Code Exchange) is the required extension for OAuth 2.0 in mobile and single-page app contexts. Driver mobile apps must use OAuth 2.0 with PKCE, not implicit flow, to prevent authorisation code interception.

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 for SSO integration. Enterprise customers (3PLs, retail chains) require SSO into the dispatcher dashboard via their corporate identity provider; OpenID Connect is the standard integration path.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- The reference checklist for securing the delivery platform's public API. Key concerns in this domain: Broken Object Level Authorisation (drivers accessing other drivers' stops), Excessive Data Exposure (location data in API responses), and Rate Limiting (protecting the route optimisation endpoint from abuse).

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/download/soc-2-type-2-reporting-on-controls-at-a-service-organization
- De-facto compliance requirement for enterprise SaaS in this category. Bringg and Onfleet are SOC 2 certified. Any platform targeting enterprise logistics buyers must achieve SOC 2 Type II to pass procurement security reviews.

**GDPR — General Data Protection Regulation**
- URL: https://gdpr.eu/
- Applies to driver location tracking, customer address data, and delivery photo storage across EU operations. Key requirements: legal basis documentation for continuous GPS tracking, proportionality (no tracking during rest periods), data minimisation, and retention limits. Platforms must support data-subject access requests and deletion workflows.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant if the delivery platform exposes an AI agent interface allowing LLMs to query delivery status, create tasks, or reassign stops via natural language.

**MCP Specification**
- URL: https://modelcontextprotocol.io/specification
- An MCP server exposing delivery operations tools (create_task, get_route_status, reassign_stop, get_driver_location) would allow AI agents (dispatcher assistants, operations copilots) to interact with the platform using standardised tool-calling. Relevant for the natural-language dispatcher communication opportunity identified in the AI-native research.

---

## Similar Products — Developer Documentation & APIs

### Onfleet
- **Description:** Mid-market last-mile delivery SaaS with a developer-first API reputation. REST API covers workers, tasks, teams, hubs, organisations, and webhooks.
- **API Documentation:** https://docs.onfleet.com/reference/introduction
- **SDKs/Libraries:** Python (https://github.com/onfleet/pyonfleet), Node.js (https://github.com/onfleet/node-onfleet), PHP, Go, Ruby, Java — all in the https://github.com/onfleet org
- **Developer Guide:** https://docs.onfleet.com/ — includes quickstart, Postman collection, and webhook reference
- **Standards:** REST/JSON, OpenAPI-compatible docs, webhooks via HTTPS POST
- **Authentication:** API key (Basic Auth header); OAuth not required for server-to-server

### Bringg
- **Description:** Enterprise delivery orchestration platform with multi-carrier network and dynamic slot booking. API connects OMS, VMS, and ERP to own-fleet and 3PL delivery operations.
- **API Documentation:** https://developers.bringg.com/reference/welcome-to-bringgs-api-reference
- **SDKs/Libraries:** No official SDKs published; REST + JSON only
- **Developer Guide:** https://developers.bringg.com/docs/overview
- **Webhooks:** https://developers.bringg.com/docs/bringg-webhooks — flexible field selection per webhook subscription
- **Standards:** REST/JSON, HTTPS POST webhooks
- **Authentication:** API key + shared secret for webhook signature verification

### Route4Me
- **Description:** Route optimisation SaaS with the broadest SDK language coverage in the category. API-driven access to route planning, territory management, and GPS tracking.
- **API Documentation:** https://route4me.io/docs/ and https://integrate.route4me.com/
- **SDKs/Libraries:** C#, VB.NET, Python, Java, Node, C++, Ruby, PHP, Go, Erlang, Perl, cURL, VBScript — all at https://github.com/route4me
- **Developer Guide:** https://support.route4me.com/api-sdk-developer-docs/
- **Standards:** REST/JSON, webhook callbacks for optimisation state changes
- **Authentication:** API key (query parameter or header)

### Routific
- **Description:** SMB-focused route optimisation with strong route-quality reputation. Two APIs: Platform API (order management) and Engine API (optimisation calculations).
- **API Documentation (Platform):** https://routific-platform.readme.io/reference/routific-projects-api
- **API Documentation (Engine):** https://docs.routific.com/
- **Developer Guide:** https://dev.routific.com/ and https://academy.routific.com/en/articles/1317922-api-overview-of-the-routific-api
- **Standards:** REST/JSON, Bearer token authentication
- **Authentication:** Bearer token (Authorization header)

### Locus DispatchIQ
- **Description:** Enterprise AI dispatch and route optimisation platform. Full REST API suite for TMS, OMS, WMS, and ERP integration. API access requires client credentials.
- **API Documentation:** https://docs.locus.sh/ (requires Locus client account)
- **API Reference Index:** https://locus.sh/resources/api-references/
- **SDKs/Libraries:** Not publicly documented; REST/JSON integration via API plugins
- **Standards:** REST/JSON, webhook event stream
- **Authentication:** Client credentials (contact Locus sales for access)

### Fleetbase (Open Source)
- **Description:** Open-source modular logistics operating system. REST API with WebSocket and webhook support. Self-hostable; AGPL-3.0 or commercial licence.
- **API Documentation:** https://docs.fleetbase.io/api/
- **Developer Guide:** https://docs.fleetbase.io/developers/api/ and https://docs.fleetbase.dev/
- **SDKs/Libraries:** Postman collection available; no official SDK packages
- **GitHub Repository:** https://github.com/fleetbase/fleetbase
- **Standards:** REST/JSON, WebSocket (RFC 6455), HTTPS webhooks
- **Authentication:** API key (per-environment, managed via admin console)

### Google Maps Platform (Routing & Geocoding)
- **Description:** Maps, geocoding, and routing APIs used as the underlying geospatial infrastructure by most delivery platforms for address resolution, distance matrices, and route polyline rendering.
- **Geocoding API:** https://developers.google.com/maps/documentation/geocoding/guides-v3/overview
- **Routes API (successor to Directions API):** https://developers.google.com/maps/documentation/routes
- **Distance Matrix API:** https://developers.google.com/maps/documentation/distance-matrix
- **Standards:** REST/JSON
- **Authentication:** API key

### HERE Maps Platform
- **Description:** Alternative enterprise mapping and routing API with logistics-specific features including truck routing, fleet management, and isoline calculation for delivery zone modelling.
- **REST APIs:** https://www.here.com/developer/rest-apis
- **Routing API:** https://developer.here.com/documentation/routing-api/dev_guide/index.html
- **Geocoding & Search API:** https://developer.here.com/documentation/geocoding-search-api/dev_guide/index.html
- **Standards:** REST/JSON, OpenAPI-compatible
- **Authentication:** API key or OAuth 2.0

### FedEx Developer Portal (Reference Carrier API)
- **Description:** FedEx Shipping and Tracking APIs are a reference implementation of a carrier integration. A last-mile platform integrating with parcel carriers will use APIs like these for label generation, rate shopping, and tracking event ingestion.
- **Ship API Documentation:** https://developer.fedex.com/api/en-us/catalog/ship/docs.html
- **Track API:** https://developer.fedex.com/api/en-us/catalog/tracking/v1/docs.html
- **Standards:** REST/JSON, OpenAPI 3.0 spec published
- **Authentication:** OAuth 2.0 Client Credentials

---

## Notes

**Geocoding accuracy as a hidden dependency:** Route optimisation output quality depends heavily on geocoding precision. Address-level accuracy (rooftop vs. parcel centroid vs. street segment) directly determines whether the routing engine sequences stops in the correct physical order. Platforms relying solely on Google Maps Geocoding API should evaluate HERE and Geoapify for last-mile delivery address accuracy in dense urban and multi-unit residential contexts.

**EPCIS 2.0 adoption is accelerating:** EPCIS 2.0's JSON-LD support means delivery platforms can emit compliance-ready supply chain events to retail partners' traceability systems without bespoke integrations. Building EPCIS 2.0 event emission into the ePOD capture workflow provides a differentiating interoperability feature with minimal additional engineering.

**GDPR and driver location data remain underspecified in existing platforms:** No platform reviewed provides explicit tooling for GDPR-compliant location data lifecycle management (automatic purge of rest-period location records, data subject access request workflows). This is an emerging compliance gap as EU enforcement of proportionality in GPS tracking has intensified.

**MCP as dispatcher AI interface:** The Model Context Protocol is not yet adopted by any delivery platform reviewed, but represents a near-term opportunity to expose dispatch operations to LLM-powered copilots without building a proprietary natural-language layer. An MCP server surface is low-cost to implement and creates immediate compatibility with Claude, GPT-4o, and Gemini-based tooling ecosystems.
