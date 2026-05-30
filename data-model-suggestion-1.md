# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Cold Chain Monitoring · Created: 2026-05-20

## Philosophy

This model follows classical third-normal-form relational design where every domain concept gets its own table with strict foreign key relationships. The schema is designed around the core business entities — organisations, sensors, shipments, and excursions — with junction tables for many-to-many relationships and reference tables for controlled vocabularies.

This approach mirrors how pharmaceutical-grade systems like Sensitech ColdStream and Controlant structure their data: every record is independently auditable, every relationship is explicit, and every field has a single authoritative source. The data integrity guarantees of PostgreSQL foreign keys and CHECK constraints directly support FDA 21 CFR Part 11 requirements for tamper-evident electronic records.

The trade-off is schema rigidity. Adding a new sensor type, a new jurisdiction-specific field, or a new compliance framework means DDL migrations. But for a domain where regulatory compliance is paramount, this rigidity is a feature — it prevents unstructured data from undermining audit integrity.

**Best for:** Pharmaceutical and vaccine cold chain deployments where regulatory compliance (21 CFR Part 11, EU GDP, WHO TRS 961) is the primary concern and data integrity cannot be compromised.

**Trade-offs:**
- (+) Strongest referential integrity; every relationship enforced at database level
- (+) Straightforward SQL queries for compliance reporting and auditing
- (+) Well-understood by enterprise development teams
- (+) Natural fit for 21 CFR Part 11 audit trail requirements
- (-) High table count (~45+ tables); schema migrations required for new entity types
- (-) JSONB flexibility sacrificed for strict typing
- (-) Multi-sensor telemetry at high frequency may strain join-heavy queries
- (-) Adding jurisdiction-specific fields requires DDL changes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 / ISO IEC 19987:2024 | `epcis_events` table models EPCIS event types (ObjectEvent, AggregationEvent, TransformationEvent); `sensor_reports` mirrors sensorElementList/sensorReport structure |
| FDA 21 CFR Part 11 | `audit_trail` table captures all record changes with user, timestamp, old/new values; `electronic_signatures` table stores signing events with meaning codes |
| EU GDP 2013/C 68/01 | `temperature_ranges` reference table defines GDP-compliant storage ranges (2-8°C, -20°C); `deviation_investigations` table tracks GDP deviation workflow |
| ISO 31512:2024 | Shipment and storage requirement fields align with ISO 31512 cold chain logistics service requirements |
| ISO 31510:2025 | Column names and enum values adopt ISO 31510 cold chain vocabulary where applicable |
| HACCP | `critical_control_points` table models CCP monitoring locations; `ccp_checks` logs temperature readings at CCPs |
| ISO 3166-1/2 | `jurisdictions` reference table uses ISO 3166 country and subdivision codes |
| MQTT v5.0 / ISO IEC 20922 | `sensor_connections` table tracks MQTT client IDs, topics, and QoS levels per device |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisations (tenants)
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL CHECK (org_type IN ('shipper', 'carrier', '3pl', 'warehouse', 'manufacturer', 'distributor')),
    parent_org_id UUID REFERENCES organisations(id),
    gln VARCHAR(13),  -- GS1 Global Location Number
    lei VARCHAR(20),  -- ISO 17442 Legal Entity Identifier
    country_code CHAR(2) NOT NULL,  -- ISO 3166-1 alpha-2
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    compliance_frameworks TEXT[] NOT NULL DEFAULT '{}',  -- e.g. {'fda_21cfr11', 'eu_gdp', 'haccp'}
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisations_parent ON organisations(parent_org_id);
CREATE INDEX idx_organisations_country ON organisations(country_code);

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
CREATE INDEX idx_users_email ON users(email);
```

## Sensor & Device Management

```sql
-- Sensor device registry
CREATE TABLE sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    serial_number VARCHAR(100) NOT NULL,
    device_type VARCHAR(50) NOT NULL CHECK (device_type IN ('ble', 'cellular', 'lorawan', 'satellite', 'wired', 'passive_label')),
    manufacturer VARCHAR(100),
    model VARCHAR(100),
    firmware_version VARCHAR(50),
    capabilities TEXT[] NOT NULL DEFAULT '{}',  -- e.g. {'temperature', 'humidity', 'shock', 'light', 'gps'}
    calibration_date DATE,
    calibration_due_date DATE,
    calibration_certificate_url VARCHAR(500),
    accuracy_celsius DECIMAL(4,2),  -- e.g. 0.50 for ±0.5°C
    is_reusable BOOLEAN NOT NULL DEFAULT true,
    status VARCHAR(30) NOT NULL DEFAULT 'available' CHECK (status IN ('available', 'deployed', 'in_transit', 'maintenance', 'retired')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, serial_number)
);

