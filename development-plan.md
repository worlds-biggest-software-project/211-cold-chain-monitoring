# Cold Chain Monitoring — Phased Development Plan

> Project: 211-cold-chain-monitoring · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ (backend) + TypeScript (frontend) | Python for MQTT sensor ingestion (paho-mqtt), ML-based excursion prediction (scikit-learn, XGBoost), LLM compliance report generation; TypeScript for real-time monitoring dashboard |
| API framework | FastAPI | Async for high-throughput MQTT message processing and concurrent sensor streams; auto-generates OpenAPI 3.1; Pydantic v2 for strict sensor data validation |
| Database | PostgreSQL 16 | Normalised schema from data-model-suggestion-1 (~45 tables); 21 CFR Part 11 audit trails require referential integrity; JSONB for sensor-specific metadata |
| Time-series | TimescaleDB (PostgreSQL extension) | High-volume sensor readings (temperature, humidity, location) — hypertables with automatic compression and retention policies |
| Migrations | Alembic | Version-controlled schema; critical for pharma-grade compliance where schema changes must be documented |
| ORM | SQLAlchemy 2.0 (async) | Complex joins across shipment→sensor→reading→excursion→investigation chains |
| Message broker | MQTT broker (EMQX) | Standard IoT protocol for BLE/cellular/LoRa sensor ingestion; EMQX handles millions of messages per second |
| Task queue | Celery + Redis | Async excursion detection, alert delivery, ML prediction, compliance report generation |
| ML / Prediction | scikit-learn + XGBoost | Excursion prediction from sensor, weather, and route data; alert noise classification |
| LLM integration | Anthropic Python SDK (Claude) | Auto-draft GDP/FDA deviation investigation reports from excursion data |
| Frontend | Next.js 15 (App Router) + Tailwind CSS | Real-time monitoring dashboard with live sensor charts, shipment map, excursion timeline |
| Real-time charts | Recharts + SSE | Live temperature/humidity charts updated via Server-Sent Events |
| Maps | Mapbox GL JS | Shipment tracking map with temperature overlay on route |
| PDF generation | WeasyPrint | 21 CFR Part 11-compliant excursion reports, chain-of-custody documents |
| Alerting | Resend (email) + Twilio (SMS) + webhooks | Multi-channel excursion alerts |
| Auth | Better Auth | Multi-org with pharma roles; electronic signature support for 21 CFR Part 11 |
| Containerisation | Docker + docker-compose | PostgreSQL+TimescaleDB, Redis, EMQX, FastAPI, Celery, Next.js |
| Testing | pytest + pytest-asyncio + httpx + Vitest | Backend (pytest); Frontend (Vitest); E2E (Playwright) |
| Linting | Ruff (Python) + Biome (TS) | Fast linting/formatting |
| Type checking | mypy (strict) + TypeScript strict | Both stacks type-checked |
| Package manager | uv (Python) + pnpm (TS) | Fast, lockfile-based |

### Project Structure

