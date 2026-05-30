# Data Model Suggestion 4: Time-Series First (TimescaleDB)

> Project: Cold Chain Monitoring · Created: 2026-05-20

## Philosophy

This model inverts the typical application-database priority. Instead of designing around the business entities (shipments, organisations) and treating sensor data as a subordinate table, it places time-series telemetry at the architectural centre. The insight: a cold chain monitoring platform at scale is fundamentally a sensor data pipeline that happens to have a business layer on top.

TimescaleDB, a PostgreSQL extension, provides hypertables — tables that look and query like standard PostgreSQL tables but are automatically partitioned by time into chunks. This enables transparent time-range queries, automatic data compression of older chunks (10-20x storage reduction), continuous aggregates (materialised views that refresh incrementally), and data retention policies — all critical for a system ingesting millions of sensor readings per day.

The operational tables (organisations, shipments, sensors) remain standard PostgreSQL tables with full relational integrity. The time-series hypertables (sensor readings, alerts, audit events) use TimescaleDB for time-partitioned storage and query optimisation. This dual-layer approach is similar to how Azure IoT Hub + Time Series Insights or AWS IoT Core + Timestream work, but self-hosted on PostgreSQL.

**Best for:** High-volume sensor deployments (thousands of sensors, sub-minute reporting intervals), analytics-heavy use cases with lane performance dashboards and trend analysis, and organisations that need long-term data retention (years of historical readings) with efficient storage.

**Trade-offs:**
- (+) Purpose-built for time-series queries (range scans, aggregations, downsampling)
- (+) Automatic time-based partitioning — no manual partition management
- (+) Compression reduces storage 10-20x for older data while maintaining queryability
- (+) Continuous aggregates provide real-time dashboards without expensive GROUP BY queries
- (+) Data retention policies automatically drop data beyond retention windows
- (+) Still PostgreSQL — standard SQL, joins with relational tables, same tooling ecosystem
- (-) Requires TimescaleDB extension (not vanilla PostgreSQL); managed options available (Timescale Cloud)
- (-) Hypertables have restrictions: no unique constraints spanning chunks without time column
- (-) Compression makes individual row updates impossible (must decompress chunk first)
- (-) Less suited for event replay patterns (event-sourced model is better for that)
- (-) Team must understand chunk management, compression, and retention policies

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 / ISO IEC 19987:2024 | `epcis_exports` table stores EPCIS event payloads; sensorElementList aggregated from hypertable readings using continuous aggregates |
| FDA 21 CFR Part 11 | `audit_events` hypertable provides time-partitioned, append-only audit log; electronic signatures in relational table |
| EU GDP 2013/C 68/01 | Temperature range compliance queries use continuous aggregates for real-time GDP reporting |
| HACCP | CCP monitoring readings stored in `sensor_readings` hypertable with CCP location tags |
| ISO 31512:2024 | Shipment lifecycle tracked in relational tables aligned with ISO 31512 process stages |
| MQTT v5.0 / ISO IEC 20922 | Sensor telemetry ingested via MQTT and written directly to hypertables |

---

## Relational Tables (Standard PostgreSQL)

