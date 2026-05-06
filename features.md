# Cold Chain Monitoring — Feature & Functionality Survey

> Candidate #211 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Sensitech ColdStream (Carrier) | SaaS + hardware | Commercial / enterprise contract | https://www.sensitech.com |
| Tive Solo Pro | SaaS + hardware | Commercial / rental + subscription | https://www.tive.com |
| Controlant | SaaS + hardware | Commercial / subscription | https://www.controlant.com |
| Roambee / Decklar | SaaS + hardware | Commercial / subscription | https://www.roambee.com |
| Emerson Oversight | SaaS + hardware | Commercial / enterprise | https://climate.emerson.com |
| Tag-N-Trac RELATIVITY | SaaS + hardware | Commercial / custom pricing | https://tagntrac.ai |
| Geotab Cold Chain | Fleet telematics SaaS | Commercial / subscription | https://www.geotab.com |
| SafetyCulture | SaaS (inspection platform) | Commercial / SaaS tiers | https://safetyculture.com |
| Varcode Smart Tag | Hardware + SaaS analytics | Commercial / label + SaaS | https://www.varcode.com |
| Particle IoT Platform | IoT PaaS + hardware | Commercial / consumption-based | https://www.particle.io |
| OpenEPCIS | Open-source EPCIS server | Apache 2.0 | https://openepcis.io |

---

## Feature Analysis by Solution

### Sensitech ColdStream (Carrier)

**Core features**
- Wireless temperature, humidity, cryogenic, and door-ajar monitoring for warehouses and vehicles
- ColdStream Site Service: automated temperature documentation replacing manual logs
- TempTale Manager Desktop and ColdStream Select software for data download, display, analysis, and configuration
- SensiWatch platform for end-to-end in-transit real-time visibility
- Validated multi-modal IoT sensor integration
- Role-based access control and secure, redundant data management
- Automated real-time alerts and alarm management

**Differentiating features**
- Longest track record and deepest penetration in pharma cold chain
- Full cGXP, FDA 21 CFR Part 11 and EU Annex 11 validated compliance out of the box
- Carrier (global HVAC/refrigeration giant) as parent provides hardware-to-cloud continuity
- Covers both in-transit and stationary (warehouse) monitoring in one platform

**UX patterns**
- Enterprise portal login with role-based dashboards
- Report-centric UX (compliance reports, excursion summaries) rather than operational UX
- Manual configuration via desktop software (TempTale Manager) for sensor setup

**Integration points**
- ColdStream Reporting portal (web-based)
- Integration with ERP/QMS via file export and API (details not publicly documented)
- Hardware-centric ecosystem; limited third-party sensor support

**Known gaps**
- Expensive hardware lock-in; limited SMB or startup-friendly pricing
- No publicly documented REST/webhook developer API
- Limited AI-driven predictive capability
- Complex, legacy UX compared to newer entrants

**Licence / IP notes**
- Fully proprietary; hardware and software both owned by Carrier/Sensitech
- No open-source components exposed

---

### Tive (Solo Pro)

**Core features**
- Multi-sensor trackers: temperature (±0.5°C), humidity, light, shock, motion, tilt
- Cellular and satellite connectivity (no gateway required)
- ISO 17025-certified probes with minimal drift (<0.01°C/year)
- Real-time cloud dashboard with custom alert rules (ETA warnings, geofence breaches, temperature deviations, shock events)
- Webhooks for event-based push notifications
- REST API (v2 and v3) with CRUD endpoints for shipments and device data
- Custom header support and template-based webhook payloads (JSON, CSV, XML)
- Validated for pharmaceutical GDP and 21 CFR Part 11 compliance (Solo Pro)
- Multi-language mobile app

**Differentiating features**
- Largest display in the market (2.66-inch) on Solo Pro — operates reliably from −20°C to +60°C
- No gateway infrastructure needed (cellular/satellite direct)
- Full public developer portal with REST API and webhook documentation
- Both in-transit (logistics) and clinical trial use cases served from a single device family

**UX patterns**
- Self-service onboarding; devices arrive pre-configured
- Web dashboard and mobile app with map-based shipment views
- Alert fatigue mitigation via configurable rule thresholds