CREATE INDEX idx_sensors_org ON sensors(org_id);
CREATE INDEX idx_sensors_status ON sensors(status);
CREATE INDEX idx_sensors_calibration_due ON sensors(calibration_due_date);

-- MQTT/connectivity configuration per sensor
CREATE TABLE sensor_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    protocol VARCHAR(20) NOT NULL CHECK (protocol IN ('mqtt', 'http', 'coap', 'lorawan', 'amqp')),
    mqtt_client_id VARCHAR(255),
    mqtt_topic VARCHAR(500),
    qos_level SMALLINT DEFAULT 1 CHECK (qos_level IN (0, 1, 2)),
    reporting_interval_seconds INT NOT NULL DEFAULT 300,
    last_seen_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sensor_connections_sensor ON sensor_connections(sensor_id);
```

## Temperature Ranges & Thresholds

```sql
-- Reference table: standard temperature ranges
CREATE TABLE temperature_ranges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,  -- e.g. 'Refrigerated (GDP)', 'Frozen', 'Cryogenic'
    min_celsius DECIMAL(6,2) NOT NULL,
    max_celsius DECIMAL(6,2) NOT NULL,
    standard_reference VARCHAR(100),  -- e.g. 'EU GDP 2013/C 68/01', 'WHO TRS 961'
    is_system BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert threshold profiles
CREATE TABLE alert_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(100) NOT NULL,
    temperature_range_id UUID REFERENCES temperature_ranges(id),
    temp_low_warn DECIMAL(6,2),
    temp_low_critical DECIMAL(6,2),
    temp_high_warn DECIMAL(6,2),
    temp_high_critical DECIMAL(6,2),
    humidity_low_warn DECIMAL(5,2),
    humidity_high_warn DECIMAL(5,2),
    shock_threshold_g DECIMAL(6,2),
    light_threshold_lux DECIMAL(8,2),
    excursion_delay_seconds INT NOT NULL DEFAULT 0,  -- grace period before alerting
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_profiles_org ON alert_profiles(org_id);
```

## Shipment Lifecycle & Chain of Custody

```sql
-- Products being shipped
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100),
    gtin VARCHAR(14),  -- GS1 Global Trade Item Number
    product_type VARCHAR(50) CHECK (product_type IN ('pharmaceutical', 'vaccine', 'biologic', 'food', 'produce', 'clinical_trial', 'other')),
    required_temp_range_id UUID REFERENCES temperature_ranges(id),
    shelf_life_hours INT,
    regulatory_class VARCHAR(50),  -- e.g. 'prescription_drug', 'otc', 'perishable_food'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_org ON products(org_id);

-- Locations (warehouses, docks, checkpoints)
CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    location_type VARCHAR(50) NOT NULL CHECK (location_type IN ('warehouse', 'cold_storage', 'dock', 'checkpoint', 'pharmacy', 'hospital', 'retail', 'airport', 'port')),
    gln VARCHAR(13),  -- GS1 Global Location Number
    address_line1 VARCHAR(255),
    city VARCHAR(100),
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2) REFERENCES jurisdictions(country_code),
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_org ON locations(org_id);
CREATE INDEX idx_locations_gln ON locations(gln);