```
cold-chain/
├── pyproject.toml
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.frontend
├── alembic.ini
├── alembic/versions/
├── src/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models/                     # SQLAlchemy ORM
│   │   ├── organisation.py
│   │   ├── sensor.py
│   │   ├── shipment.py
│   │   ├── sensor_reading.py       # TimescaleDB hypertable
│   │   ├── excursion.py
│   │   ├── investigation.py
│   │   ├── alert.py
│   │   ├── audit_trail.py          # 21 CFR Part 11
│   │   ├── electronic_signature.py
│   │   └── epcis_event.py
│   ├── schemas/
│   │   ├── sensors.py
│   │   ├── shipments.py
│   │   ├── readings.py
│   │   ├── excursions.py
│   │   └── compliance.py
│   ├── api/
│   │   ├── sensors.py
│   │   ├── shipments.py
│   │   ├── readings.py
│   │   ├── excursions.py
│   │   ├── compliance.py
│   │   ├── reports.py
│   │   └── ai.py
│   ├── services/
│   │   ├── mqtt_ingestion.py       # MQTT subscriber
│   │   ├── excursion_detector.py   # Real-time threshold checking
│   │   ├── alert_service.py        # Email, SMS, webhook
│   │   ├── shipment_tracker.py
│   │   ├── compliance_engine.py    # 21 CFR Part 11 audit
│   │   ├── epcis_exporter.py       # EPCIS 2.0 event generation
│   │   ├── report_generator.py     # PDF compliance reports
│   │   ├── excursion_predictor.py  # ML prediction (v1.1)
│   │   ├── noise_classifier.py    # Alert noise reduction (v1.1)
│   │   └── shelf_life.py          # Dynamic shelf-life (v1.1)
│   ├── ml/
│   │   ├── prediction_model.py
│   │   ├── noise_model.py
│   │   └── features.py
│   ├── tasks/
│   │   ├── alerts.py
│   │   ├── reports.py
│   │   ├── prediction.py
│   │   └── epcis.py
│   └── lib/
│       ├── auth.py
│       ├── audit.py               # 21 CFR Part 11 audit trail
│       ├── esig.py                # Electronic signatures
│       ├── mqtt.py
│       └── pdf.py
├── frontend/
│   ├── package.json
│   └── src/app/
│       ├── dashboard/             # Real-time monitoring
│       ├── shipments/
│       ├── sensors/
│       ├── excursions/
│       ├── compliance/
│       ├── reports/
│       └── settings/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│       ├── mqtt_reading.json
│       ├── sample_excursion.json
│       └── epcis_event.json
└── templates/
    ├── excursion_report.html      # WeasyPrint template
    └── chain_of_custody.html
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema from data-model-suggestion-1 (organisations, sensors, shipments, sensor_readings, excursions), 21 CFR Part 11 audit trail module, authentication, and Docker environment.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create pyproject.toml, FastAPI app, Pydantic Settings, Docker with PostgreSQL+TimescaleDB, Redis, EMQX.

**Design**:

```python
class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://coldchain:coldchain@localhost:5432/coldchain"
    redis_url: str = "redis://localhost:6379/0"
    mqtt_broker_url: str = "mqtt://localhost:1883"
    mqtt_topic_prefix: str = "coldchain/sensors"
    anthropic_api_key: str = ""
    resend_api_key: str = ""
    twilio_account_sid: str = ""
    twilio_auth_token: str = ""
    site_url: str = "http://localhost:3000"
    model_config = {"env_prefix": "COLD_"}
```

**Testing**:
- Unit: config parses env correctly
- Integration: `docker-compose up -d` → PostgreSQL+TimescaleDB, Redis, EMQX healthy

#### 1.2 — Database Schema — Core Tables and TimescaleDB

**What**: Implement organisations, sensors, shipments, sensor_readings (TimescaleDB hypertable), temperature_ranges, excursions, alerts from data-model-suggestion-1.

**Design**:

```python
# TimescaleDB hypertable for sensor readings
class SensorReading(Base):
    __tablename__ = "sensor_readings"
    time: Mapped[datetime]          # TimescaleDB time column
    sensor_id: Mapped[UUID]
    shipment_id: Mapped[UUID | None]
    reading_type: Mapped[str]       # temperature, humidity, shock, light, location
    value: Mapped[Decimal]          # temperature in °C, humidity in %, shock in G
    unit: Mapped[str]               # celsius, fahrenheit, percent, g_force
    latitude: Mapped[Decimal | None]
    longitude: Mapped[Decimal | None]
    battery_pct: Mapped[int | None]
    signal_strength: Mapped[int | None]
    raw_payload: Mapped[dict | None]  # original sensor message for audit

# SELECT create_hypertable('sensor_readings', 'time');
# SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');
# SELECT add_retention_policy('sensor_readings', INTERVAL '5 years');  -- FDA retention

# Excursion table
class Excursion(Base):
    __tablename__ = "excursions"
    id: Mapped[UUID]
    shipment_id: Mapped[UUID]
    sensor_id: Mapped[UUID]
    excursion_type: Mapped[str]     # temperature_high, temperature_low, humidity_high, shock
    threshold_value: Mapped[Decimal]
    peak_value: Mapped[Decimal]     # worst reading during excursion
    started_at: Mapped[datetime]
    ended_at: Mapped[datetime | None]
    duration_minutes: Mapped[int | None]
    status: Mapped[str]             # active, resolved, under_investigation, closed
    severity: Mapped[str]           # minor, major, critical
    cumulative_exposure_degree_minutes: Mapped[Decimal | None]