**Integration points**
- REST API v3 (developers.tive.com): shipments, alerts, devices, locations
- Webhooks with custom headers and payload templates
- ERP/TMS integration via API
- Cellular/satellite network agnostic

**Known gaps**
- Per-use cost model ($3–5/device/day) becomes expensive at scale
- AI-driven predictive excursion analysis not yet prominent
- Limited self-hosted or on-premises deployment option

**Licence / IP notes**
- Proprietary SaaS; REST API is public but access requires account
- No open-source SDK published

---

### Controlant

**Core features**
- Reusable BLE/cellular logger devices for pharma and food supply chains
- Real-time temperature, humidity, and location monitoring
- Excursion management: automated decision support (continue, quarantine, destroy)
- GxP-compliant audit trails and electronic signatures
- Quality reports auto-generated on shipment completion
- 24/7 monitoring and response services
- AI capabilities for supply chain risk insights

**Differentiating features**
- Reusable hardware reduces per-shipment cost for high-volume shippers
- Deeply embedded in pharma GDP workflows; used by global pharma 3PLs
- Managed services (24/7 monitoring operations centre) differentiate from pure-software platforms
- Zero-waste pharma supply chain positioning

**UX patterns**
- Workflow-oriented UX focused on GxP decision-making
- Quality report generation tied to shipment lifecycle events
- Operator and QA role separation in UI

**Integration points**
- ERP, QMS, and TMS integrations (enterprise; specific APIs not publicly documented)
- Partner ecosystem with major pharma 3PLs
- Possible EPCIS-compatible data export

**Known gaps**
- Pricing not transparent; no SMB tier
- Limited public developer API documentation
- Primarily pharma-focused; food/retail cold chain capabilities less developed

**Licence / IP notes**
- Proprietary; reusable hardware and software both owned by Controlant

---

### Roambee / Decklar

**Core features**
- Multi-modal sensor deployment (BLE beacons, cellular, LoRa depending on use case)
- AI-powered supply chain visibility: OTIF scoring, quality compliance (QC), dwell time, TAT insights
- Supply risk prediction at individual nodes (airports, ports, transshipment points)
- Cold chain lane analysis: identifies the 20% of lanes causing 60%+ of excursions
- Integration with carrier data aggregation platforms, port/airport operations, freight forwarder feeds
- Real-time location and condition data fused with order information
- API platform for third-party system integration

**Differentiating features**
- Strongest AI/ML analytics layer among competitors — quantified outcomes (70% better multimodal ETAs, 80%+ cold chain compliance)
- Multi-modal data fusion (IoT + non-IoT: flight data, vessel AIS, port dwell data)
- Acquired Modum (Swiss pharma cold chain specialist) for pharma depth
- Works without owned hardware — asset-light deployments using existing carrier or 3PL data streams
- Lane-level root cause analysis distinguishing systemic from one-off excursions

**UX patterns**
- Data science / analytics-forward interface; BI-style dashboards
- SLA prediction cards by lane and carrier
- Heavy use of map visualisation with risk overlays

**Integration points**
- REST API for third-party app integration
- Carrier and 3PL data platform connections
- Supports existing logistics systems and legacy ERP

**Known gaps**
- Complex onboarding; requires data mapping and carrier integration setup
- No public developer portal or published API reference
- Small and mid-size shipper adoption limited by implementation complexity

**Licence / IP notes**
- Proprietary AI platform; no open-source components
- Modum technology folded into Roambee

---

### Emerson Oversight

**Core features**
- In-transit temperature, location, humidity, and light monitoring via GO Real-Time cellular trackers
- Automated reporting: historical temperature and location reports, proof-of-delivery records
- Cargo security alerts (door open, tamper detection)
- Multi-language support (10 languages including English, Chinese, Spanish, German, Russian, French)
- Oversight Mobile app for field use
- Single sign-on (SSO) support (added in Oversight 2)
- Foodservice digital food safety: temperature probing devices for prepared food compliance

**Differentiating features**
- Strong retail and foodservice vertical specialisation
- Physical integration with Emerson/Copeland refrigeration control systems for warehouse use
- Multi-language support suited for global carrier networks

**UX patterns**
- Report-and-alert centric; operational use case for carrier/broker
- Mobile app for field and last-mile visibility