```sql
-- Organisations
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL CHECK (org_type IN ('shipper', 'carrier', '3pl', 'warehouse', 'manufacturer', 'distributor')),
    parent_org_id UUID REFERENCES organisations(id),
    gln VARCHAR(13),
    country_code CHAR(2) NOT NULL,
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    compliance_frameworks TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL CHECK (role IN ('admin', 'quality_manager', 'logistics_operator', 'read_only', 'system')),
    is_active BOOLEAN NOT NULL DEFAULT true,
    password_hash VARCHAR(255),
    mfa_enabled BOOLEAN NOT NULL DEFAULT false,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(org_id);

-- Sensors
CREATE TABLE sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    serial_number VARCHAR(100) NOT NULL,
    device_type VARCHAR(50) NOT NULL CHECK (device_type IN ('ble', 'cellular', 'lorawan', 'satellite', 'wired', 'passive_label')),
    manufacturer VARCHAR(100),
    model VARCHAR(100),
    firmware_version VARCHAR(50),
    capabilities TEXT[] NOT NULL DEFAULT '{}',
    accuracy_celsius DECIMAL(4,2),
    calibration_date DATE,
    calibration_due_date DATE,
    protocol VARCHAR(20) CHECK (protocol IN ('mqtt', 'http', 'coap', 'lorawan', 'amqp')),
    mqtt_topic VARCHAR(500),
    reporting_interval_seconds INT NOT NULL DEFAULT 300,
    status VARCHAR(30) NOT NULL DEFAULT 'available' CHECK (status IN ('available', 'deployed', 'in_transit', 'maintenance', 'retired')),
    last_seen_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, serial_number)
);

CREATE INDEX idx_sensors_org_status ON sensors(org_id, status);

-- Products
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100),
    gtin VARCHAR(14),
    product_type VARCHAR(50) CHECK (product_type IN ('pharmaceutical', 'vaccine', 'biologic', 'food', 'produce', 'clinical_trial', 'other')),
    temp_range_min DECIMAL(6,2) NOT NULL,
    temp_range_max DECIMAL(6,2) NOT NULL,
    shelf_life_hours INT,
    regulatory_class VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_org ON products(org_id);

-- Locations
CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    location_type VARCHAR(50) NOT NULL CHECK (location_type IN ('warehouse', 'cold_storage', 'dock', 'checkpoint', 'pharmacy', 'hospital', 'retail', 'airport', 'port')),
    gln VARCHAR(13),
    country_code CHAR(2) NOT NULL,
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    address_line1 VARCHAR(255),
    city VARCHAR(100),
    postal_code VARCHAR(20),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_org ON locations(org_id);

-- Shipments
CREATE TABLE shipments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    shipment_number VARCHAR(100) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'created' CHECK (status IN ('created', 'packed', 'in_transit', 'at_checkpoint', 'delivered', 'quarantined', 'released', 'rejected', 'cancelled')),
    product_id UUID NOT NULL REFERENCES products(id),
    origin_location_id UUID REFERENCES locations(id),
    destination_location_id UUID REFERENCES locations(id),
    carrier_org_id UUID REFERENCES organisations(id),
    batch_lot_number VARCHAR(100),
    quantity INT,
    temp_low_critical DECIMAL(6,2),
    temp_high_critical DECIMAL(6,2),
    temp_low_warn DECIMAL(6,2),
    temp_high_warn DECIMAL(6,2),
    expected_departure_at TIMESTAMPTZ,
    actual_departure_at TIMESTAMPTZ,
    expected_arrival_at TIMESTAMPTZ,
    actual_arrival_at TIMESTAMPTZ,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, shipment_number)
);

CREATE INDEX idx_shipments_org_status ON shipments(org_id, status);
CREATE INDEX idx_shipments_dates ON shipments(actual_departure_at, actual_arrival_at);

-- Shipment-sensor assignments
CREATE TABLE shipment_sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    removed_at TIMESTAMPTZ,
    assigned_by UUID REFERENCES users(id),
    UNIQUE(shipment_id, sensor_id, assigned_at)
);

-- Excursions (operational workflow)
CREATE TABLE excursions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    excursion_type VARCHAR(30) NOT NULL CHECK (excursion_type IN ('high_temp', 'low_temp', 'high_humidity', 'low_humidity', 'shock', 'light_exposure', 'door_open')),
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('warning', 'critical')),
    status VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'resolved', 'acknowledged', 'investigating', 'closed')),
    threshold_value DECIMAL(10,4) NOT NULL,
    peak_value DECIMAL(10,4),
    unit VARCHAR(10) NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    investigation_number VARCHAR(50),
    root_cause TEXT,
    corrective_action TEXT,
    product_disposition VARCHAR(30),
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_excursions_shipment ON excursions(shipment_id);
CREATE INDEX idx_excursions_status ON excursions(status);

-- Electronic signatures (21 CFR Part 11)
CREATE TABLE electronic_signatures (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    signature_meaning VARCHAR(100) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    credentials_verified BOOLEAN NOT NULL DEFAULT true,
    signed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_signatures_record ON electronic_signatures(table_name, record_id);

-- Lanes
CREATE TABLE lanes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    origin_location_id UUID REFERENCES locations(id),
    destination_location_id UUID REFERENCES locations(id),
    carrier_org_id UUID REFERENCES organisations(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Time-Series Hypertables (TimescaleDB)

```sql
-- ============================================================
-- SENSOR READINGS HYPERTABLE
-- The core time-series table for all sensor telemetry
-- ============================================================
CREATE TABLE sensor_readings (
    time TIMESTAMPTZ NOT NULL,          -- TimescaleDB requires 'time' as partition column
    sensor_id UUID NOT NULL,
    shipment_id UUID,
    org_id UUID NOT NULL,               -- denormalised for partition pruning
    -- Primary measurements
    temperature DECIMAL(6,2),
    humidity DECIMAL(5,2),
    shock_g DECIMAL(6,2),
    light_lux DECIMAL(8,2),
    pressure_hpa DECIMAL(8,2),
    door_status SMALLINT,               -- 0=closed, 1=open
    battery_pct DECIMAL(5,2),
    -- Location
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    -- Flags
    is_excursion BOOLEAN NOT NULL DEFAULT false,
    excursion_id UUID
);