```

Temperature ranges reference table: `frozen` (-25°C to -15°C), `refrigerated` (2°C to 8°C), `controlled_room_temp` (15°C to 25°C) — per USP <1079> and EU GDP guidelines.

**Testing**:
- Integration: migrations apply, TimescaleDB hypertable created
- Integration: insert 100,000 sensor readings → TimescaleDB partitions correctly
- Unit: excursion severity classification: deviation > 5°C = critical, > 2°C = major, > 0.5°C = minor

#### 1.3 — 21 CFR Part 11 Audit Trail and Electronic Signatures

**What**: Immutable audit trail capturing all record changes; electronic signature module for compliance sign-offs.

**Design**:

```python
# 21 CFR Part 11 requires:
# 1. Tamper-evident audit trail of all record changes
# 2. User identification tied to every action
# 3. Electronic signatures with meaning, timestamp, and signer identity
# 4. Computer-generated, time-stamped audit trails

class AuditTrailEntry(Base):
    __tablename__ = "audit_trail"
    id: Mapped[UUID]
    table_name: Mapped[str]
    record_id: Mapped[UUID]
    action: Mapped[str]              # create, update, delete, sign
    user_id: Mapped[UUID]
    user_email: Mapped[str]          # snapshot at time of action
    old_values: Mapped[dict | None]  # JSONB
    new_values: Mapped[dict | None]  # JSONB
    reason: Mapped[str | None]       # required for modifications per 21 CFR Part 11
    ip_address: Mapped[str]
    created_at: Mapped[datetime]     # tamper-evident: auto-generated, no manual override

class ElectronicSignature(Base):
    __tablename__ = "electronic_signatures"
    id: Mapped[UUID]
    record_type: Mapped[str]         # excursion_investigation, shipment_release, report
    record_id: Mapped[UUID]
    signer_id: Mapped[UUID]
    signer_name: Mapped[str]
    meaning: Mapped[str]             # "I have reviewed and approve this record"
    signature_hash: Mapped[str]      # HMAC of record content + signer identity
    signed_at: Mapped[datetime]
```

Audit trail is insert-only; no UPDATE or DELETE operations on audit_trail table (enforced via database trigger or application middleware).

**Testing**:
- Unit: update excursion status → audit_trail entry with old/new values
- Unit: audit_trail entry has correct user, timestamp, and IP address
- Unit: attempt to modify audit_trail row → error (insert-only table)
- Unit: electronic signature with meaning → signature_hash verifiable
- Integration: sign investigation report → electronic_signature record created, audit_trail logged

#### 1.4 — Authentication and Roles

**What**: Multi-org auth with cold chain-specific roles and 21 CFR Part 11 user identification.

**Design**:

Roles: `admin` (full access), `quality_manager` (excursions, investigations, reports, e-signatures), `logistics_operator` (shipments, sensor management), `read_only` (dashboards, reports), `api_service` (sensor ingestion).

21 CFR Part 11 user controls: unique user ID, password complexity, session timeout, failed login lockout.

**Testing**:
- Unit: quality_manager can sign investigation → allowed
- Unit: logistics_operator cannot sign investigation → 403
- Unit: 5 failed logins → account locked

---

## Phase 2: Sensor Ingestion and Real-Time Monitoring

### Purpose
Connect to IoT sensors via MQTT and HTTP; ingest, validate, and store readings in TimescaleDB. This is the data foundation — everything depends on sensor data flowing in.

### Tasks

#### 2.1 — MQTT Sensor Ingestion

**What**: Subscribe to MQTT topics for sensor readings; validate, normalise, and store in TimescaleDB.

**Design**:

```python
# src/services/mqtt_ingestion.py
class MQTTIngestionService:
    TOPIC_PATTERN = "coldchain/sensors/{sensor_id}/readings"

    async def on_message(self, topic: str, payload: bytes) -> None:
        """
        1. Parse JSON payload: { "type": "temperature", "value": 5.2, "unit": "celsius",
           "lat": 40.7128, "lng": -74.0060, "battery": 85, "timestamp": "..." }
        2. Validate: value within physically plausible range (-80°C to 60°C for temp)
        3. Normalise: convert Fahrenheit to Celsius if needed
        4. Store in sensor_readings TimescaleDB hypertable
        5. Check against thresholds → trigger excursion detection
        """
