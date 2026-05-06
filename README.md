# Last-Mile Delivery Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for orchestrating last-mile delivery — driver app, route optimisation, proof of delivery, and customer tracking in one stack.

The Last-Mile Delivery Platform is a logistics operating system for retailers, 3PLs, courier companies, and grocery, pharmacy, and restaurant chains who need to plan, dispatch, execute, and prove delivery of physical orders. It targets the most expensive segment of the supply chain — last-mile costs account for roughly 41% of total supply chain spend — with continuously re-optimising routing, AI-assisted ePOD verification, and hybrid fleet orchestration spanning owned drivers, 3PLs, and gig networks.

---

## Why Last-Mile Delivery Platform?

- Incumbent enterprise platforms (Bringg, Locus, FarEye) require significant professional services to implement and gate API documentation behind sales engagement, freezing out smaller operators.
- SMB-oriented tools (Onfleet, Routific, SmartRoutes, Spoke Dispatch) are easier to adopt but price scales steeply with stop volume and lock real-time tracking and ePOD behind higher tiers.
- Most route engines still rely on pre-shift batch optimisation; mid-day disruptions (failed stops, traffic, new orders) trigger manual reshuffling rather than continuous AI-driven re-routing.
- ePOD remains a trust gap: photo, signature, and geofence data are captured but rarely analysed for fraud or anomalous delivery claims.
- The only open-source entrant (Fleetbase) is AGPL-3.0 with a less mature route optimisation engine and no built-in multi-carrier network — leaving room for a permissively licensed, optimisation-first alternative.

---

## Key Features

### Routing & Dispatch

- Multi-stop route optimisation with time-window and vehicle-capacity constraints
- Continuous mid-route re-optimisation triggered by failed stops, new orders, and traffic
- Hybrid fleet support combining internal drivers with 3PL and carrier allocation
- Territory and zone management with custom-shape drawing
- Auto-dispatch driven by rules across cost, SLA, and driver capacity

### Driver Experience

- Native iOS and Android driver app with turn-by-turn navigation
- Stop workflow with barcode scan, photo, signature, and notes capture
- Two-way driver-dispatcher communication and exception handling
- Live driver GPS tracking surfaced on a dispatcher map view

### Proof of Delivery & Customer Communication

- Electronic proof of delivery: photo, signature, timestamp, and barcode scan
- Automated customer SMS and email notifications with live tracking links and ETAs
- Branded customer tracking portal
- Post-delivery customer satisfaction survey integrated into the ePOD flow

### Analytics & Sustainability

- Dispatcher analytics: on-time rate, completion counts, average stop time
- CO2 per-delivery reporting for ESG and procurement requirements
- OTIF (On Time In Full) compliance reporting

### Integration Surface

- REST API with webhook event stream for order ingestion and status push
- Multi-carrier connectors (FedEx, UPS, DHL as reference implementations)
- Connectors for OMS, WMS, ERP, and e-commerce platforms

---

## AI-Native Advantage

The platform replaces static batch route planning with continuous AI-driven re-optimisation that incorporates live traffic, failed-attempt signals, and driver capacity throughout the day. Predictive delivery-window estimation uses historical driver performance, address-level difficulty scores, and weather to push commitment accuracy beyond the 2-hour windows incumbents offer. AI-assisted ePOD fraud detection flags suspicious photos, geofence anomalies, and pattern deviations, while natural-language driver messaging lets dispatchers handle exception management at scale without per-incident phone calls. Automated carrier selection decides in real time whether each stop should be fulfilled by an owned driver, a gig platform, or a parcel carrier based on cost, SLA, and current capacity.

---

## Tech Stack & Deployment

The platform is designed for both self-hosted and cloud deployment, with a REST API plus webhook event stream as the primary integration surface — mirroring conventions established by Onfleet, Bringg, and Locus. GS1 barcode and RFID standards underpin package identification and ePOD scanning. OAuth 2.0 / JWT secures driver mobile apps and third-party integrations. The architecture targets API-first consumption so that OMS, WMS, ERP, and e-commerce systems can ingest delivery state without bespoke connectors.

---

## Market Context

Estimates of the last-mile delivery software market range from USD 7–32 billion in 2025 to USD 15–320 billion by 2032–2035 depending on scope (Market Research Future, 2025; Spherical Insights, 2025). Pricing varies widely: SMB tools start at $39–$200 per driver per month (SmartRoutes, Routific, Circuit), mid-market tools begin around $200/month (Route4Me), and enterprise platforms (Onfleet, Bringg, Locus, FarEye) are custom-priced by stop volume or fleet size. Primary buyers are logistics operations managers at e-commerce retailers, grocery and pharmacy chains, restaurant groups, 3PLs, and courier and parcel fleets.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
