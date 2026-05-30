# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Cold Chain Monitoring · Created: 2026-05-20

## Philosophy

This model treats the immutable event log as the single source of truth. Every state change — a sensor reading, a shipment status transition, a custody handover, an excursion acknowledgment — is recorded as an append-only event in a central event store. Current state is derived by replaying or projecting events into materialised read models optimised for specific query patterns.

This is the natural architecture for a domain where "what happened and when" is more important than "what is the current state." FDA 21 CFR Part 11 requires that record changes never obscure previously recorded information — event sourcing satisfies this by design, since events are never updated or deleted. The EPCIS 2.0 standard itself is fundamentally event-based (ObjectEvent, AggregationEvent, etc.), making event sourcing a philosophically aligned choice.

The CQRS (Command Query Responsibility Segregation) pattern separates the write path (commands that produce events) from the read path (materialised views optimised for dashboards, reports, and API queries). For IoT telemetry, the write side handles high-throughput sensor data ingestion via MQTT, while the read side maintains pre-computed aggregations for dashboard rendering and compliance reporting.

**Best for:** Deployments requiring complete, tamper-proof audit history and temporal queries ("What was the temperature at 3pm last Tuesday?", "Show me all state changes for this shipment"). Ideal for pharmaceutical and clinical trial cold chains where full traceability is non-negotiable.

**Trade-offs:**
- (+) Perfect audit trail by design — events are never modified or deleted
- (+) Temporal queries trivial — replay events to any point in time
- (+) Natural fit for EPCIS 2.0 event model and 21 CFR Part 11
- (+) Write path optimised for high-throughput sensor ingestion
- (+) Read models can be rebuilt or new ones added without changing the event store
- (-) Higher complexity — developers must understand event sourcing and CQRS patterns
- (-) Eventually consistent read models (slight delay between write and read)
- (-) Event store grows indefinitely; requires snapshotting strategy for long-lived entities
- (-) Schema evolution of event payloads requires careful versioning
- (-) Simple CRUD queries require joining materialised views rather than querying source tables directly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 / ISO IEC 19987:2024 | Event store event types directly map to EPCIS event types; sensorElementList data embedded as event payload. Export is a direct serialisation of stored events. |
| FDA 21 CFR Part 11 | Append-only event store inherently satisfies the requirement that changes never obscure previously recorded information. Electronic signature events recorded as first-class events. |
| EU GDP 2013/C 68/01 | Deviation and disposition events capture the full GDP decision workflow as an immutable sequence |
| HACCP | CCP check events form a time-ordered audit log of critical control point monitoring |
| ISO 31512:2024 | Shipment lifecycle events align with ISO 31512 cold chain logistics process stages |
| MQTT v5.0 / ISO IEC 20922 | Sensor telemetry events ingested via MQTT broker and appended directly to the event store |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth: all domain events
CREATE TABLE event_store (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,          -- aggregate root ID (shipment, sensor, org, etc.)
    stream_type VARCHAR(50) NOT NULL,  -- 'shipment', 'sensor', 'organisation', 'excursion', 'user'
    event_type VARCHAR(100) NOT NULL,  -- e.g. 'SensorReadingRecorded', 'ShipmentDeparted', 'ExcursionDetected'
    event_version INT NOT NULL,        -- monotonically increasing per stream
    payload JSONB NOT NULL,            -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',  -- correlation IDs, causation, user context
    org_id UUID NOT NULL,              -- tenant isolation
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_by UUID,                  -- user or system that caused the event
    
    UNIQUE(stream_id, event_version)   -- optimistic concurrency control
);

-- Primary query patterns
CREATE INDEX idx_event_store_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_store_type ON event_store(event_type, recorded_at);
CREATE INDEX idx_event_store_org ON event_store(org_id, recorded_at);
CREATE INDEX idx_event_store_time ON event_store(recorded_at);