```

Also support HTTP POST for non-MQTT sensors:
```python
# POST /api/v1/sensors/{sensorId}/readings (batch)
class ReadingBatchRequest(BaseModel):
    readings: list[SensorReadingInput]  # up to 1000 per batch
```

**Testing**:
- Unit: valid MQTT message → sensor_reading stored with correct values
- Unit: temperature 500°C → rejected as implausible
- Unit: Fahrenheit reading → converted to Celsius before storage
- Integration: publish 100 MQTT messages → 100 readings in TimescaleDB within 2 seconds
- Unit: HTTP batch of 500 readings → all stored correctly

#### 2.2 — Excursion Detection and Alerting

**What**: Detect temperature excursions in real time; generate alerts via email, SMS, and webhook.

**Design**:

```python
class ExcursionDetector:
    async def check_reading(self, reading: SensorReading) -> Excursion | None:
        """
        1. Get active shipment for this sensor
        2. Get temperature range for shipment's product type
        3. If reading.value outside range:
           a. Check if active excursion exists for this sensor → extend it
           b. If no active excursion → create new excursion, trigger alert
        4. If reading.value back in range and active excursion exists:
           → Close excursion, compute duration and cumulative exposure
        5. Compute severity from deviation magnitude and duration
        """

    async def send_alerts(self, excursion: Excursion) -> None:
        """
        Send to all subscribed users for this shipment:
        1. Email via Resend
        2. SMS via Twilio (for critical severity only)
        3. Webhook POST to registered URLs
        """
```

Cumulative thermal exposure: `degree_minutes = Σ(|reading - threshold| × interval_minutes)` — used for shelf-life impact assessment.

**Testing**:
- Unit: reading 10°C with 2-8°C range → excursion created, severity = minor
- Unit: reading 15°C with 2-8°C range → excursion with severity = major
- Unit: reading returns to 5°C with active excursion → excursion closed, duration computed
- Unit: cumulative exposure correctly computed from 5 readings over 30 minutes
- Integration (mocked): excursion → email + SMS sent to subscribed users
- Integration: webhook delivered with correct excursion payload

---

## Phase 3: Shipment Lifecycle and Dashboard

### Purpose
Track shipments from creation through delivery with chain-of-custody logging. Build the operations dashboard with live sensor charts and shipment status.

### Tasks

#### 3.1 — Shipment Lifecycle Management

**What**: Create, track, and close shipments with sensor assignment and chain-of-custody events.

**Design**:

Shipment state: `created` → `sensor_assigned` → `in_transit` → `at_checkpoint` → `delivered` → `closed`.

```python
# POST /api/v1/shipments
class ShipmentCreate(BaseModel):
    productName: str
    productType: str                  # vaccine, biologic, food_frozen, food_refrigerated, reagent
    temperatureRangeId: UUID          # FK to temperature_ranges
    originLocationName: str
    destinationLocationName: str
    carrierName: str | None
    estimatedDepartureAt: datetime
    estimatedArrivalAt: datetime
    sensorIds: list[UUID]             # sensors to attach

# Chain of custody: POST /api/v1/shipments/{id}/custody-events
class CustodyEvent(BaseModel):
    eventType: Literal["pickup", "handoff", "checkpoint", "delivery"]
    locationName: str
    latitude: Decimal | None
    longitude: Decimal | None
    custodianName: str
    notes: str | None
    photoUrl: str | None              # photo evidence