-- Jurisdictions reference
CREATE TABLE jurisdictions (
    country_code CHAR(2) PRIMARY KEY,  -- ISO 3166-1 alpha-2
    country_name VARCHAR(100) NOT NULL,
    subdivision_code VARCHAR(6),  -- ISO 3166-2
    applicable_regulations TEXT[] NOT NULL DEFAULT '{}'  -- e.g. {'fda_21cfr11', 'eu_gdp', 'fsma'}
);

-- Shipments
CREATE TABLE shipments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    shipment_number VARCHAR(100) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'created' CHECK (status IN ('created', 'packed', 'in_transit', 'at_checkpoint', 'delivered', 'quarantined', 'released', 'rejected', 'cancelled')),
    product_id UUID NOT NULL REFERENCES products(id),
    alert_profile_id UUID NOT NULL REFERENCES alert_profiles(id),
    origin_location_id UUID REFERENCES locations(id),
    destination_location_id UUID REFERENCES locations(id),
    carrier_org_id UUID REFERENCES organisations(id),
    quantity INT,
    batch_lot_number VARCHAR(100),
    expected_departure_at TIMESTAMPTZ,
    actual_departure_at TIMESTAMPTZ,
    expected_arrival_at TIMESTAMPTZ,
    actual_arrival_at TIMESTAMPTZ,
    lane_id UUID REFERENCES lanes(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, shipment_number)
);

CREATE INDEX idx_shipments_org_status ON shipments(org_id, status);
CREATE INDEX idx_shipments_product ON shipments(product_id);
CREATE INDEX idx_shipments_carrier ON shipments(carrier_org_id);
CREATE INDEX idx_shipments_dates ON shipments(expected_departure_at, expected_arrival_at);

-- Sensor assignments to shipments
CREATE TABLE shipment_sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    removed_at TIMESTAMPTZ,
    assigned_by UUID REFERENCES users(id),
    UNIQUE(shipment_id, sensor_id, assigned_at)
);

CREATE INDEX idx_shipment_sensors_shipment ON shipment_sensors(shipment_id);
CREATE INDEX idx_shipment_sensors_sensor ON shipment_sensors(sensor_id);

-- Chain of custody events
CREATE TABLE custody_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    event_type VARCHAR(50) NOT NULL CHECK (event_type IN ('created', 'packed', 'departed', 'checkpoint_arrival', 'checkpoint_departure', 'delivered', 'quarantined', 'released', 'rejected', 'handover')),
    location_id UUID REFERENCES locations(id),
    from_org_id UUID REFERENCES organisations(id),
    to_org_id UUID REFERENCES organisations(id),
    performed_by UUID REFERENCES users(id),
    notes TEXT,
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_custody_events_shipment ON custody_events(shipment_id);
CREATE INDEX idx_custody_events_recorded ON custody_events(recorded_at);

-- Lanes (recurring routes for analytics)
CREATE TABLE lanes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    origin_location_id UUID REFERENCES locations(id),
    destination_location_id UUID REFERENCES locations(id),
    carrier_org_id UUID REFERENCES organisations(id),
    typical_duration_hours DECIMAL(8,2),
    risk_score DECIMAL(5,2),  -- AI-computed 0-100
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lanes_org ON lanes(org_id);
```

## Sensor Telemetry

```sql
-- Sensor readings (primary telemetry table)
CREATE TABLE sensor_readings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    shipment_id UUID REFERENCES shipments(id),
    reading_type VARCHAR(30) NOT NULL CHECK (reading_type IN ('temperature', 'humidity', 'shock', 'light', 'pressure', 'door_status', 'battery', 'location')),
    value_numeric DECIMAL(10,4),
    unit VARCHAR(10) NOT NULL,  -- 'CEL', '%RH', 'g', 'lux', 'hPa' per EPCIS 2.0 UoM codes
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    is_excursion BOOLEAN NOT NULL DEFAULT false,
    recorded_at TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition by month for query performance
