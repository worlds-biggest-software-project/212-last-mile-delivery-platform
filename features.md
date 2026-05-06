# Last-Mile Delivery Platform — Feature & Functionality Survey

> Candidate #212 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Onfleet | SaaS | Commercial (custom/tiered) | https://onfleet.com |
| Bringg | SaaS | Commercial (enterprise custom) | https://www.bringg.com |
| Locus DispatchIQ | SaaS | Commercial (enterprise custom) | https://locus.sh |
| Route4Me | SaaS | Commercial (from ~$200/mo) | https://route4me.com |
| Routific | SaaS | Commercial (from ~$150/mo) | https://routific.com |
| Spoke Dispatch (fmr. Circuit) | SaaS | Commercial (stop-volume tiers) | https://spoke.com/dispatch |
| SmartRoutes | SaaS | Commercial (from ~$39/driver/mo) | https://smartroutes.io |
| FarEye | SaaS | Commercial (enterprise custom) | https://fareye.com |
| DispatchTrack | SaaS | Commercial (enterprise custom) | https://dispatchtrack.com |
| Fleetbase | Open Source / Commercial | AGPL-3.0 + commercial licence | https://fleetbase.io |

---

## Feature Analysis by Solution

### Onfleet

**Core features**
- AI-powered route optimisation trained on 400M+ deliveries, accounting for traffic, capacity, and time windows
- Auto-dispatch: automatically assigns tasks to available drivers based on rules
- Real-time driver tracking with live map dashboard
- Electronic proof of delivery (ePOD): photos, barcode scans, signatures, age verification, delivery notes
- Customer notifications: automated SMS/email with live tracking link and ETA
- Driver mobile app (iOS and Android) with turn-by-turn navigation
- RESTful API with webhook support and multi-language SDKs (Python, Node, PHP, Go, Ruby, Java)
- Analytics and reporting: on-time rate, average completion time, stop counts

**Differentiating features**
- API quality described as rivalling Stripe's for completeness and reliability — a strong developer-first reputation
- Continuous mid-route re-optimisation (not just batch planning)
- Supports hybrid fleet: internal drivers plus third-party carrier handoff in one workflow

**UX patterns**
- Web dashboard for dispatchers; responsive but lacks a native mobile manager app
- Driver app is the primary mobile experience; clean onboarding flow
- Progressive disclosure: simple default view with advanced filtering available

**Integration points**
- REST API + webhooks; SDKs for Python, Node.js, PHP, Go, Ruby, Java
- Pre-built integrations with Shopify, WooCommerce, and Square
- Zapier connector for no-code workflow integration
- OMS, WMS, and POS integrations via API

**Known gaps**
- No native mobile dashboard for dispatchers (responsive web only)
- Route optimisation routing flaws reported when pushing orders from certain POS systems
- Dashboard can slow under heavy concurrent filters
- Pricing scales steeply at high stop volumes

**Licence / IP notes**
- Proprietary SaaS. API documentation is public. Open-source SDK wrappers are MIT-licensed.

---

### Bringg

**Core features**
- Multi-carrier orchestration: single integration connecting 250+ carriers across 70 countries
- Dynamic delivery slot booking: precise ETA windows at checkout based on real-time data
- Real-time fleet map combining owned drivers and third-party carriers
- Automated dispatch based on cost, driver availability, SLAs, and labour skills
- Customer self-service scheduling and rescheduling via SMS/email
- Exception management with custom alert rules and two-way driver communication
- Automated customer notifications (SMS, email, in-app) at each delivery milestone
- Enterprise-grade security: SOC 2, MFA, SSO, Google Cloud infrastructure

**Differentiating features**
- Dynamic Delivery Slots at checkout: eCommerce retailers can surface precise delivery windows to shoppers before purchase, improving cart conversion
- Single API connecting 250+ last-mile carriers — unmatched carrier network breadth
- Modular platform architecture allows selective module adoption without full platform lock-in

**UX patterns**
- Designed for enterprise operations teams; complex initial configuration
- Modular onboarding: customers activate only the modules they need
- Driver app customisable per workflow (inventory, signature, photo, etc.)