```

**Testing**:
- Unit: create shipment with 2 sensors → sensors linked to shipment
- Unit: custody events logged with timestamps → chain visible in timeline
- Unit: shipment delivered → status = `delivered`
- Integration: full lifecycle → audit trail records all transitions

#### 3.2 — Real-Time Monitoring Dashboard

**What**: Live dashboard showing sensor readings, shipment map, and excursion timeline.

**Design**:

`/dashboard` page: 
- **Active Shipments** card: count, in-transit vs. at-checkpoint
- **Active Excursions** card: count by severity (red badge for critical)
- **Live Sensor Chart**: Recharts line chart with temperature over time, threshold bands shaded; auto-refreshes via SSE every 30 seconds
- **Shipment Map**: Mapbox map with route lines and sensor position pins, coloured by status (green = in range, red = excursion)
- **Excursion Timeline**: recent excursions with severity badge, duration, shipment link

SSE endpoint: `GET /api/v1/sensors/{sensorId}/readings/stream` → pushes new readings as SSE events.

**Testing**:
- E2E: 5 active shipments → dashboard shows correct counts
- E2E: new sensor reading → chart updates within 30 seconds
- E2E: active excursion → red badge visible on dashboard and map
- Unit: SSE endpoint delivers readings in correct format

---

## Phase 4: Compliance Reporting

### Purpose
Generate 21 CFR Part 11-compliant excursion investigation reports and chain-of-custody documents. The compliance output that regulators and quality teams require.

### Tasks

#### 4.1 — Excursion Investigation Workflow

**What**: Quality manager investigates excursion, documents root cause, and signs off with electronic signature.

**Design**:

Investigation state: `opened` → `under_investigation` → `corrective_action` → `signed_off` → `closed`.

```python
# POST /api/v1/excursions/{id}/investigation
class InvestigationCreate(BaseModel):
    rootCause: str
    impactAssessment: str              # product impact determination
    correctiveAction: str
    preventiveAction: str
    disposition: Literal["release", "quarantine", "reject", "return_to_sender"]
    productAffected: bool
    investigatorUserId: UUID

# POST /api/v1/excursions/{id}/investigation/sign
class SignRequest(BaseModel):
    meaning: str                       # "I have reviewed and approve this investigation"
    password: str                      # re-authentication required for e-sig per 21 CFR Part 11
```

**Testing**:
- Unit: create investigation → status = `under_investigation`
- Unit: sign investigation → electronic signature created, status = `signed_off`
- Unit: sign without valid password → 401 (re-authentication required)
- Unit: signed investigation → immutable (no further edits allowed)
- Integration: full workflow → audit trail records every step

#### 4.2 — PDF Report Generation

**What**: Generate FDA/GDP-compliant PDF excursion reports and chain-of-custody documents.

**Design**:

WeasyPrint renders HTML template → PDF. Report includes: shipment details, temperature chart (entire journey), excursion details (start, end, peak, duration, cumulative exposure), investigation findings, corrective action, electronic signature block, audit trail excerpt.

```python
class ReportGenerator:
    async def generate_excursion_report(self, excursion_id: UUID) -> bytes:
        """
        1. Load excursion with investigation, readings, shipment, custody events
        2. Generate temperature chart as inline SVG
        3. Render HTML template with all data
        4. Convert to PDF via WeasyPrint
        5. Store PDF, create audit trail entry
        """
```

**Testing**:
- Unit: excursion with investigation → valid PDF with all required sections
- Unit: PDF contains temperature chart, excursion highlighted in red
- Unit: electronic signature block present with signer name and timestamp
- Integration: generate report → PDF readable, stored in S3

---

## Phase 5: EPCIS 2.0 Export and ERP Integration

### Purpose
Export shipment and sensor events in GS1 EPCIS 2.0 format for cross-platform interoperability. Provides the data portability that no incumbent offers natively.

### Tasks

#### 5.1 — EPCIS 2.0 Event Generation

**What**: Generate EPCIS 2.0-compliant events from shipment and sensor data for cross-vendor portability.

**Design**:

```python
class EPCISExporter:
    async def export_shipment_events(self, shipment_id: UUID) -> list[dict]:
        """Generate EPCIS 2.0 events per ISO/IEC 19987:2024:
        1. ObjectEvent (type=shipping) at origin pickup
        2. SensorElement events for temperature readings (sensorReport with type, value, uom)
        3. AggregationEvent at checkpoint handoffs
        4. ObjectEvent (type=receiving) at destination delivery
        Return as JSON-LD per EPCIS 2.0 REST binding.
        """

