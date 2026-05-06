# Standards & API Reference

> Project: Cold Chain Monitoring · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISO Standards

**ISO 31512:2024 — Cold Chain Logistics Services (B2B) — Requirements and Guidelines for Storage and Transport**
- URL: https://www.iso.org/standard/83684.html
- Published December 2024 as the world's first international standard for cold chain logistics services. Specifies requirements for refrigerated storage and transport of foods in B2B logistics, covering temperature maintenance, hygiene management, and quality controls. A new platform should align its shipment and storage data models with the requirements defined in this standard.

**ISO 23412:2020 — Indirect Temperature-Controlled Refrigerated Delivery Services — Land Transport of Parcels**
- URL: https://www.iso.org/standard/75468.html
- Specifies requirements for the provision of indirect refrigerated delivery services for temperature-sensitive goods by land. Relevant to parcel-level cold chain monitoring, defining minimum requirements for vehicles, cold stores, staff, and temperature monitoring documentation.

**ISO 31510:2025 — Cold Chain Logistics — Vocabulary**
- URL: https://www.iso.org/standard/84041.html
- Published 2025; provides standardised terminology for cold chain logistics. A cold chain monitoring platform should adopt the vocabulary defined here to ensure interoperability and clear communication with logistics partners and regulators.

**ISO/TC 315 — Cold Chain Logistics Technical Committee**
- URL: https://www.iso.org/committee/6880159.html
- The ISO technical committee responsible for all cold chain logistics standards (including ISO 23412, ISO 31510, ISO 31512). Monitoring this committee's work programme identifies upcoming standards relevant to the platform.

---

### Regulatory Frameworks

**FDA 21 CFR Part 11 — Electronic Records and Electronic Signatures**
- URL: https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11
- US FDA regulation governing electronic records and electronic signatures in regulated industries. Mandatory for pharmaceutical cold chain platforms operating under US regulations. Requires audit trails, access controls, tamper-evident logs, and validated software.

**EU GMP Annex 11 — Computerised Systems**
- URL: https://health.ec.europa.eu/system/files/2016-11/annex11_01-2011_en_0.pdf
- EU GMP annex governing computerised systems used in pharmaceutical manufacturing and logistics. Any platform serving European pharmaceutical customers must demonstrate Annex 11 compliance, covering validation, data integrity, electronic signatures, and audit trails.

**EU Good Distribution Practice (GDP) Guidelines 2013/C 68/01**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A52013XC0307%2801%29
- European Commission GDP guidelines for the proper distribution of medicinal products. Sets temperature monitoring requirements for storage (2–8°C for most medicines, −20°C for frozen biologics) and transportation, including documentation, deviation management, and qualified persons.

**FDA Food Safety Modernization Act (FSMA) — Preventive Controls Rule**
- URL: https://www.fda.gov/food/food-safety-modernization-act-fsma
- US law requiring preventive controls and traceability for food supply chains. Drives cold chain data retention requirements and temperature monitoring at critical control points. Relevant to food industry customers.

**WHO Technical Report Series (TRS) 961 — Annex 9: Model Guidance for the Storage and Transport of Time- and Temperature-Sensitive Pharmaceutical Products**
- URL: https://www.who.int/publications/m/item/trs961-annex9
- WHO guidelines covering temperature requirements for vaccines and biologics throughout the cold chain. Essential reference for vaccine cold chain modules, specifying temperature ranges, alarm limits, and monitoring frequency.

**US Pharmacopeia (USP) <1079> — Good Storage and Distribution Practices for Drug Products**
- URL: https://www.usp.org/packaging-storage-distribution/distribution-practices
- USP chapter providing practical guidance on storage and distribution conditions for drug products. Addresses temperature mapping, monitoring equipment qualification, and documentation requirements directly applicable to cold chain monitoring software.

**HACCP (Hazard Analysis and Critical Control Points)**
- URL: https://www.fda.gov/food/hazard-analysis-critical-control-point-haccp
- Food safety management framework requiring temperature monitoring at critical control points. Drives the need for automated CCP logging, configurable alert thresholds, and corrective action workflows in food cold chain monitoring platforms.

---