**Integration points**
- Open REST APIs and webhooks
- Pre-built connectors for OMS, ERP, eCommerce (Salesforce, SAP, Magento), and WMS
- Developer portal at developers.bringg.com with API reference and webhook docs

**Known gaps**
- Complex to implement; significant professional services engagement usually required
- Less suitable for SMBs due to enterprise pricing and setup complexity
- Carrier breadth can create configuration overhead for focused operations

**Licence / IP notes**
- Proprietary enterprise SaaS. No open-source components disclosed.

---

### Locus DispatchIQ

**Core features**
- Constraint-aware AI dispatch across 250+ variables (capacity, time windows, vehicle type, SLAs)
- Dynamic mid-route re-optimisation triggered by cancellations, traffic, or new orders
- TrackIQ real-time visibility layer with geo-fenced alerts and proactive exception notifications
- ShipFlex multi-carrier orchestration: dynamic allocation across 3PL partners by cost, speed, SLA
- Sustainability-aware routing: balances speed with emission intensity and load efficiency
- CO2 reporting per route and per stop
- Full API suite for TMS, WMS, OMS, and ERP integration

**Differentiating features**
- Operates at the top of the "Last Mile Orchestration Maturity Model" with AI across 250+ dispatch variables
- Sustainability-first routing engine as a first-class feature, not a bolt-on
- Dedicated ShipFlex module for real-time carrier selection across 3PL networks

**UX patterns**
- Enterprise-oriented dashboard; designed for operations managers overseeing large fleets
- Analytics-heavy interface with drill-down reporting
- API-first design allows deep OMS/ERP embedding

**Integration points**
- Full REST API suite; docs at docs.locus.sh (access requires client credentials)
- Plug-and-play connectors for SAP, Oracle, Salesforce, and major WMS/TMS platforms
- Webhook event stream for real-time order and dispatch status

**Known gaps**
- API documentation not publicly accessible — requires client relationship to integrate
- Primarily serves mid-to-large enterprise; limited SMB pricing or self-serve onboarding
- Limited North American market penetration vs. Asia/Middle East

**Licence / IP notes**
- Proprietary SaaS. No open-source components disclosed.

---

### Route4Me

**Core features**
- Multi-stop route planning and optimisation (100% RESTful routing engine)
- Territory-based routing: custom-shaped geographic territories with address book mapping
- Real-time GPS driver tracking and route progress monitoring
- Customer notifications with accurate arrival times
- Telematics gateway: bi-directional integration with fleet telematics (Geotab, Verizon, Samsara)
- API marketplace with CRM, ERP, and e-commerce integrations
- SDKs for C#, VB.NET, Python, Java, Node, C++, Ruby, PHP, Go, Erlang, Perl, Bash, and VBScript

**Differentiating features**
- Broadest SDK language support in the category (14+ languages including Erlang and VBScript)
- Territory optimisation as a first-class feature — well-suited to field sales alongside delivery
- Telematics gateway normalises data from multiple GPS hardware vendors into one database

**UX patterns**
- Web interface described as functional but dated; extensive feature surface can overwhelm new users
- Address book and territory tools provide strong medium-term operational structure
- Mobile driver app provides turn-by-turn navigation and stop confirmation

**Integration points**
- Publicly documented REST API at route4me.io/docs
- SDKs for 14+ languages on GitHub
- Telematics Gateway for fleet hardware integration
- Zapier and marketplace third-party integrations

**Known gaps**
- UI perceived as dated compared to newer entrants
- Advanced analytics and reporting less sophisticated than enterprise platforms
- Pricing and feature bundling can be confusing

**Licence / IP notes**
- Proprietary SaaS. SDKs are open source (Apache 2.0 licence on GitHub).

---

### Routific

**Core features**
- Route optimisation focused on quality: tighter clusters and fewer crossed paths than most competitors
- Order import and batch route planning with time windows and driver constraints
- Live driver tracking and progress monitoring throughout the delivery day
- Customer notifications with ETA updates
- Webhook events for real-time driver-ground status data
- REST API (Engine API and Platform API) with Bearer token authentication
- Free tier (up to 100 orders/month)