# EPCIS sensorReport structure (per GS1 EPCIS 2.0):
# {
#   "type": "SensorElement",
#   "sensorReport": [{
#     "type": "gs1:Temperature",
#     "value": 5.2,
#     "uom": "CEL",
#     "time": "2026-05-29T14:30:00Z",
#     "deviceID": "urn:epc:id:giai:1234567890.sensor001"
#   }]
# }

# GET /api/v1/shipments/{id}/epcis → EPCIS 2.0 JSON-LD event list
```

**Testing**:
- Unit: shipment with 5 custody events → 5 EPCIS ObjectEvent/AggregationEvent records
- Unit: sensor readings → EPCIS SensorElement events with correct GS1 URIs
- Unit: EPCIS output validates against EPCIS 2.0 JSON Schema
- Fixture: `tests/fixtures/epcis_event.json` → golden reference

---

## Phase 6: Lane and Carrier Analytics

### Purpose
Track carrier and lane performance over time to identify high-risk routes and carriers. Informs quality audit prioritisation.

### Tasks

#### 6.1 — Performance Analytics Dashboard

**What**: Lane and carrier scorecards showing excursion rates, on-time delivery, and temperature compliance.

**Design**:

Metrics per carrier/lane: excursion rate (excursions / total shipments), avg excursion duration, avg cumulative exposure, on-time delivery %, temperature compliance % (readings within range / total readings).

`/analytics` page: carrier comparison table, lane heatmap (origin × destination coloured by excursion rate), trend charts over 12 months.

**Testing**:
- Unit: 100 shipments, 5 with excursions → excursion rate = 5%
- E2E: analytics page → carrier scorecard renders with correct metrics
- Unit: lane with 20% excursion rate → highlighted red in heatmap

---

## Phase 7: AI-Driven Features (v1.1)

### Purpose
Add AI differentiators: predictive excursion forecasting, alert noise reduction, LLM-assisted compliance reports, and dynamic shelf-life estimation.

### Tasks

#### 7.1 — Predictive Excursion Forecasting

**What**: ML model predicting temperature excursions hours before they occur from sensor trends, weather, and route data.

**Design**:

```python
class ExcursionPredictor:
    def predict(self, sensor_id: UUID, shipment_id: UUID) -> ExcursionPrediction:
        """
        Features: current_temp, temp_trend_1h, temp_trend_4h, ambient_temp_forecast,
                  carrier_historical_excursion_rate, product_sensitivity,
                  time_in_transit_hours, remaining_distance_km
        Model: XGBoost classifier (excursion within next 4 hours: yes/no)
        Output: probability, risk_tier, contributing_factors, recommended_action
        """

@dataclass
class ExcursionPrediction:
    probability: float          # 0-1
    risk_tier: str              # low, medium, high, critical
    predicted_breach_in_hours: float
    contributing_factors: list[str]  # ["rising_temp_trend", "ambient_forecast_32C"]
    recommended_action: str         # "Pre-cool at next checkpoint" or "Reroute to closer DC"
```

Runs every 30 minutes for all active shipments via Celery task. Alert sent if probability > 0.7.

**Testing**:
- Unit: rising temp trend + high ambient forecast → probability > 0.7
- Unit: stable temp, cool ambient → probability < 0.2
- Integration: prediction run → alert sent for high-risk shipments

#### 7.2 — Alert Noise Classification

**What**: ML classifier distinguishing genuine excursions from defrost cycles, door-open events, and sensor noise.

**Design**:

```python
class NoiseClassifier:
    def classify(self, excursion: Excursion, readings: list[SensorReading]) -> NoiseResult:
        """
        Features: excursion_duration_minutes, peak_deviation, recovery_speed,
                  time_of_day (defrost cycles are scheduled), concurrent_sensor_agreement,
                  preceding_reading_pattern
        Classes: genuine_excursion, defrost_cycle, door_open, sensor_noise
        Model: Random Forest trained on labelled historical excursions
        """