### Data Model & Traceability Standards

**GS1 EPCIS 2.0 / ISO IEC 19987:2024 — Electronic Product Code Information Services**
- URL: https://www.gs1.org/standards/epcis
- OpenAPI spec: https://ref.gs1.org/standards/epcis/artefacts
- The primary global standard for supply chain event data exchange. EPCIS 2.0 (ratified June 2022, adopted as ISO/IEC 19987:2024) adds native sensor data support (`sensorElementList`) for temperature, humidity, and other environmental readings. Uses JSON-LD encoding and a REST API for event capture and query. A cold chain platform should support EPCIS 2.0 event export as the standard interoperability format for supply chain partners.

**GS1 CBV (Core Business Vocabulary) 2.0**
- URL: https://www.gs1.org/standards/epcis
- Companion standard to EPCIS defining standard business vocabulary and event types. Includes cold chain-relevant event types such as temperature excursion events and sensor observation events.

**OpenEPCIS — Open-Source GS1-Compliant EPCIS Implementation**
- URL: https://openepcis.io/docs/
- Apache 2.0 open-source EPCIS 2.0 server implementation. Provides a reference implementation of the EPCIS REST API that a cold chain monitoring platform could embed or integrate for supply chain data sharing.

---

### IoT Communication Protocols & Standards

**MQTT v5.0 (OASIS Standard / ISO/IEC 20922:2016)**
- URL: https://mqtt.org/ · ISO: https://www.iso.org/standard/69466.html
- The de-facto IoT messaging protocol, standardised by OASIS. Lightweight publish-subscribe protocol over TCP/IP suited for constrained sensor devices. Studies show MQTT at QoS 0–1 delivers 6–8% energy savings over HTTP for cold chain telemetry. Cold chain sensor devices should publish temperature events over MQTT to a platform broker.

**AMQP 1.0 (OASIS Standard / ISO/IEC 19464:2014)**
- URL: https://www.amqp.org/
- Enterprise-grade message queuing protocol for reliable, guaranteed-delivery messaging. Suitable for cold chain platform back-end event buses where message loss is unacceptable. Used in Azure Service Bus and RabbitMQ.

**CoAP — Constrained Application Protocol (IETF RFC 7252)**
- URL: https://www.rfc-editor.org/rfc/rfc7252
- Lightweight REST-like protocol for IP-based constrained IoT devices (IPv6/6LoWPAN sensor nodes). Relevant for cold chain sensors running on very low-power microcontrollers without TCP/IP stack.

**LwM2M — Lightweight M2M (OMA SpecWorks)**
- URL: https://lwm2m.openmobilealliance.org/
- Device management and telemetry protocol for constrained IoT devices. Standardises device registration, firmware update, and sensor data reporting; relevant for managing large fleets of cold chain sensors.

**LoRaWAN Specification (LoRa Alliance)**
- URL: https://lora-alliance.org/lorawan-for-developers/
- Low-power wide-area network (LPWAN) specification for long-range sensor connectivity. Widely used in warehouse and large-site cold chain monitoring where cellular is cost-prohibitive. Defines the MAC layer and regional frequency plans for LoRa sensor deployments.

---

### Security & Authentication Standards

**OAuth 2.0 (IETF RFC 6749) and OpenID Connect 1.0**
- URL: https://oauth.net/2/ · https://openid.net/connect/
- Standard protocols for delegated authorisation and authentication. A cold chain monitoring REST API must support OAuth 2.0 bearer tokens; multi-tenant SaaS should support OpenID Connect for SSO integration with enterprise identity providers (Okta, Azure AD, etc.).

**SAML 2.0 (OASIS Standard)**
- URL: https://wiki.oasis-open.org/security/FrontPage
- XML-based single sign-on standard widely required by pharmaceutical and food enterprise customers for SSO integration. Emerson Oversight 2 added SSO; new platforms should support it from the start.

**OWASP IoT Security Verification Standard (ISVS)**
- URL: https://owasp.org/www-project-iot-security-verification-standard/
- OWASP standard for verifying IoT device and platform security. Covers secure boot, firmware update, communications encryption, and API security relevant to cold chain sensor hardware and cloud platforms.