**Differentiating features**
- Consistently praised route quality on complex multi-stop routes — fewer criss-crossing paths
- Free entry tier lowers SMB adoption barrier
- Platform API designed for automated order ingestion in real time as orders flow in

**UX patterns**
- Clean, simple interface optimised for smaller operations
- Manager dashboard and driver app have minimal learning curve
- Basic plan withholds live tracking and ePOD — features gated behind paid tier

**Integration points**
- REST Engine API at docs.routific.com
- Platform API at routific-platform.readme.io
- Webhook support for real-time driver event streaming
- Does not publish a broad SDK ecosystem; integration primarily via REST

**Known gaps**
- Real-time tracking and proof of delivery locked to higher-tier plans
- Struggles with multiple clustered stops and very complex constraint sets
- No native multi-carrier or crowdsourced delivery network support
- Smaller integration ecosystem than Onfleet or Route4Me

**Licence / IP notes**
- Proprietary SaaS. API documentation publicly accessible.

---

### Spoke Dispatch (formerly Circuit for Teams)

**Core features**
- Browser-based dispatcher planning interface with drag-and-drop stop management
- Multi-stop route planning and sequencing
- Live driver tracking
- Proof of delivery (photo, signature)
- Customer notifications
- In-vehicle package-finder feature on driver mobile app
- Stop-volume subscription billing model

**Differentiating features**
- In-app package-finder tool: helps drivers locate packages in their vehicle by stop number — reduces per-delivery time
- Rebranded under Spoke brand in late 2025; positioning towards broader logistics suite

**UX patterns**
- Clean, intuitive interface cited as easiest onboarding in the SMB segment
- Driver app designed for low-skill users; minimal configuration required
- Emphasis on simplicity over advanced feature depth

**Integration points**
- REST API (documentation on spoke.com/dispatch)
- Zapier integration available
- Fewer native enterprise integrations than Onfleet or Bringg

**Known gaps**
- Less advanced route optimisation than Routific for complex stop sets
- Limited enterprise system integrations
- Fewer analytics and reporting capabilities
- No multi-carrier orchestration

**Licence / IP notes**
- Proprietary SaaS. No open-source components disclosed.

---

### SmartRoutes

**Core features**
- Route planning and optimisation dashboard (web)
- Live vehicle and order tracking
- Electronic proof of delivery (photo, signature, notes)
- Driver mobile app (iOS and Android)
- Customer notification portal with branded tracking links
- Territory-based delivery zone management
- Per-driver pricing model (~$39/driver/month)

**Differentiating features**
- Most affordable entry price per driver in the category
- Full-suite tool (route planning + ePOD + tracking + notifications) at SMB price point
- Branded customer tracking portal without upcharge

**UX patterns**
- Straightforward setup; oriented toward owner-operators and small fleets
- Less configuration overhead than enterprise platforms
- Mobile app prioritises simplicity over depth

**Integration points**
- API available (limited public documentation)
- WooCommerce and Shopify integrations
- Fewer enterprise connectors than larger platforms

**Known gaps**
- Limited enterprise-grade integrations (no native SAP, Salesforce, Oracle connectors)
- Less sophisticated route optimisation for large fleets
- Analytics and reporting relatively basic

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### FarEye

**Core features**
- End-to-end delivery orchestration: first, middle, and last mile in one platform
- Contactless ePOD with digital signatures and photo capture
- Branded customer tracking portal with real-time ETA updates via SMS and email
- Automated notifications reducing WISMO (Where Is My Order) calls
- Two-way driver-dispatcher-support communication with exception management
- Compliance and sustainability metrics reporting

**Differentiating features**
- Strongest coverage of the retail-adjacent verticals: grocery, pharmacy, QSR
- Full multi-mile orchestration (first + middle + last) in one platform
- Branded experience layer: customer-facing tracking portal is highly configurable