-- Partition by month for time-based retention and query performance
-- ALTER TABLE event_store PARTITION BY RANGE (recorded_at);
```

### Event Type Catalogue

```sql
-- Reference: all known event types and their schema versions
CREATE TABLE event_type_registry (
    event_type VARCHAR(100) PRIMARY KEY,
    category VARCHAR(50) NOT NULL,  -- 'telemetry', 'shipment', 'excursion', 'compliance', 'system'
    schema_version INT NOT NULL DEFAULT 1,
    payload_schema JSONB NOT NULL,   -- JSON Schema for payload validation
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Payload Examples

**SensorReadingRecorded:**
```json
{
  "sensor_id": "550e8400-e29b-41d4-a716-446655440001",
  "shipment_id": "550e8400-e29b-41d4-a716-446655440002",
  "reading_type": "temperature",
  "value": 4.7,
  "unit": "CEL",
  "latitude": 51.5074,
  "longitude": -0.1278,
  "sensor_timestamp": "2026-05-20T14:30:00Z"
}
```

**ShipmentDeparted:**
```json
{
  "shipment_id": "550e8400-e29b-41d4-a716-446655440002",
  "shipment_number": "SHP-2026-00142",
  "origin_location_id": "550e8400-e29b-41d4-a716-446655440003",
  "destination_location_id": "550e8400-e29b-41d4-a716-446655440004",
  "carrier_org_id": "550e8400-e29b-41d4-a716-446655440005",
  "product_id": "550e8400-e29b-41d4-a716-446655440006",
  "batch_lot_number": "LOT-2026-A1",
  "departed_by": "550e8400-e29b-41d4-a716-446655440007",
  "sensors_assigned": ["550e8400-e29b-41d4-a716-446655440001"]
}
```

**ExcursionDetected:**
```json
{
  "excursion_id": "550e8400-e29b-41d4-a716-446655440010",
  "shipment_id": "550e8400-e29b-41d4-a716-446655440002",
  "sensor_id": "550e8400-e29b-41d4-a716-446655440001",
  "excursion_type": "high_temp",
  "severity": "critical",
  "threshold_value": 8.0,
  "observed_value": 12.3,
  "unit": "CEL",
  "detected_at": "2026-05-20T15:45:00Z"
}
```

**ElectronicSignatureApplied:**
```json
{
  "signature_id": "550e8400-e29b-41d4-a716-446655440020",
  "target_stream_id": "550e8400-e29b-41d4-a716-446655440010",
  "target_stream_type": "excursion",
  "meaning": "approved_for_release",
  "signer_full_name": "Dr. Jane Smith",
  "signer_email": "jane.smith@pharma.com",
  "credentials_verified": true,
  "reason": "Temperature excursion duration within product stability budget per ICH Q1A"
}
```

**DeviationInvestigationCompleted:**
```json
{
  "investigation_id": "550e8400-e29b-41d4-a716-446655440030",
  "excursion_id": "550e8400-e29b-41d4-a716-446655440010",
  "root_cause": "Refrigeration unit compressor failure during highway segment",
  "impact_assessment": "Product exposed to >8°C for 47 minutes. Within stability budget.",
  "corrective_action": "Carrier scheduled compressor maintenance",
  "preventive_action": "Added predictive maintenance monitoring to carrier fleet",
  "product_disposition": "release",
  "ai_generated_summary": "Based on the 47-minute excursion peaking at 12.3°C...",
  "investigator_id": "550e8400-e29b-41d4-a716-446655440007",
  "reviewer_id": "550e8400-e29b-41d4-a716-446655440008"
}
```

---

## Materialised Read Models (Projections)

These tables are derived from the event store and can be rebuilt at any time by replaying events.

### Shipment Read Model

```sql
-- Current state of shipments (projected from events)
CREATE TABLE rm_shipments (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL,
    shipment_number VARCHAR(100) NOT NULL,
    status VARCHAR(30) NOT NULL,
    product_name VARCHAR(255),
    product_type VARCHAR(50),
    origin_name VARCHAR(255),
    destination_name VARCHAR(255),
    carrier_name VARCHAR(255),
    batch_lot_number VARCHAR(100),
    temp_range_min DECIMAL(6,2),
    temp_range_max DECIMAL(6,2),
    departed_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    current_temp DECIMAL(6,2),
    current_latitude DECIMAL(10,7),
    current_longitude DECIMAL(10,7),
    excursion_count INT NOT NULL DEFAULT 0,
    has_active_excursion BOOLEAN NOT NULL DEFAULT false,
    last_reading_at TIMESTAMPTZ,
    last_event_version INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_shipments_org_status ON rm_shipments(org_id, status);
CREATE INDEX idx_rm_shipments_active_excursion ON rm_shipments(org_id) WHERE has_active_excursion = true;
```

### Sensor Telemetry Read Model (Time-Series Optimised)

```sql
-- Telemetry summary per sensor per hour (projected from SensorReadingRecorded events)
CREATE TABLE rm_sensor_telemetry_hourly (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id UUID NOT NULL,
    shipment_id UUID,
    hour_bucket TIMESTAMPTZ NOT NULL,  -- truncated to hour
    reading_type VARCHAR(30) NOT NULL,
    reading_count INT NOT NULL DEFAULT 0,
    min_value DECIMAL(10,4),
    max_value DECIMAL(10,4),
    avg_value DECIMAL(10,4),
    stddev_value DECIMAL(10,4),
    unit VARCHAR(10) NOT NULL,
    has_excursion BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(sensor_id, hour_bucket, reading_type)
);

CREATE INDEX idx_rm_telemetry_sensor_time ON rm_sensor_telemetry_hourly(sensor_id, hour_bucket);
CREATE INDEX idx_rm_telemetry_shipment_time ON rm_sensor_telemetry_hourly(shipment_id, hour_bucket);

-- Latest reading per sensor (for real-time dashboard)
CREATE TABLE rm_sensor_latest (
    sensor_id UUID PRIMARY KEY,
    org_id UUID NOT NULL,
    shipment_id UUID,
    status VARCHAR(30) NOT NULL,
    last_temperature DECIMAL(6,2),
    last_humidity DECIMAL(5,2),
    last_latitude DECIMAL(10,7),
    last_longitude DECIMAL(10,7),
    last_battery_pct DECIMAL(5,2),
    last_reading_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_sensor_latest_org ON rm_sensor_latest(org_id);
CREATE INDEX idx_rm_sensor_latest_shipment ON rm_sensor_latest(shipment_id);
```

### Excursion Read Model

```sql
-- Current excursion state (projected from excursion events)
CREATE TABLE rm_excursions (
    id UUID PRIMARY KEY,
    shipment_id UUID NOT NULL,
    shipment_number VARCHAR(100),
    sensor_id UUID NOT NULL,
    excursion_type VARCHAR(30) NOT NULL,
    severity VARCHAR(20) NOT NULL,
    status VARCHAR(30) NOT NULL,
    threshold_value DECIMAL(10,4),
    peak_value DECIMAL(10,4),
    unit VARCHAR(10) NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    investigation_status VARCHAR(30),
    product_disposition VARCHAR(30),
    org_id UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_excursions_org_status ON rm_excursions(org_id, status);
CREATE INDEX idx_rm_excursions_shipment ON rm_excursions(shipment_id);
```

### Lane Analytics Read Model

```sql
-- Lane performance aggregates (projected from shipment and excursion events)
CREATE TABLE rm_lane_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL,
    origin_location_id UUID,
    destination_location_id UUID,
    carrier_org_id UUID,
    lane_name VARCHAR(255),
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    shipment_count INT NOT NULL DEFAULT 0,
    excursion_count INT NOT NULL DEFAULT 0,
    excursion_rate DECIMAL(5,4),
    avg_transit_hours DECIMAL(8,2),
    avg_peak_deviation_celsius DECIMAL(6,2),
    risk_score DECIMAL(5,2),
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, origin_location_id, destination_location_id, carrier_org_id, period_start)
);

CREATE INDEX idx_rm_lane_analytics_org ON rm_lane_analytics(org_id);
CREATE INDEX idx_rm_lane_analytics_risk ON rm_lane_analytics(risk_score DESC);
```

### Compliance Report Read Model

```sql
-- Audit event timeline (projected from all events — for compliance UI)
CREATE TABLE rm_audit_timeline (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL,
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    summary TEXT NOT NULL,  -- human-readable event description
    user_name VARCHAR(255),
    user_email VARCHAR(255),
    recorded_at TIMESTAMPTZ NOT NULL,
    event_id UUID NOT NULL REFERENCES event_store(event_id)
);

CREATE INDEX idx_rm_audit_timeline_org ON rm_audit_timeline(org_id, recorded_at DESC);
CREATE INDEX idx_rm_audit_timeline_stream ON rm_audit_timeline(stream_id, recorded_at);
```

---

## Reference Data (Non-Event Tables)

```sql
-- These are configuration/reference tables, not event-sourced

CREATE TABLE ref_organisations (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL,
    gln VARCHAR(13),
    country_code CHAR(2) NOT NULL,
    compliance_frameworks TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_users (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES ref_organisations(id),
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_products (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES ref_organisations(id),
    name VARCHAR(255) NOT NULL,
    gtin VARCHAR(14),
    product_type VARCHAR(50),
    temp_range_min DECIMAL(6,2),
    temp_range_max DECIMAL(6,2),
    shelf_life_hours INT
);

CREATE TABLE ref_locations (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES ref_organisations(id),
    name VARCHAR(255) NOT NULL,
    location_type VARCHAR(50) NOT NULL,
    gln VARCHAR(13),
    country_code CHAR(2),
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7)
);

CREATE TABLE ref_sensors (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES ref_organisations(id),
    serial_number VARCHAR(100) NOT NULL,
    device_type VARCHAR(50) NOT NULL,
    capabilities TEXT[] NOT NULL DEFAULT '{}',
    status VARCHAR(30) NOT NULL DEFAULT 'available'
);

CREATE TABLE ref_alert_profiles (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES ref_organisations(id),
    name VARCHAR(100) NOT NULL,
    temp_low_critical DECIMAL(6,2),
    temp_high_critical DECIMAL(6,2),
    temp_low_warn DECIMAL(6,2),
    temp_high_warn DECIMAL(6,2)
);
```

---

## Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots of aggregate state to avoid full event replay
CREATE TABLE event_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    snapshot_version INT NOT NULL,      -- event_version at time of snapshot
    state JSONB NOT NULL,               -- serialised aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, snapshot_version)
);