CREATE INDEX idx_sensor_readings_sensor_time ON sensor_readings(sensor_id, recorded_at);
CREATE INDEX idx_sensor_readings_shipment_time ON sensor_readings(shipment_id, recorded_at);
CREATE INDEX idx_sensor_readings_excursion ON sensor_readings(shipment_id, is_excursion) WHERE is_excursion = true;
```

## Excursion Management

```sql
-- Excursions (temperature breach events)
CREATE TABLE excursions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    excursion_type VARCHAR(30) NOT NULL CHECK (excursion_type IN ('high_temp', 'low_temp', 'high_humidity', 'low_humidity', 'shock', 'light_exposure', 'door_open')),
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('warning', 'critical')),
    threshold_value DECIMAL(10,4) NOT NULL,
    peak_value DECIMAL(10,4),
    unit VARCHAR(10) NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    status VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'resolved', 'acknowledged', 'investigating', 'closed')),
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_excursions_shipment ON excursions(shipment_id);
CREATE INDEX idx_excursions_status ON excursions(status);
CREATE INDEX idx_excursions_started ON excursions(started_at);

-- Deviation investigations (GDP/FDA compliance)
CREATE TABLE deviation_investigations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    excursion_id UUID NOT NULL REFERENCES excursions(id),
    investigation_number VARCHAR(50) NOT NULL UNIQUE,
    status VARCHAR(30) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'in_progress', 'pending_review', 'closed_released', 'closed_rejected', 'closed_destroyed')),
    root_cause TEXT,
    impact_assessment TEXT,
    corrective_action TEXT,
    preventive_action TEXT,
    product_disposition VARCHAR(30) CHECK (product_disposition IN ('continue', 'quarantine', 'release', 'reject', 'destroy')),
    investigator_id UUID REFERENCES users(id),
    reviewer_id UUID REFERENCES users(id),
    ai_generated_summary TEXT,  -- LLM-drafted investigation report
    opened_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deviation_investigations_excursion ON deviation_investigations(excursion_id);
CREATE INDEX idx_deviation_investigations_status ON deviation_investigations(status);
```

## Alerting & Notifications

```sql
-- Alert rules
CREATE TABLE alert_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    condition_type VARCHAR(50) NOT NULL CHECK (condition_type IN ('threshold_breach', 'geofence_exit', 'eta_delay', 'sensor_offline', 'battery_low', 'door_open')),
    is_active BOOLEAN NOT NULL DEFAULT true,
    notification_channels TEXT[] NOT NULL DEFAULT '{"email"}',  -- email, sms, webhook, push
    escalation_delay_minutes INT DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert notifications sent