**Integration points**
- SSO (SAML / OAuth integration implied)
- Emerson refrigeration hardware ecosystem
- File-based or API export (specific API not publicly documented)

**Known gaps**
- Less suited for pharmaceutical/GDP-regulated workflows
- Limited AI/predictive analytics
- No public developer API or webhook documentation
- Aging UX compared to Tive and Roambee

**Licence / IP notes**
- Proprietary Emerson product; Oversight 2 is current version

---

### Tag-N-Trac RELATIVITY

**Core features**
- BLE + 5G cellular smart labels and tags for in-warehouse and in-transit tracking
- SKU-to-container level granularity within a single platform
- Continuous environmental monitoring: temperature, humidity, tamper/shock alerts, chain of custody
- RELATIVITY platform: processes over 250,000 cold chain shipments/year, 3+ billion data points/year
- Backward compatibility with existing logistics systems
- Airline-approved BLE and 5G cargo trackers
- Disposable smart labels (2025 product line) for low-cost per-shipment use

**Differentiating features**
- Single platform spanning in-warehouse and in-transit (industry first claimed)
- Disposable smart label format at a fraction of tracker cost
- Identiv partnership expanding into pharmaceutical compliance market
- 5G direct cellular eliminates gateway dependency

**UX patterns**
- Supply chain intelligence dashboard converting raw data points to actionable alerts
- Fast implementation positioning; backward-compatible integration

**Integration points**
- Device-agnostic sensor integration
- Backward-compatible with existing logistics systems
- API available (specific documentation not publicly accessible)

**Known gaps**
- Smaller vendor; more limited ecosystem than Sensitech or Tive
- AI analytics less mature than Roambee
- Public developer documentation sparse

**Licence / IP notes**
- Proprietary hardware and software platform

---

### Geotab Cold Chain

**Core features**
- Real-time temperature monitoring from refrigeration units (Thermo King, Carrier, etc.) via IOX-COLD hardware
- Multi-zone temperature monitoring (supports multi-temperature loads)
- Custom alert rules with automatic setpoint-based adjustment
- Proof of delivery reporting
- Hardware error code detection and remote diagnostics
- Historical data: interactive graphs, grids, and maps for trend analysis
- Cloud integration with Thermo King TracKing platform

**Differentiating features**
- Deep OEM refrigeration unit integration (native reefer unit data vs. ambient sensor readings)
- Fleet telematics + cold chain in one platform — driver behaviour, routing, and temperature in one view
- IOX-COLD RUGGED (IP67) for external mounting on trailer units
- Part of broader Geotab fleet management ecosystem with 50,000+ customer base

**UX patterns**
- Fleet management console (MyGeotab) with cold chain module tab
- Add-in architecture within existing fleet software; no separate login
- Alert rules configured within existing Geotab rule engine

**Integration points**
- Geotab Marketplace (ecosystem of 300+ add-ins and integrations)
- ColdChain BI connector for Power BI analytics
- IOX accessory bus for hardware expansion
- SDK and REST API via MyGeotab developer platform

**Known gaps**
- Requires Geotab fleet management as prerequisite (not standalone cold chain solution)
- Not suited for pharmaceutical GDP compliance workflows
- Primarily transport-focused; limited warehouse/storage coverage

**Licence / IP notes**
- Proprietary; Geotab Open Platform SDK is publicly available for fleet integrations

---

### SafetyCulture (iAuditor)

**Core features**
- Digital cold chain checklists (HACCP, daily temperature logs, receiving inspections)
- Continuous temperature sensor monitoring with configurable threshold alerts
- Automated temperature log recording for commercial refrigeration
- HACCP compliance digitisation: audit trails, action item assignment
- Training management for cold chain staff
- Sensor integration: Bluetooth, WiFi, and wired temperature/humidity sensors
- Mobile-first design for frontline workers

**Differentiating features**
- Broadest checklist/inspection template library (industry templates for HACCP, FSMA, GDP)
- Most affordable and SMB-accessible platform in the space ($0–$24/user/month)
- Not hardware-locked: integrates with diverse sensor hardware
- Combined sensor monitoring + operational inspection/audit in one product

**UX patterns**
- Mobile-first, designed for operational/frontline workers not logistics managers
- Template-driven inspection workflows
- Guided action items and corrective action tracking
- App-centric with web dashboard for managers