**UX patterns**
- Enterprise-oriented with vertical-specific workflows pre-configured
- Driver app with customisable task flows per delivery type
- High configuration investment at initial deployment

**Integration points**
- API key-authenticated integrations (support team coordinates initial setup)
- Motive (GoMotive) telematics integration documented
- Connectors for enterprise OMS, WMS, and ERP systems

**Known gaps**
- Less suited to small and mid-size businesses
- API integration requires engagement with FarEye's support team (not self-serve)
- Limited North American market presence relative to Asian and European footprint

**Licence / IP notes**
- Proprietary SaaS. No open-source components disclosed.

---

### DispatchTrack

**Core features**
- Delivery management specialised for big-and-bulky items (furniture, appliances, medical equipment)
- AI-powered CO2 tracking: per-route, per-stop, per-vehicle carbon emissions data
- Automated customer scheduling, confirmation, and rescheduling via text, email, or phone
- Post-delivery survey tool for customer satisfaction measurement
- Route optimisation with capacity and service-time constraints
- Digital proof of delivery (photo, signature, timestamp)

**Differentiating features**
- Industry's first AI-powered CO2 tracking integrated into route and reporting screens
- Deep big-and-bulky delivery workflows: handles multi-item, multi-person service deliveries
- Automated pre-delivery customer outreach (scheduling confirmation prompts)

**UX patterns**
- Vertically focused: onboarding assumes furniture/appliance delivery context
- CO2 data surfaces in standard routing and reporting screens (not a separate module)
- Post-delivery survey integrated into driver app completion flow

**Integration points**
- REST API for OMS, WMS, and ERP integration
- Spoke.com integration review references DispatchTrack's API capabilities
- Carrier and 3PL integration connectors

**Known gaps**
- Vertical focus limits applicability outside big-and-bulky use cases
- Less breadth of carrier integrations than Bringg or Locus
- Smaller developer ecosystem and SDK coverage

**Licence / IP notes**
- Proprietary SaaS. No open-source components disclosed.

---

### Fleetbase

**Core features**
- Modular open-source logistics and supply chain operating system (LSOS)
- REST API with WebSocket support for real-time data
- Webhook triggers for event-driven automation
- Driver mobile app (Navigator) with turn-by-turn navigation
- Order, vehicle, and driver resource management
- Extension marketplace for adding capabilities (or building and publishing custom extensions)
- Deployable on-premise or in cloud
- API key management and developer tooling built into the admin console

**Differentiating features**
- Only open-source platform in the category with a full logistics operating system scope
- Extension marketplace model: community-buildable, self-hostable feature additions
- AGPL-3.0 with commercial licence option — allows proprietary forks under commercial terms
- Event-driven architecture with WebSocket + webhook as first-class infrastructure

**UX patterns**
- Developer-first: primary audience is engineering teams building logistics products
- Admin console with API log viewer and key management
- Less polished consumer-facing UX compared to commercial SaaS tools

**Integration points**
- Full REST API documented at docs.fleetbase.io
- WebSocket events for real-time updates
- Postman collection available
- IDB (Inter-American Development Bank) has catalogued it as an open-source supply chain solution

**Known gaps**
- Route optimisation engine is less mature than commercial specialists
- Smaller community and ecosystem compared to established commercial platforms
- AGPL licence creates complications for commercial derivative products (requires commercial licence purchase)
- No built-in multi-carrier network comparable to Bringg's 250+ carrier connections

**Licence / IP notes**
- AGPL-3.0 by default. Commercial licence (FCL) available from Fleetbase for proprietary use. Derivative works under AGPL must be released under the same licence.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-stop route optimisation with time-window constraints
- Real-time driver GPS tracking with dispatcher map view
- Driver mobile app (iOS + Android) with turn-by-turn navigation
- Electronic proof of delivery: photo capture, signature, and timestamp
- Automated customer notifications (SMS and/or email) with ETA and tracking link
- RESTful API with webhook support for order and status integration
- Basic analytics: on-time rate, completion counts, stop duration