**TLS 1.3 (IETF RFC 8446)**
- URL: https://www.rfc-editor.org/rfc/rfc8446
- All sensor-to-cloud and API communications should use TLS 1.3. Pharmaceutical and food enterprise customers require encrypted data in transit; TLS 1.3 is the minimum requirement for 21 CFR Part 11 and EU Annex 11 compliance.

**NIST SP 800-53 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- US NIST framework for information security controls. Relevant for pharmaceutical customers operating under US federal guidelines or seeking FedRAMP compliance for government-adjacent cold chain use cases.

---

### Data Format & API Specification Standards

**OpenAPI 3.1 Specification**
- URL: https://spec.openapis.org/oas/v3.1.0
- Standard for documenting RESTful APIs. A cold chain monitoring platform API must be documented in OpenAPI 3.1 to enable client SDK generation, API gateway integration, and third-party developer adoption.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/draft/2020-12/json-schema-core.html
- Standard for defining and validating JSON data structures. Used by EPCIS 2.0 to formally define event schemas; relevant for validating sensor telemetry payloads and API request/response models.

**RFC 8259 — The JavaScript Object Notation (JSON) Data Interchange Format**
- URL: https://www.rfc-editor.org/rfc/rfc8259
- Baseline standard for all REST API payloads. Cold chain platform APIs should use JSON as the primary serialisation format.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- Open protocol for connecting AI language models to external tools and data sources. A cold chain monitoring MCP server could expose shipment status, excursion history, lane analytics, and compliance report generation as LLM-callable tools, enabling AI agents to query and act on cold chain data in natural language workflows.

---

## Similar Products — Developer Documentation & APIs

### Tive REST API
- **Description:** Tive provides a public REST API and webhook system for programmatic access to shipment data, tracker status, alert events, and location history. Targeted at logistics software developers integrating cold chain monitoring into TMS/ERP systems.
- **API Documentation:** https://developers.tive.com/docs/rest-api-overview
- **API Reference (v3):** https://api.tive.com/public/v3/docs/index.html
- **Webhooks:** https://developers.tive.com/docs/webhooks
- **Webhook Data Structures:** https://developers.tive.com/docs/data-structures
- **SDKs/Libraries:** Not publicly documented; REST/JSON only
- **Standards:** REST/JSON, OpenAPI-style documentation
- **Authentication:** Bearer token (API key); OAuth 2.0 implied for enterprise

---

### GS1 EPCIS 2.0 REST API (Reference Implementation)
- **Description:** The GS1 EPCIS 2.0 standard includes a normative REST API for capturing and querying supply chain events, including sensor-based temperature data. The official artefacts include an OpenAPI 3.0 specification.
- **API Documentation:** https://ref.gs1.org/standards/epcis/artefacts
- **OpenAPI Spec:** Available at https://ref.gs1.org/standards/epcis/artefacts (EPCIS 2.0.1 OpenAPI)
- **Standard:** GS1 EPCIS 2.0 / ISO IEC 19987:2024
- **Authentication:** Not mandated by standard; implementation-specific (typically OAuth 2.0)

---

### OpenEPCIS (Open-Source EPCIS Server)
- **Description:** A GS1-compliant open-source EPCIS 2.0 server. Provides a production-ready implementation of the EPCIS capture and query REST API in Java. Can be embedded in a cold chain monitoring platform to handle supply chain event data sharing.
- **API Documentation:** https://openepcis.io/docs/epcis/
- **Developer Guide:** https://openepcis.io/docs/
- **SDKs/Libraries:** Java (primary); REST API is language-agnostic
- **Standards:** EPCIS 2.0 / ISO IEC 19987:2024, JSON-LD
- **Authentication:** Configurable; supports OAuth 2.0

---

### Particle IoT Platform (Developer API)
- **Description:** Particle provides a REST API and webhook framework for managing IoT device fleets, dispatching commands, and receiving telemetry data. Open firmware reference implementation available for cold chain asset tracking.
- **API Documentation:** https://docs.particle.io/reference/cloud-apis/api/
- **SDK/Libraries:** JavaScript, Python, iOS, Android (open-source)
- **Developer Guide:** https://docs.particle.io/getting-started/
- **Standards:** REST/JSON, MQTT (device-to-cloud), Webhook
- **Authentication:** OAuth 2.0