**Integration points**
- REST API for data export and integration
- Zapier/Make connectors
- Slack, Microsoft Teams notifications
- Webhooks for audit events

**Known gaps**
- Not suited for pharmaceutical-grade GxP compliance (no 21 CFR Part 11 validation)
- Limited transport/in-transit monitoring; primarily stationary (warehouse, store)
- No predictive analytics or AI-driven excursion detection

**Licence / IP notes**
- Commercial SaaS; free tier available for small teams
- No open-source components

---

### Varcode Smart Tag

**Core features**
- Temperature indicator barcode labels with dynamically shifting barcode pattern as temperatures change
- Smart Tag Management System (STMS) for cloud-based label tracking
- Analytics dashboard for dock-to-doorstep cumulative temperature data
- Email and text alerts when shipments exceed temperature thresholds
- Mobile scanning via iOS/Android (standard barcode readers or smartphone)
- Supports time-temperature integration for cumulative exposure tracking

**Differentiating features**
- Label-form-factor: no active electronics required for basic read — passive indicator with digital audit capability
- Cumulative thermal exposure model (not just threshold alerts) enables shelf-life estimation
- Compatible with existing barcode scanning infrastructure
- Very low per-unit cost compared to cellular trackers

**UX patterns**
- Scan-at-dock UX: workers scan labels at transfer points; platform aggregates readings
- Dashboard for logistics managers to review excursion history
- Alert-based workflow: email/SMS triggered on threshold breach

**Integration points**
- Mobile barcode scanning API (iOS/Android SDK implied)
- Cloud analytics dashboard
- Email/SMS alert integrations

**Known gaps**
- Passive scan-based model misses real-time continuous monitoring between scan points
- No GPS location tracking
- Not suitable for pharmaceutical GxP validation workflows
- Limited API/developer documentation publicly available

**Licence / IP notes**
- Proprietary patented technology (barcode shifting mechanism is patented)

---

### Particle IoT Platform

**Core features**
- IoT PaaS with cellular, WiFi, and LoRa hardware modules
- Asset Tracker system with GNSS + dead-reckoning, temperature sensor integration
- Open firmware application framework extensible with off-the-shelf IoT sensors
- Fleet-wide configuration service (reporting interval, geofence, etc.)
- REST API for device state queries and command dispatch
- Webhook support for third-party integrations
- Particle Device Cloud with integrations marketplace (AWS IoT, Azure IoT Hub, Google Cloud IoT, ThingsBoard, etc.)
- Consumption-based pricing suited for developer/startup deployments

**Differentiating features**
- Strongest developer-first platform in the space; full open firmware framework
- Hardware-agnostic sensor addition via modular firmware hooks
- Multi-network connectivity (cellular, WiFi, LoRa on one platform)
- Ready-to-use cold chain tracking reference firmware available openly

**UX patterns**
- Developer-centric: firmware IDE, API console, device management portal
- Not an end-user cold chain product but a platform for building one
- Progressive disclosure from prototype to production fleet management

**Integration points**
- REST API (device management, data ingestion)
- Webhook to any HTTP endpoint
- Native integrations: AWS IoT, Azure IoT Hub, Google Cloud, Zapier, ThingsBoard
- Serial/I2C/SPI sensor bus for hardware expansion

**Known gaps**
- Not a turnkey cold chain solution; requires development effort
- No built-in compliance (21 CFR Part 11, GDP) features
- No excursion management or regulatory reporting out of the box
- UI/dashboard must be built or sourced separately

**Licence / IP notes**
- Particle Device Cloud: commercial SaaS
- Particle firmware is largely open-source (Apache 2.0 for device OS)
- Tracker firmware reference is open-source

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Real-time or near-real-time temperature monitoring with configurable alert thresholds
- Historical temperature data storage with downloadable/exportable logs
- Email and/or SMS/push notification alerts on excursions
- Web dashboard and mobile app access
- Multi-user account management with role-based permissions
- Support for multiple temperature ranges (frozen, refrigerated, ambient-controlled)
- Chain-of-custody timestamp logging at transfer points