CREATE INDEX idx_event_snapshots_stream ON event_snapshots(stream_id, snapshot_version DESC);
```

---

## Example Queries

### Replay shipment history to a specific point in time

```sql
-- "What was the state of shipment X at 3pm on May 15th?"
SELECT event_type, payload, recorded_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440002'
  AND stream_type = 'shipment'
  AND recorded_at <= '2026-05-15T15:00:00Z'
ORDER BY event_version ASC;
```

### Get all excursion events for a shipment with full audit trail

```sql
-- Complete excursion lifecycle: detection → acknowledgment → investigation → disposition
SELECT e.event_type, e.payload, e.recorded_at, e.recorded_by, u.full_name
FROM event_store e
LEFT JOIN ref_users u ON e.recorded_by = u.id
WHERE e.stream_type = 'excursion'
  AND e.payload->>'shipment_id' = '550e8400-e29b-41d4-a716-446655440002'
ORDER BY e.recorded_at ASC;
```

### Build EPCIS 2.0 export from events

```sql
-- Export sensor events in EPCIS-compatible structure
SELECT
    e.event_id,
    e.recorded_at AS "eventTime",
    'OBSERVE' AS "action",
    e.payload->>'reading_type' AS sensor_type,
    e.payload->>'value' AS value,
    e.payload->>'unit' AS uom,
    e.payload->>'latitude' AS latitude,
    e.payload->>'longitude' AS longitude