```

If classified as defrost_cycle/door_open/sensor_noise → alert downgraded, excursion tagged accordingly. Reduces alert fatigue by an estimated 30-50%.

**Testing**:
- Unit: 5-minute spike at 3am (defrost) → classified as defrost_cycle
- Unit: 30-second spike during business hours → classified as door_open
- Unit: sustained 2-hour deviation → classified as genuine_excursion

#### 7.3 — LLM-Assisted Compliance Report Drafting

**What**: Claude auto-drafts GDP/FDA deviation investigation reports from excursion and shipment data.

**Design**:

```python
class ComplianceReportDrafter:
    SYSTEM_PROMPT = """You are a pharmaceutical quality assurance specialist drafting
    GDP and FDA 21 CFR Part 11 deviation investigation reports. Given excursion data,
    sensor readings, shipment details, and product information, draft a formal investigation
    report including: description of deviation, impact assessment, root cause analysis,
    corrective and preventive actions (CAPA), and disposition recommendation.
    Use formal pharmaceutical regulatory language."""

    async def draft_report(self, excursion_id: UUID) -> str:
        """
        Context provided to Claude: excursion details, full temperature chart data,
        product type and sensitivity, carrier history, custody events, weather conditions.
        Output: structured investigation narrative for quality manager review.
        """
```

Quality manager reviews and edits draft before signing off.

**Testing**:
- Unit (mocked): vaccine excursion with peak 12°C → report mentions cold chain break, recommends quarantine
- Unit (mocked): food excursion with 30-minute duration → report assesses minimal impact
- Integration: draft report → text returned, quality manager edits in UI

#### 7.4 — Dynamic Shelf-Life Estimation

**What**: Estimate remaining product shelf life from cumulative thermal exposure history.

**Design**:

```python
class ShelfLifeEstimator:
    async def estimate(self, shipment_id: UUID) -> ShelfLifeEstimate:
        """
        1. Load all temperature readings for shipment
        2. Compute cumulative thermal exposure: ∫|T - T_optimal| dt
        3. Apply product-specific Arrhenius model or simple MKT (Mean Kinetic Temperature)
        4. Compute remaining shelf life = original_shelf_life × degradation_factor
        5. Return: remaining_shelf_life_days, mkt_celsius, degradation_pct
        """

@dataclass
class ShelfLifeEstimate:
    original_shelf_life_days: int
    remaining_shelf_life_days: int
    mkt_celsius: Decimal              # Mean Kinetic Temperature
    degradation_pct: float            # % of shelf life consumed by thermal exposure
    is_within_spec: bool
```

**Testing**:
- Unit: perfect cold chain (constant 5°C for vaccine) → remaining ≈ original
- Unit: 4-hour excursion at 15°C → shelf life reduced, degradation_pct > 0
- Unit: MKT calculation correct for known temperature series (compare to USP reference)

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Sensor Ingestion & Monitoring ─── requires Phase 1
    │
Phase 3: Shipment Lifecycle & Dashboard ─── requires Phase 2
    │
Phase 4: Compliance Reporting          ─── requires Phase 3
    │
Phase 5: EPCIS 2.0 Export             ─── requires Phase 3 (parallel with Phase 4)
    │
Phase 6: Analytics                     ─── requires Phase 3 (parallel with Phases 4-5)
    │
Phase 7: AI Features                  ─── requires Phase 2 + Phase 3
```

Parallelism opportunities:
- Phases 4, 5, 6 can be developed concurrently after Phase 3
- Phase 7 tasks (7.1-7.4) can be developed independently after Phase 2+3

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`pytest` + `vitest`).
3. Ruff linting passes (Python); Biome passes (TypeScript).
4. mypy strict passes; tsc --noEmit passes.
5. Docker build succeeds.
6. `docker-compose up` brings all services to healthy state (including EMQX and TimescaleDB).
7. Feature works end-to-end.
8. 21 CFR Part 11 audit trail captures all record changes (verified by DB query).
9. Electronic signatures require re-authentication (verified by E2E test).
10. Sensor readings stored in TimescaleDB hypertable with correct partitioning.
11. New API endpoints appear in auto-generated OpenAPI spec.
12. Database migrations created and tested.
13. Excursion detection fires within 30 seconds of threshold breach.
14. New environment variables documented in config.py.