### Differentiating Features
- Continuous mid-route re-optimisation (not just pre-shift batch planning)
- Multi-carrier / hybrid fleet orchestration combining owned drivers, 3PLs, and gig networks
- Dynamic delivery slot booking at checkout (Bringg speciality)
- AI-powered constraint dispatch with 250+ variables (Locus)
- CO2 emissions tracking per route and per stop (DispatchTrack, Locus)
- ePOD fraud detection and anomaly flagging
- In-vehicle package-finder tool (Spoke Dispatch)
- Territory management with custom-shaped geographic zones (Route4Me)
- Vertical-specific workflows: big-and-bulky (DispatchTrack), grocery/QSR (FarEye)

### Underserved Areas / Opportunities
- Self-serve API onboarding for SMBs without enterprise contracts (Locus, FarEye APIs require sales engagement)
- AI-assisted fraud detection on ePOD submissions (photo analysis for false delivery claims)
- Predictive address-level delivery difficulty scoring to improve ETA accuracy beyond 2-hour windows
- Natural-language dispatcher-to-driver communication for exception handling at scale
- Real-time crowdsourced/gig platform integration with dynamic selection logic (cost vs. SLA vs. availability)
- Open-source route optimisation engine competitive with commercial quality
- Unified CO2 reporting across multi-carrier and own-fleet deliveries for ESG compliance
- Driver rest-period location data removal compliant with GDPR proportionality requirements (few platforms handle this explicitly)

### AI-Augmentation Candidates
- Route re-optimisation: replacing static batch planning with continuous AI-driven adjustment
- Delivery window prediction: using historical driver performance, address difficulty, and weather signals
- ePOD anomaly detection: image analysis and geo-fence validation to flag suspicious delivery confirmations
- Carrier selection: automated cost/SLA/capacity triage across owned fleet, 3PLs, and gig networks
- Exception triage: AI-routed escalation of failed deliveries and driver communication without dispatcher calls
- Demand forecasting to pre-position drivers ahead of high-density delivery windows

---

## Legal & IP Summary

No patent encumbrances on the core feature set were identified. Route optimisation algorithms are well-studied operations research problems (vehicle routing problem variants) with extensive public academic literature; no evidence of proprietary patent barriers was found. Commercial platforms are proprietary SaaS products — their software cannot be reused, but their feature patterns, API conventions, and UX approaches are freely observable and replicable. The one open-source platform (Fleetbase) uses AGPL-3.0: any product that embeds or modifies Fleetbase code and distributes it must release source under AGPL, or must obtain a commercial licence from Fleetbase. An independent greenfield build would not be subject to this restriction. GS1 standards (SSCC, EPCIS, GS1-128 barcode encoding) are openly specified and freely implementable. No copyright or licensing concerns were identified that would prevent building an independent platform inspired by this feature landscape.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Multi-stop route optimisation with time-window and vehicle-capacity constraints
- Real-time driver tracking with dispatcher map view
- Driver mobile app (iOS and Android) with navigation and stop workflow
- Electronic proof of delivery: photo capture, signature, timestamp, and barcode scan
- Automated customer SMS/email notifications with live tracking link and ETA
- REST API with webhook event stream for order ingestion and status push
- Basic dispatcher analytics: on-time rate, completion counts, average stop time

**Should-have (v1.1)**
- Continuous mid-route re-optimisation (triggered by failed stops, new orders, traffic)
- Hybrid fleet support: internal drivers plus 3PL/carrier allocation in one workflow
- AI-based ETA prediction using historical driver performance and address-level signals
- CO2 per-delivery reporting for ESG and procurement requirements
- Multi-carrier integration connectors (at least FedEx, UPS, DHL as reference implementations)

**Nice-to-have (backlog)**
- ePOD fraud detection: ML image analysis and geo-fence anomaly flagging
- Natural-language driver messaging for dispatcher exception management
- Dynamic delivery slot booking surfaced at checkout via merchant embed
- Territory and zone management with custom-shape drawing
- Post-delivery customer satisfaction survey integrated into ePOD completion flow
- Extension/plugin architecture for vertical-specific workflow customisation