### Differentiating Features
- Pharmaceutical-grade validation: 21 CFR Part 11 and EU Annex 11 compliance with audit trails and electronic signatures
- Predictive/AI analytics: lane performance scoring, risk prediction, excursion forecasting before breach
- Multi-sensor fusion: temperature + humidity + shock + light + location in one tracker
- No-gateway cellular/satellite connectivity for true plug-and-play deployment
- Fleet telematics integration (combining vehicle tracking with cargo condition)
- Cumulative thermal exposure modelling for dynamic shelf-life estimation
- Managed services (24/7 monitoring operations centre)

### Underserved Areas / Opportunities
- **SMB-accessible pharma compliance**: 21 CFR Part 11-compliant solutions are all enterprise-priced; no SMB-friendly validated platform exists
- **Cross-platform data portability**: no vendor supports EPCIS 2.0 natively as an export format; data is siloed per vendor
- **Predictive excursion prevention**: most platforms detect excursions after they start; true prediction (hours before breach) based on ambient and route conditions is rare
- **Open-source reference stack**: no production-ready open-source platform combining sensor ingestion, excursion management, and compliance reporting
- **Dynamic shelf-life integration**: linking temperature history to actual product shelf-life models (beyond Varcode's passive label approach) is absent from mainstream platforms
- **LLM-assisted compliance reporting**: auto-drafted GDP deviation investigation reports and excursion assessments have no native AI implementation in any platform
- **Multi-standard compliance in one platform**: covering FDA, EU GDP, WHO vaccine guidelines, and FSMA simultaneously requires manual bridging in every current solution

### AI-Augmentation Candidates
- **Excursion prediction**: ML models on historical sensor + weather + route data can forecast breaches hours in advance
- **Lane/carrier risk scoring**: AI can rank lanes by chronic excursion risk so quality teams focus audits effectively
- **Alert noise reduction**: ML classification distinguishing genuine excursions from door-open events, defrost cycles, and sensor noise
- **Compliance report generation**: LLMs can auto-draft deviation investigation reports, MBR sections, and GDP-compliant excursion summaries
- **Packaging recommendation**: AI can suggest optimal packaging configurations based on route profile, ambient forecast, and product sensitivity
- **Anomaly detection on equipment health**: predictive maintenance for refrigeration units based on compressor telemetry patterns

---

## Legal & IP Summary

No open-source licensing concerns were identified for a new AI-native cold chain monitoring platform. The core sensor communication protocols (MQTT, AMQP, CoAP) are open standards. GS1 EPCIS 2.0 (ISO/IEC 19987:2024) is a freely implementable standard. Varcode holds patents on its specific shifting-barcode temperature indicator mechanism; a new platform should avoid replicating this precise encoding approach. Sensitech/Carrier and Tive hold patents on specific hardware designs but the software platform concept is unencumbered. The 21 CFR Part 11 compliance framework is a regulatory requirement, not a proprietary standard. No copyright or patent concerns were identified that would prevent development of an open-source cold chain monitoring platform using standard IoT protocols, EPCIS data models, and AI-driven analytics.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Real-time sensor data ingestion via MQTT/HTTP from BLE, cellular, and LoRa sensors
- Configurable temperature and humidity alert thresholds with email/SMS/webhook notifications
- Historical excursion log with exportable PDF and CSV reports
- Multi-user role-based access (admin, quality, logistics, read-only)
- Shipment lifecycle tracking (created → in transit → delivered) with chain-of-custody timestamps
- REST API with OpenAPI 3.1 specification for third-party integration

**Should-have (v1.1)**
- EPCIS 2.0 (ISO/IEC 19987:2024) event export for supply chain interoperability
- AI-driven excursion prediction (ML model on sensor + route + ambient data)
- LLM-assisted compliance report generation (GDP/FDA excursion deviation reports)
- 21 CFR Part 11-compliant audit trail module (electronic signatures, tamper-evident logs)
- Multi-sensor support: temperature + humidity + shock + light in unified event model
- Lane/carrier performance analytics dashboard

**Nice-to-have (backlog)**
- Dynamic shelf-life estimation module (cumulative thermal exposure model per product SKU)
- Computer vision integration at receiving docks (packaging damage detection)
- Blockchain-anchored immutable audit trail option
- In-app HACCP critical control point checklist workflows
- Packaging configuration recommender (route-aware, AI-driven)
- Predictive maintenance alerts for refrigeration equipment (compressor telemetry analysis)