-- Convert to hypertable (auto-partitioned by time)
SELECT create_hypertable('sensor_readings', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Indexes on hypertable
CREATE INDEX idx_readings_sensor ON sensor_readings(sensor_id, time DESC);
CREATE INDEX idx_readings_shipment ON sensor_readings(shipment_id, time DESC) WHERE shipment_id IS NOT NULL;
CREATE INDEX idx_readings_org ON sensor_readings(org_id, time DESC);
CREATE INDEX idx_readings_excursion ON sensor_readings(shipment_id, time DESC) WHERE is_excursion = true;


-- ============================================================
-- CONTINUOUS AGGREGATES — pre-computed roll-ups
-- ============================================================

-- Hourly temperature statistics per sensor
CREATE MATERIALIZED VIEW sensor_readings_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    sensor_id,
    shipment_id,
    org_id,
    COUNT(*) AS reading_count,
    MIN(temperature) AS temp_min,
    MAX(temperature) AS temp_max,
    AVG(temperature) AS temp_avg,
    STDDEV(temperature) AS temp_stddev,
    MIN(humidity) AS humidity_min,
    MAX(humidity) AS humidity_max,
    AVG(humidity) AS humidity_avg,
    MAX(shock_g) AS shock_max,
    bool_or(is_excursion) AS has_excursion
FROM sensor_readings
GROUP BY bucket, sensor_id, shipment_id, org_id;

-- Refresh policy: update every 30 minutes, covering last 2 hours
SELECT add_continuous_aggregate_policy('sensor_readings_hourly',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes');

-- Daily temperature statistics per shipment
CREATE MATERIALIZED VIEW sensor_readings_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    shipment_id,
    org_id,
    COUNT(*) AS reading_count,
    MIN(temperature) AS temp_min,
    MAX(temperature) AS temp_max,
    AVG(temperature) AS temp_avg,
    COUNT(*) FILTER (WHERE is_excursion = true) AS excursion_reading_count
FROM sensor_readings
WHERE shipment_id IS NOT NULL
GROUP BY bucket, shipment_id, org_id;

SELECT add_continuous_aggregate_policy('sensor_readings_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- COMPRESSION POLICY — compress chunks older than 7 days
-- ============================================================
ALTER TABLE sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id, shipment_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');

-- ============================================================
-- RETENTION POLICY — drop raw data older than 2 years
-- (continuous aggregates retain summaries indefinitely)
-- ============================================================
SELECT add_retention_policy('sensor_readings', INTERVAL '2 years');


-- ============================================================
-- ALERT EVENTS HYPERTABLE
-- All alert and notification events, time-partitioned
-- ============================================================
CREATE TABLE alert_events (
    time TIMESTAMPTZ NOT NULL,
    org_id UUID NOT NULL,
    excursion_id UUID,
    shipment_id UUID,
    sensor_id UUID,
    alert_type VARCHAR(50) NOT NULL,  -- 'threshold_breach', 'sensor_offline', 'battery_low', 'geofence_exit'
    severity VARCHAR(20) NOT NULL,
    channel VARCHAR(20),  -- 'email', 'sms', 'webhook', 'push'
    recipient VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    message TEXT
);

SELECT create_hypertable('alert_events', 'time',
    chunk_time_interval => INTERVAL '1 week');

CREATE INDEX idx_alert_events_org ON alert_events(org_id, time DESC);
CREATE INDEX idx_alert_events_shipment ON alert_events(shipment_id, time DESC) WHERE shipment_id IS NOT NULL;


-- ============================================================
-- AUDIT EVENTS HYPERTABLE
-- Append-only audit log, time-partitioned
-- ============================================================
CREATE TABLE audit_events (
    time TIMESTAMPTZ NOT NULL DEFAULT now(),
    org_id UUID NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    field_name VARCHAR(100),
    old_value TEXT,
    new_value TEXT,
    user_id UUID,
    user_email VARCHAR(255),
    reason TEXT,
    ip_address INET
);

SELECT create_hypertable('audit_events', 'time',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_audit_events_record ON audit_events(table_name, record_id, time DESC);
CREATE INDEX idx_audit_events_user ON audit_events(user_id, time DESC) WHERE user_id IS NOT NULL;

-- Compress audit events after 30 days (but never delete — regulatory retention)
ALTER TABLE audit_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'table_name, record_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('audit_events', INTERVAL '30 days');
-- No retention policy on audit_events — regulatory requirement to keep indefinitely


-- ============================================================
-- CUSTODY EVENTS HYPERTABLE
-- Shipment lifecycle events, time-partitioned
-- ============================================================
CREATE TABLE custody_events (
    time TIMESTAMPTZ NOT NULL DEFAULT now(),
    shipment_id UUID NOT NULL,
    org_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL CHECK (event_type IN ('created', 'packed', 'departed', 'checkpoint_arrival', 'checkpoint_departure', 'delivered', 'quarantined', 'released', 'rejected', 'handover')),
    location_id UUID,
    from_org_id UUID,
    to_org_id UUID,
    performed_by UUID,
    notes TEXT,
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7)
);

SELECT create_hypertable('custody_events', 'time',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_custody_events_shipment ON custody_events(shipment_id, time DESC);
CREATE INDEX idx_custody_events_org ON custody_events(org_id, time DESC);


-- ============================================================
-- ML PREDICTIONS HYPERTABLE
-- AI excursion predictions over time
-- ============================================================
CREATE TABLE ml_predictions (
    time TIMESTAMPTZ NOT NULL DEFAULT now(),
    shipment_id UUID NOT NULL,
    org_id UUID NOT NULL,
    model_version VARCHAR(50) NOT NULL,
    predicted_type VARCHAR(30),
    probability DECIMAL(5,4) NOT NULL,
    predicted_breach_at TIMESTAMPTZ,
    recommended_action TEXT,
    features JSONB
);

SELECT create_hypertable('ml_predictions', 'time',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_ml_predictions_shipment ON ml_predictions(shipment_id, time DESC);
```

---

## Example Queries

### Real-time dashboard: latest reading per sensor for an org

```sql
-- Uses TimescaleDB last() aggregate for efficient "latest value" queries
SELECT DISTINCT ON (sensor_id)
    sensor_id, time, temperature, humidity, latitude, longitude, battery_pct, is_excursion
FROM sensor_readings
WHERE org_id = '...'
  AND time > now() - INTERVAL '1 hour'
ORDER BY sensor_id, time DESC;
```

### Temperature trend for a shipment (using continuous aggregate)

```sql
SELECT bucket, temp_min, temp_max, temp_avg, has_excursion
FROM sensor_readings_hourly
WHERE shipment_id = '...'
  AND bucket BETWEEN '2026-05-15' AND '2026-05-20'
ORDER BY bucket;
```

### Lane performance analysis over 90 days

```sql
SELECT
    l.name AS lane_name,
    COUNT(DISTINCT s.id) AS shipment_count,
    COUNT(DISTINCT e.id) AS excursion_count,
    ROUND(COUNT(DISTINCT e.id)::numeric / NULLIF(COUNT(DISTINCT s.id), 0) * 100, 1) AS excursion_rate_pct,
    AVG(EXTRACT(EPOCH FROM (s.actual_arrival_at - s.actual_departure_at)) / 3600) AS avg_transit_hours
FROM lanes l
JOIN shipments s ON s.org_id = l.org_id
    AND s.origin_location_id = l.origin_location_id
    AND s.destination_location_id = l.destination_location_id
LEFT JOIN excursions e ON e.shipment_id = s.id
WHERE l.org_id = '...'
  AND s.actual_departure_at > now() - INTERVAL '90 days'
GROUP BY l.id, l.name
ORDER BY excursion_rate_pct DESC;
```

### Compliance query: all readings outside GDP range for a shipment

```sql
SELECT sr.time, sr.temperature, sr.sensor_id,
       p.temp_range_min, p.temp_range_max
FROM sensor_readings sr
JOIN shipments s ON sr.shipment_id = s.id
JOIN products p ON s.product_id = p.id
WHERE sr.shipment_id = '...'
  AND (sr.temperature < p.temp_range_min OR sr.temperature > p.temp_range_max)
ORDER BY sr.time;
```

### Storage efficiency: check compression ratio

```sql
SELECT
    hypertable_name,
    pg_size_pretty(before_compression_total_bytes) AS before,
    pg_size_pretty(after_compression_total_bytes) AS after,
    ROUND((1 - after_compression_total_bytes::numeric / before_compression_total_bytes) * 100, 1) AS compression_pct
FROM timescaledb_information.compressed_hypertable_stats;
```

---

## Table Count Summary

| Category | Tables | Type | Notes |
|----------|--------|------|-------|
| Identity & Multi-Tenancy | 2 | Relational | organisations, users |
| Sensors & Devices | 1 | Relational | sensors |
| Products & Locations | 2 | Relational | products, locations |
| Shipments & Workflow | 3 | Relational | shipments, shipment_sensors, lanes |
| Excursions | 1 | Relational | excursions |
| Electronic Signatures | 1 | Relational | electronic_signatures |
| Sensor Telemetry | 1 | Hypertable | sensor_readings (auto-partitioned by day) |
| Continuous Aggregates | 2 | Materialised View | sensor_readings_hourly, sensor_readings_daily |
| Alert Events | 1 | Hypertable | alert_events (partitioned by week) |
| Audit Trail | 1 | Hypertable | audit_events (partitioned by month, never deleted) |
| Custody Events | 1 | Hypertable | custody_events (partitioned by month) |
| ML Predictions | 1 | Hypertable | ml_predictions (partitioned by month) |
| **Total** | **17** | | 12 relational + 5 hypertables + 2 continuous aggregates |

---

## Key Design Decisions

1. **TimescaleDB hypertables for all time-oriented data**: Sensor readings, alerts, audit events, custody events, and ML predictions are all append-heavy, time-ordered data. Using hypertables gives automatic time partitioning, compression, and retention policies without manual DDL management.

2. **Denormalised `org_id` on hypertables**: Hypertables cannot have foreign key constraints. To enable efficient tenant-scoped queries and partition pruning, `org_id` is denormalised onto every hypertable row. The application layer enforces referential integrity.

3. **Continuous aggregates replace manual roll-up jobs**: Rather than scheduled ETL jobs that compute hourly/daily summaries, TimescaleDB continuous aggregates maintain pre-computed roll-ups that refresh incrementally. Dashboards query aggregates instead of raw data.

4. **Compression with segment-by sensor_id**: Compressed chunks are segmented by `sensor_id` and `shipment_id`, meaning queries that filter by these columns only decompress relevant segments. This balances compression ratio with query performance.

5. **Retention policy on raw readings, not on aggregates**: Raw sensor readings are dropped after 2 years (configurable per regulatory requirement). Hourly and daily continuous aggregates are retained indefinitely, providing long-term trend data without raw storage costs.

6. **No retention policy on audit_events**: FDA 21 CFR Part 11 and EU GDP require indefinite audit trail retention. Audit events are compressed after 30 days but never deleted. This is a conscious architectural choice despite storage implications.

7. **Flat measurement columns instead of EAV**: Rather than an entity-attribute-value design (`reading_type`, `value`), each measurement gets its own column (`temperature`, `humidity`, `shock_g`). This enables efficient continuous aggregates and avoids the EAV anti-pattern for time-series data. New measurement types require a schema migration, but the set of physical measurements is stable.

8. **Relational tables for operational workflow**: Excursions, shipments, and electronic signatures remain standard PostgreSQL tables because they require UPDATE operations (status changes, investigation workflow). TimescaleDB hypertables are optimised for append; updates on compressed chunks require decompression. The operational tables stay relational while the observational data goes time-series.