---

### Geotab MyGeotab SDK
- **Description:** Geotab's fleet telematics platform exposes a REST API and JavaScript SDK enabling third parties to build cold chain add-ins and integrate temperature monitoring data with other fleet management data.
- **API Documentation:** https://geotab.github.io/sdk/
- **SDKs/Libraries:** .NET, JavaScript, Python (open-source)
- **Developer Guide:** https://geotab.github.io/sdk/software/guides/getting-started/
- **Standards:** REST/JSON, MyGeotab data model
- **Authentication:** Token-based (username/password token exchange)

---

### ThingsBoard (Open-Source IoT Platform)
- **Description:** Open-source IoT data platform supporting device connectivity (MQTT, CoAP, HTTP), real-time dashboards, rule engine, and REST API. Often used as the software layer for custom cold chain monitoring deployments with Milesight or Particle hardware.
- **API Documentation:** https://thingsboard.io/docs/reference/rest-api/
- **SDKs/Libraries:** Java, Python, JavaScript; open-source
- **Developer Guide:** https://thingsboard.io/docs/getting-started-guides/
- **Standards:** MQTT, CoAP, HTTP/REST, JSON
- **Authentication:** OAuth 2.0, JWT tokens

---

### AWS IoT Core
- **Description:** AWS managed IoT broker and device management service. Supports MQTT, HTTP, and WebSocket for sensor data ingestion at scale. Cold chain platforms built on AWS commonly use IoT Core for telemetry ingestion feeding Lambda/Kinesis processing pipelines.
- **API Documentation:** https://docs.aws.amazon.com/iot/latest/apireference/
- **SDKs/Libraries:** C, Python, JavaScript, Java, Go, .NET (open-source)
- **Developer Guide:** https://docs.aws.amazon.com/iot/latest/developerguide/
- **Standards:** MQTT 3.1/5.0, HTTP/REST, WebSocket, OpenAPI
- **Authentication:** AWS SigV4, X.509 mutual TLS (device), IAM roles

---

### Azure IoT Hub
- **Description:** Microsoft's managed IoT connectivity and device management service. MQTT/AMQP/HTTPS support; integrates with Azure Stream Analytics, Event Hubs, and Time Series Insights for cold chain telemetry processing and historical analysis.
- **API Documentation:** https://docs.microsoft.com/en-us/rest/api/iothub/
- **SDKs/Libraries:** C, Python, JavaScript, Java, .NET (open-source)
- **Developer Guide:** https://docs.microsoft.com/en-us/azure/iot-hub/
- **Standards:** MQTT 3.1, AMQP 1.0, HTTPS, OpenAPI
- **Authentication:** SAS tokens, X.509 certificates, Azure AD

---

## Notes

- **EPCIS 2.0 adoption is still nascent**: despite being ratified in 2022 and adopted as ISO/IEC 19987:2024, no major commercial cold chain monitoring vendor (Sensitech, Tive, Controlant) currently advertises native EPCIS 2.0 event export. This is a meaningful interoperability gap and an opportunity for an open-source platform to lead.
- **No sector-specific cold chain API standard exists**: unlike financial services (OpenBanking) or healthcare (FHIR), there is no mandated API schema for cold chain monitoring. EPCIS 2.0 is the closest candidate but it is not a REST API standard in the traditional sense.
- **MCP server opportunity**: no cold chain platform currently exposes an MCP server. An open-source platform providing an MCP server would allow LLM-powered quality management agents to query shipment status, generate excursion reports, and recommend corrective actions natively.
- **LoRaWAN regional variance**: LoRaWAN frequency plans vary by region (EU868, US915, AS923, AU915). Any platform supporting LoRaWAN sensors must handle multi-regional network server configurations.
- **21 CFR Part 11 software validation**: compliance with this regulation requires IQ/OQ/PQ validation documentation, which is a significant ongoing effort. SaaS vendors typically provide pre-written validation packages (IQ/OQ protocols) as a paid service add-on.