CREATE TABLE alert_notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    excursion_id UUID REFERENCES excursions(id),
    alert_rule_id UUID REFERENCES alert_rules(id),
    channel VARCHAR(20) NOT NULL CHECK (channel IN ('email', 'sms', 'webhook', 'push', 'slack', 'teams')),
    recipient VARCHAR(255) NOT NULL,
    subject VARCHAR(500),
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'delivered', 'failed')),
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_notifications_excursion ON alert_notifications(excursion_id);
```

## Compliance & Audit Trail

```sql
-- Audit trail (21 CFR Part 11 compliant)
CREATE TABLE audit_trail (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    field_name VARCHAR(100),
    old_value TEXT,
    new_value TEXT,
    user_id UUID REFERENCES users(id),
    user_email VARCHAR(255),
    ip_address INET,
    user_agent TEXT,
    reason TEXT,  -- required for 21 CFR Part 11 — why the change was made
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_trail_table_record ON audit_trail(table_name, record_id);
CREATE INDEX idx_audit_trail_user ON audit_trail(user_id);
CREATE INDEX idx_audit_trail_recorded ON audit_trail(recorded_at);

-- Electronic signatures (21 CFR Part 11)
CREATE TABLE electronic_signatures (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    signature_meaning VARCHAR(100) NOT NULL,  -- e.g. 'approved', 'reviewed', 'released', 'rejected'
    full_name VARCHAR(255) NOT NULL,
    credentials_verified BOOLEAN NOT NULL DEFAULT true,
    signed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address INET
);

CREATE INDEX idx_electronic_signatures_record ON electronic_signatures(table_name, record_id);
CREATE INDEX idx_electronic_signatures_user ON electronic_signatures(user_id);

-- Compliance reports
CREATE TABLE compliance_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    report_type VARCHAR(50) NOT NULL CHECK (report_type IN ('excursion_summary', 'deviation_investigation', 'shipment_certificate', 'lane_performance', 'ccp_log', 'annual_review')),
    shipment_id UUID REFERENCES shipments(id),
    excursion_id UUID REFERENCES excursions(id),
    title VARCHAR(255) NOT NULL,
    format VARCHAR(10) NOT NULL DEFAULT 'pdf' CHECK (format IN ('pdf', 'csv', 'xlsx', 'json')),
    file_url VARCHAR(500),
    ai_generated BOOLEAN NOT NULL DEFAULT false,
    generated_by UUID REFERENCES users(id),
    signed_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_reports_org ON compliance_reports(org_id);
CREATE INDEX idx_compliance_reports_shipment ON compliance_reports(shipment_id);
```

## EPCIS 2.0 Interoperability

```sql
-- EPCIS events for supply chain interoperability
CREATE TABLE epcis_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(30) NOT NULL CHECK (event_type IN ('ObjectEvent', 'AggregationEvent', 'TransactionEvent', 'TransformationEvent', 'AssociationEvent')),
    event_time TIMESTAMPTZ NOT NULL,
    event_timezone_offset VARCHAR(10) NOT NULL,
    action VARCHAR(10) NOT NULL CHECK (action IN ('ADD', 'OBSERVE', 'DELETE')),
    biz_step VARCHAR(255),  -- GS1 CBV business step URI
    disposition VARCHAR(255),  -- GS1 CBV disposition URI
    read_point VARCHAR(255),  -- GS1 location URI
    biz_location VARCHAR(255),
    shipment_id UUID REFERENCES shipments(id),
    epc_list TEXT[],  -- EPCs involved
    hash_id VARCHAR(255),  -- EPCIS event hash for deduplication
    exported_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_epcis_events_shipment ON epcis_events(shipment_id);
CREATE INDEX idx_epcis_events_time ON epcis_events(event_time);

-- EPCIS sensor reports (mirrors sensorElementList)
CREATE TABLE epcis_sensor_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epcis_event_id UUID NOT NULL REFERENCES epcis_events(id),
    sensor_id UUID REFERENCES sensors(id),
    start_time TIMESTAMPTZ,
    end_time TIMESTAMPTZ,
    measurement_type VARCHAR(50) NOT NULL,  -- 'Temperature', 'RelativeHumidity', etc.
    value DECIMAL(10,4),
    min_value DECIMAL(10,4),
    max_value DECIMAL(10,4),
    mean_value DECIMAL(10,4),
    std_dev DECIMAL(10,4),
    uom VARCHAR(10) NOT NULL,  -- 'CEL', '%', 'g', 'lux'
    is_exception BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_epcis_sensor_reports_event ON epcis_sensor_reports(epcis_event_id);
```

## HACCP Critical Control Points

```sql
-- Critical control points
CREATE TABLE critical_control_points (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    name VARCHAR(255) NOT NULL,
    ccp_number VARCHAR(20) NOT NULL,
    hazard_description TEXT NOT NULL,
    critical_limit_min DECIMAL(6,2),
    critical_limit_max DECIMAL(6,2),
    monitoring_frequency_minutes INT NOT NULL DEFAULT 60,
    corrective_action TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ccp_org_location ON critical_control_points(org_id, location_id);

-- CCP temperature checks
CREATE TABLE ccp_checks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ccp_id UUID NOT NULL REFERENCES critical_control_points(id),
    sensor_id UUID REFERENCES sensors(id),
    temperature_celsius DECIMAL(6,2) NOT NULL,
    is_compliant BOOLEAN NOT NULL,
    corrective_action_taken TEXT,
    checked_by UUID REFERENCES users(id),
    checked_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ccp_checks_ccp ON ccp_checks(ccp_id);
CREATE INDEX idx_ccp_checks_time ON ccp_checks(checked_at);
```

## AI & Analytics

```sql
-- ML predictions
CREATE TABLE excursion_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    model_version VARCHAR(50) NOT NULL,
    predicted_excursion_type VARCHAR(30),
    probability DECIMAL(5,4) NOT NULL,  -- 0.0000 to 1.0000
    predicted_breach_at TIMESTAMPTZ,
    recommended_action TEXT,
    features_used TEXT[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_excursion_predictions_shipment ON excursion_predictions(shipment_id);

-- Lane risk scores (AI-computed)
CREATE TABLE lane_risk_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lane_id UUID NOT NULL REFERENCES lanes(id),
    score DECIMAL(5,2) NOT NULL,  -- 0-100
    excursion_rate DECIMAL(5,4),  -- historical excursion percentage
    avg_excursion_duration_minutes DECIMAL(8,2),
    sample_shipment_count INT NOT NULL,
    model_version VARCHAR(50),
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lane_risk_scores_lane ON lane_risk_scores(lane_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisations, users, jurisdictions |
| Sensor & Device Management | 2 | sensors, sensor_connections |
| Configuration & Thresholds | 2 | temperature_ranges, alert_profiles |
| Products & Locations | 2 | products, locations |
| Shipment Lifecycle | 4 | shipments, shipment_sensors, custody_events, lanes |
| Sensor Telemetry | 1 | sensor_readings |
| Excursion Management | 2 | excursions, deviation_investigations |
| Alerting | 2 | alert_rules, alert_notifications |
| Compliance & Audit | 3 | audit_trail, electronic_signatures, compliance_reports |
| EPCIS Interoperability | 2 | epcis_events, epcis_sensor_reports |
| HACCP | 2 | critical_control_points, ccp_checks |
| AI & Analytics | 2 | excursion_predictions, lane_risk_scores |
| **Total** | **27** | |

---

## Key Design Decisions

1. **Separate `sensor_readings` table from `excursions`**: Raw telemetry is high-volume append-only data; excursions are derived business events that require workflow tracking. Keeping them separate allows the readings table to be partitioned by time while excursions remain a traditional operational table.

2. **EPCIS tables mirror the GS1 standard structure**: Rather than converting EPCIS concepts into an internal model, we store them in their native structure (`epcis_events` + `epcis_sensor_reports`) so export to EPCIS 2.0 JSON-LD is a direct mapping.

3. **Audit trail as a separate table, not triggers**: While PostgreSQL triggers could auto-populate an audit trail, the 21 CFR Part 11 requirement for a `reason` field (why the change was made) means the application layer must provide context that triggers cannot capture. The audit trail table is designed to be append-only.

4. **Electronic signatures are independent records**: Per 21 CFR Part 11, electronic signatures must be linked to but not embedded in the records they sign. The `electronic_signatures` table references any table/record via `table_name` + `record_id`, allowing any entity to be signed.

5. **Lane analytics as a first-class entity**: The `lanes` table with `lane_risk_scores` supports the AI-driven lane/carrier risk scoring feature identified as a key differentiator. Lanes are explicitly modelled rather than derived from shipment origin/destination pairs.

6. **GS1 identifiers throughout**: `gln` (Global Location Number) on locations and organisations, `gtin` on products — enabling EPCIS interoperability without identifier translation.

7. **Temperature ranges as reference data**: GDP, WHO, and HACCP all define specific temperature ranges. Making these a reference table (rather than hardcoded values) allows the platform to support multiple regulatory frameworks simultaneously.

8. **Partitioning strategy for sensor_readings**: The `sensor_readings` table should be partitioned by `recorded_at` (monthly or weekly) using PostgreSQL native partitioning. This is noted in the index design but left to the DBA to configure based on expected volume.