FROM event_store e
WHERE e.event_type = 'SensorReadingRecorded'
  AND e.payload->>'shipment_id' = '550e8400-e29b-41d4-a716-446655440002'
  AND e.recorded_at BETWEEN '2026-05-15' AND '2026-05-20'
ORDER BY e.recorded_at ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store, event_type_registry |
| Read Models — Shipments | 1 | rm_shipments |
| Read Models — Telemetry | 2 | rm_sensor_telemetry_hourly, rm_sensor_latest |
| Read Models — Excursions | 1 | rm_excursions |
| Read Models — Analytics | 1 | rm_lane_analytics |
| Read Models — Compliance | 1 | rm_audit_timeline |
| Reference Data | 6 | ref_organisations, ref_users, ref_products, ref_locations, ref_sensors, ref_alert_profiles |
| Snapshots | 1 | event_snapshots |
| **Total** | **15** | Read models are disposable and rebuildable |

---

## Key Design Decisions

1. **Single event store table with JSONB payload**: Rather than separate tables per event type, a single `event_store` table with a typed JSONB `payload` column keeps the write path simple and allows new event types to be added without DDL changes. The `event_type_registry` provides schema validation.

2. **Stream-based organisation**: Events are grouped by `stream_id` (the aggregate root, typically a shipment or sensor). The `UNIQUE(stream_id, event_version)` constraint provides optimistic concurrency control — two concurrent commands against the same shipment will not silently overwrite each other.

3. **Read models are explicitly disposable**: Every `rm_*` table can be dropped and rebuilt by replaying events from `event_store`. This means new read models (e.g., a new dashboard view) can be added without modifying the event store.

4. **Sensor telemetry as events, not separate time-series**: Every sensor reading is a `SensorReadingRecorded` event in the event store. The `rm_sensor_telemetry_hourly` read model provides pre-aggregated time-series data for dashboards. This trades storage efficiency for audit completeness.

5. **Electronic signatures as events**: Rather than a separate signatures table, signature events (`ElectronicSignatureApplied`) are recorded in the event store, providing an immutable, time-ordered record that satisfies 21 CFR Part 11. The signature event references the target stream for cross-referencing.

6. **Snapshots for long-lived aggregates**: Shipments with thousands of sensor readings would be expensive to replay from event zero. The `event_snapshots` table stores periodic state snapshots; replay starts from the latest snapshot rather than the beginning.

7. **Reference data outside the event store**: Organisations, users, products, and locations are not event-sourced because they change infrequently and are referenced by many events. They use simple CRUD tables prefixed with `ref_`.

8. **EPCIS export as a projection**: EPCIS 2.0 events are not stored separately — they are generated on-demand by querying the event store and transforming internal events into EPCIS JSON-LD format. This avoids duplicating data and ensures exports always reflect the latest event history.
