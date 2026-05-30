# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Cold Chain Monitoring · Created: 2026-05-20

## Philosophy

This model uses strongly-typed relational columns for core fields that are queried, indexed, and constrained, combined with JSONB columns for variable, jurisdiction-specific, or rapidly-evolving data. The key insight is that cold chain monitoring spans many regulatory regimes (FDA, EU GDP, WHO, HACCP, FSMA), multiple sensor types with different data shapes, and diverse product categories — each adding fields that do not apply universally.

Rather than creating a table for every variation (normalized approach) or storing everything as events (event-sourced approach), this hybrid reserves relational columns for the 80% of fields that are universal — shipment ID, temperature, timestamp, status — and pushes the variable 20% into JSONB columns. A `regulatory_data` JSONB column on a shipment can hold FDA-specific fields for US pharma shipments and EU GDP fields for European ones, without requiring separate tables or DDL migrations.

PostgreSQL's JSONB support — GIN indexing, containment operators, JSON path queries — makes this approach practical for production workloads. The pattern is widely used in multi-tenant SaaS platforms where customers have different configuration needs. It balances the integrity of relational design with the flexibility to accommodate a fragmented regulatory and sensor landscape.

**Best for:** Multi-vertical deployments serving both pharma (21 CFR Part 11) and food (HACCP/FSMA) customers, or platforms that need to iterate rapidly on features without frequent schema migrations.

**Trade-offs:**
- (+) Fewer tables than fully normalized (~20 vs ~27+); faster development velocity
- (+) New sensor types, regulatory fields, and product attributes added without DDL migrations
- (+) Core fields still have relational integrity (foreign keys, CHECK constraints, indexes)
- (+) JSONB GIN indexes enable efficient querying of variable fields
- (+) Natural fit for EPCIS 2.0 sensorElementList (already JSON-native)
- (-) JSONB fields lack database-level type checking; validation must happen in application layer
- (-) JSONB columns can become "junk drawers" without schema discipline
- (-) Complex JSONB queries can be slower than equivalent relational joins
- (-) Reporting tools may struggle with JSONB column extraction
- (-) 21 CFR Part 11 auditing of JSONB field changes requires deep JSON diffing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 / ISO IEC 19987:2024 | `sensor_data` JSONB column on readings stores the full sensorElementList/sensorReport structure natively; EPCIS export is near-zero transformation |
| FDA 21 CFR Part 11 | `regulatory_data` JSONB on shipments stores FDA-specific fields; audit trail captures JSON diffs for JSONB changes |
| EU GDP 2013/C 68/01 | GDP-specific deviation fields stored in `regulatory_data` JSONB; universal deviation workflow in relational columns |
| HACCP / FSMA | HACCP CCP configurations stored as JSONB arrays on locations; CCP checks have a `details` JSONB for variable inspection data |
| ISO 31512:2024 | Core shipment fields align with ISO 31512; jurisdiction-specific logistics requirements in JSONB |
| ISO 31510:2025 | Relational column names follow ISO 31510 vocabulary; JSONB keys use the same vocabulary |

---

## Core Tables

```sql
-- Organisations (tenants)
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(50) NOT NULL CHECK (org_type IN ('shipper', 'carrier', '3pl', 'warehouse', 'manufacturer', 'distributor')),
    parent_org_id UUID REFERENCES organisations(id),
    gln VARCHAR(13),
    country_code CHAR(2) NOT NULL,
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    compliance_frameworks TEXT[] NOT NULL DEFAULT '{}',
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_alert_channels": ["email", "sms"],
    --   "retention_days": 730,
    --   "sso_provider": "okta",
    --   "branding": {"logo_url": "...", "primary_color": "#003366"}
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisations_parent ON organisations(parent_org_id);

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
    preferences JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "temperature_unit": "CEL",
    --   "dashboard_layout": "map_first",
    --   "notification_quiet_hours": {"start": "22:00", "end": "07:00"}
    -- }
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(org_id);
```

## Sensors & Devices

```sql
CREATE TABLE sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    serial_number VARCHAR(100) NOT NULL,
    device_type VARCHAR(50) NOT NULL CHECK (device_type IN ('ble', 'cellular', 'lorawan', 'satellite', 'wired', 'passive_label')),
    status VARCHAR(30) NOT NULL DEFAULT 'available' CHECK (status IN ('available', 'deployed', 'in_transit', 'maintenance', 'retired')),
    capabilities TEXT[] NOT NULL DEFAULT '{}',
    device_metadata JSONB NOT NULL DEFAULT '{}',
    -- device_metadata example for cellular tracker:
    -- {
    --   "manufacturer": "Tive",
    --   "model": "Solo Pro",
    --   "firmware_version": "3.2.1",
    --   "accuracy_celsius": 0.5,
    --   "battery_capacity_mah": 3200,
    --   "connectivity": {"type": "cellular", "bands": ["LTE-M", "NB-IoT"]},
    --   "calibration": {
    --     "date": "2026-01-15",
    --     "due_date": "2027-01-15",
    --     "certificate_url": "https://..."
    --   }
    -- }
    --
    -- device_metadata example for LoRaWAN sensor:
    -- {
    --   "manufacturer": "Milesight",
    --   "model": "EM500-TEMP",
    --   "firmware_version": "1.4.0",
    --   "accuracy_celsius": 0.3,
    --   "lorawan": {
    --     "dev_eui": "0018B2000012ABCD",
    --     "app_key": "...",
    --     "frequency_plan": "EU868"
    --   }
    -- }
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- connection_config example:
    -- {
    --   "protocol": "mqtt",
    --   "topic": "sensors/0018B2000012ABCD/telemetry",
    --   "qos": 1,
    --   "reporting_interval_seconds": 300
    -- }
    last_seen_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, serial_number)
);

CREATE INDEX idx_sensors_org_status ON sensors(org_id, status);
CREATE INDEX idx_sensors_device_metadata ON sensors USING GIN (device_metadata);
```

## Locations

```sql
CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    location_type VARCHAR(50) NOT NULL CHECK (location_type IN ('warehouse', 'cold_storage', 'dock', 'checkpoint', 'pharmacy', 'hospital', 'retail', 'airport', 'port')),
    gln VARCHAR(13),
    country_code CHAR(2) NOT NULL,
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    address JSONB NOT NULL DEFAULT '{}',
    -- address example:
    -- {
    --   "line1": "123 Cold Storage Way",
    --   "city": "Chicago",
    --   "state": "IL",
    --   "postal_code": "60601",
    --   "country": "US"
    -- }
    haccp_config JSONB,
    -- haccp_config example (for food locations with CCPs):
    -- {
    --   "ccps": [
    --     {
    --       "ccp_number": "CCP-1",
    --       "name": "Receiving Dock Temperature Check",
    --       "hazard": "Pathogen growth above 4°C",
    --       "critical_limit_max_celsius": 4.0,
    --       "monitoring_frequency_minutes": 30,
    --       "corrective_action": "Reject shipment; notify supplier"
    --     },
    --     {
    --       "ccp_number": "CCP-2",
    --       "name": "Walk-in Cooler",
    --       "hazard": "Temperature excursion",
    --       "critical_limit_max_celsius": 4.0,
    --       "monitoring_frequency_minutes": 60,
    --       "corrective_action": "Move product to backup cooler; call maintenance"
    --     }
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_org ON locations(org_id);
CREATE INDEX idx_locations_gln ON locations(gln) WHERE gln IS NOT NULL;
```

## Products

```sql
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
    product_attributes JSONB NOT NULL DEFAULT '{}',
    -- product_attributes example for pharma:
    -- {
    --   "regulatory_class": "prescription_drug",
    --   "ndc": "12345-678-90",
    --   "stability_budget_minutes": {"2_8c": null, "8_15c": 480, "15_25c": 120, "above_25c": 30},
    --   "light_sensitive": true,
    --   "shock_sensitive": false,
    --   "storage_standard": "USP <1079>"
    -- }
    --
    -- product_attributes example for food:
    -- {
    --   "regulatory_class": "perishable_food",
    --   "upc": "012345678901",
    --   "shelf_life_model": "arrhenius",
    --   "q10_value": 2.5,
    --   "reference_temp_celsius": 4.0,
    --   "allergens": ["milk", "soy"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_org ON products(org_id);
CREATE INDEX idx_products_gtin ON products(gtin) WHERE gtin IS NOT NULL;
```

## Shipments

```sql
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
    -- Denormalised alert thresholds (snapshot at shipment creation)
    temp_low_warn DECIMAL(6,2),
    temp_low_critical DECIMAL(6,2),
    temp_high_warn DECIMAL(6,2),
    temp_high_critical DECIMAL(6,2),
    -- Timestamps
    expected_departure_at TIMESTAMPTZ,
    actual_departure_at TIMESTAMPTZ,
    expected_arrival_at TIMESTAMPTZ,
    actual_arrival_at TIMESTAMPTZ,
    -- Variable regulatory and logistics data
    regulatory_data JSONB NOT NULL DEFAULT '{}',
    -- regulatory_data example for FDA-regulated pharma shipment:
    -- {
    --   "framework": "fda_21cfr11",
    --   "qualified_person": "Dr. Jane Smith",
    --   "validation_protocol": "VP-2026-042",
    --   "stability_budget_remaining_minutes": 420,
    --   "gdp_deviation_required": true,
    --   "customs_declarations": [
    --     {"country": "US", "declaration_number": "CBP-2026-12345"}
    --   ]
    -- }
    --
    -- regulatory_data example for HACCP food shipment:
    -- {
    --   "framework": "haccp_fsma",
    --   "fda_traceability_lot": "TL-2026-5678",
    --   "receiving_inspection_required": true,
    --   "haccp_plan_ref": "HACCP-PLAN-007"
    -- }
    logistics_data JSONB NOT NULL DEFAULT '{}',
    -- logistics_data example:
    -- {
    --   "tracking_number": "1Z999AA10123456784",
    --   "container_id": "MSKU1234567",
    --   "vehicle_plate": "AB12 CDE",
    --   "packaging_type": "insulated_pallet",
    --   "coolant_type": "dry_ice",
    --   "coolant_kg": 5.0,
    --   "route_waypoints": [
    --     {"name": "Chicago Hub", "expected_at": "2026-05-20T10:00:00Z"},
    --     {"name": "Memphis Hub", "expected_at": "2026-05-20T18:00:00Z"}
    --   ]
    -- }
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, shipment_number)
);

CREATE INDEX idx_shipments_org_status ON shipments(org_id, status);
CREATE INDEX idx_shipments_dates ON shipments(expected_departure_at, expected_arrival_at);
CREATE INDEX idx_shipments_regulatory ON shipments USING GIN (regulatory_data);
```

## Sensor Assignments & Custody

```sql
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

-- Chain of custody and shipment lifecycle events
CREATE TABLE shipment_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    event_type VARCHAR(50) NOT NULL CHECK (event_type IN ('created', 'packed', 'departed', 'checkpoint_arrival', 'checkpoint_departure', 'delivered', 'quarantined', 'released', 'rejected', 'handover', 'note_added')),
    location_id UUID REFERENCES locations(id),
    from_org_id UUID REFERENCES organisations(id),
    to_org_id UUID REFERENCES organisations(id),
    performed_by UUID REFERENCES users(id),
    event_data JSONB NOT NULL DEFAULT '{}',
    -- event_data example for handover:
    -- {
    --   "notes": "Received in good condition",
    --   "photo_urls": ["https://..."],
    --   "signature_capture": true,
    --   "ambient_temp_celsius": 22.5
    -- }
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shipment_events_shipment ON shipment_events(shipment_id);
CREATE INDEX idx_shipment_events_time ON shipment_events(recorded_at);
```

## Sensor Readings

```sql
-- Sensor telemetry — relational core + JSONB for variable sensor data
CREATE TABLE sensor_readings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    shipment_id UUID REFERENCES shipments(id),
    -- Core relational fields (always present, always indexed)
    temperature DECIMAL(6,2),   -- CEL, the primary measurement
    recorded_at TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    latitude DECIMAL(10,7),
    longitude DECIMAL(10,7),
    is_excursion BOOLEAN NOT NULL DEFAULT false,
    -- Variable sensor data in JSONB
    sensor_data JSONB NOT NULL DEFAULT '{}',
    -- sensor_data example for multi-sensor tracker:
    -- {
    --   "humidity_pct": 45.2,
    --   "shock_g": 0.3,
    --   "light_lux": 0.0,
    --   "pressure_hpa": 1013.25,
    --   "door_status": "closed",
    --   "battery_pct": 87.5,
    --   "signal_strength_dbm": -72
    -- }
    --
    -- sensor_data example matching EPCIS 2.0 sensorReport:
    -- {
    --   "epcis_sensor_report": {
    --     "type": "Temperature",
    --     "value": 4.7,
    --     "minValue": 4.2,
    --     "maxValue": 5.1,
    --     "meanValue": 4.65,
    --     "uom": "CEL"
    --   }
    -- }
    epcis_event_id UUID  -- optional link to an exported EPCIS event
);

-- Time-based partitioning recommended: PARTITION BY RANGE (recorded_at)
CREATE INDEX idx_readings_sensor_time ON sensor_readings(sensor_id, recorded_at);
CREATE INDEX idx_readings_shipment_time ON sensor_readings(shipment_id, recorded_at);
CREATE INDEX idx_readings_excursion ON sensor_readings(shipment_id) WHERE is_excursion = true;
CREATE INDEX idx_readings_sensor_data ON sensor_readings USING GIN (sensor_data);
```

## Excursions

```sql
CREATE TABLE excursions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    sensor_id UUID NOT NULL REFERENCES sensors(id),
    excursion_type VARCHAR(30) NOT NULL CHECK (excursion_type IN ('high_temp', 'low_temp', 'high_humidity', 'low_humidity', 'shock', 'light_exposure', 'door_open')),
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('warning', 'critical')),
    status VARCHAR(30) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'resolved', 'acknowledged', 'investigating', 'closed_released', 'closed_rejected', 'closed_destroyed')),
    threshold_value DECIMAL(10,4) NOT NULL,
    peak_value DECIMAL(10,4),
    unit VARCHAR(10) NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    -- Investigation and disposition data (variable by framework)
    investigation JSONB NOT NULL DEFAULT '{}',
    -- investigation example:
    -- {
    --   "investigation_number": "DEV-2026-042",
    --   "root_cause": "Compressor failure during highway segment",
    --   "impact_assessment": "47 min exposure at max 12.3°C; within stability budget",
    --   "corrective_action": "Carrier scheduled compressor maintenance",
    --   "preventive_action": "Added predictive maintenance alerts",
    --   "product_disposition": "release",
    --   "investigator_id": "550e8400-...",
    --   "reviewer_id": "550e8400-...",
    --   "ai_generated_summary": "Based on 47-minute excursion peaking at 12.3°C...",
    --   "signed_off_at": "2026-05-20T18:00:00Z"
    -- }
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_excursions_shipment ON excursions(shipment_id);
CREATE INDEX idx_excursions_status ON excursions(status);
CREATE INDEX idx_excursions_investigation ON excursions USING GIN (investigation);
```

## Alerting

```sql
CREATE TABLE alert_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    rule_config JSONB NOT NULL,
    -- rule_config example:
    -- {
    --   "condition": "threshold_breach",
    --   "channels": ["email", "sms", "webhook"],
    --   "escalation_delay_minutes": 15,
    --   "recipients": [
    --     {"type": "email", "address": "quality@pharma.com"},
    --     {"type": "webhook", "url": "https://hooks.slack.com/..."}
    --   ],
    --   "filters": {
    --     "product_types": ["pharmaceutical", "vaccine"],
    --     "severity": ["critical"]
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_rules_org ON alert_rules(org_id);

CREATE TABLE alert_notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    excursion_id UUID REFERENCES excursions(id),
    alert_rule_id UUID REFERENCES alert_rules(id),
    channel VARCHAR(20) NOT NULL,
    recipient VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'delivered', 'failed')),
    sent_at TIMESTAMPTZ,
    delivery_details JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_notifications_excursion ON alert_notifications(excursion_id);
```

## Audit Trail & Compliance

```sql
-- Audit trail (supports 21 CFR Part 11)
CREATE TABLE audit_trail (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    changes JSONB NOT NULL,
    -- changes example for relational field:
    -- {"field": "status", "old": "in_transit", "new": "delivered"}
    --
    -- changes example for JSONB field diff:
    -- {"field": "regulatory_data.stability_budget_remaining_minutes", "old": 420, "new": 373}
    user_id UUID REFERENCES users(id),
    reason TEXT,
    ip_address INET,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_trail_record ON audit_trail(table_name, record_id);
CREATE INDEX idx_audit_trail_time ON audit_trail(recorded_at);

-- Electronic signatures
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
```

## AI & Analytics

```sql
-- ML excursion predictions
CREATE TABLE excursion_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    model_version VARCHAR(50) NOT NULL,
    probability DECIMAL(5,4) NOT NULL,
    predicted_breach_at TIMESTAMPTZ,
    prediction_details JSONB NOT NULL DEFAULT '{}',
    -- prediction_details example:
    -- {
    --   "predicted_type": "high_temp",
    --   "confidence_interval": [0.72, 0.89],
    --   "top_risk_factors": [
    --     {"factor": "ambient_forecast_35c", "weight": 0.35},
    --     {"factor": "carrier_excursion_history", "weight": 0.25},
    --     {"factor": "packaging_age_days", "weight": 0.20}
    --   ],
    --   "recommended_actions": ["reroute_via_cooled_hub", "add_gel_packs"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_predictions_shipment ON excursion_predictions(shipment_id);
```

---

## Example JSONB Queries

### Find all pharma shipments with stability budget under 60 minutes remaining

```sql
SELECT id, shipment_number, status,
       (regulatory_data->>'stability_budget_remaining_minutes')::int AS budget_minutes
FROM shipments
WHERE org_id = '...'
  AND regulatory_data->>'framework' = 'fda_21cfr11'
  AND (regulatory_data->>'stability_budget_remaining_minutes')::int < 60
  AND status = 'in_transit';
```

### Find sensors with specific LoRaWAN frequency plan

```sql
SELECT id, serial_number, device_metadata->'lorawan'->>'frequency_plan' AS freq_plan
FROM sensors
WHERE device_metadata @> '{"lorawan": {"frequency_plan": "EU868"}}';
```

### Get all humidity readings above 70% from sensor_data JSONB

```sql
SELECT id, sensor_id, temperature, recorded_at,
       (sensor_data->>'humidity_pct')::decimal AS humidity
FROM sensor_readings
WHERE shipment_id = '...'
  AND (sensor_data->>'humidity_pct')::decimal > 70.0
ORDER BY recorded_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | organisations, users |
| Sensors & Devices | 1 | sensors (device_metadata + connection_config in JSONB) |
| Locations & Products | 2 | locations (haccp_config in JSONB), products (product_attributes in JSONB) |
| Shipments & Lifecycle | 3 | shipments, shipment_sensors, shipment_events |
| Sensor Telemetry | 1 | sensor_readings (sensor_data JSONB for variable measurements) |
| Excursions | 1 | excursions (investigation JSONB for variable workflow data) |
| Alerting | 2 | alert_rules (rule_config JSONB), alert_notifications |
| Audit & Compliance | 2 | audit_trail (changes JSONB), electronic_signatures |
| AI & Analytics | 1 | excursion_predictions (prediction_details JSONB) |
| **Total** | **15** | JSONB columns replace ~10 tables from the normalized model |

---

## Key Design Decisions

1. **Temperature as a first-class relational column on sensor_readings**: Despite using JSONB for variable sensor data, temperature gets its own indexed `DECIMAL` column because it is the most queried field in the entire system. Every dashboard, alert rule, and compliance report filters on temperature. Pushing it into JSONB would trade query performance for consistency.

2. **Regulatory framework variation handled by JSONB, not table inheritance**: Rather than `pharma_shipments`, `food_shipments`, and `vaccine_shipments` tables (or a complex single-table inheritance scheme), the `regulatory_data` JSONB column holds framework-specific fields. Application-layer JSON Schema validation ensures structure.

3. **HACCP configuration as JSONB on locations**: HACCP critical control points are specific to food-industry locations. Rather than a separate `critical_control_points` table that is empty for pharma customers, CCPs are stored as a JSONB array on locations that have them.

4. **Investigation workflow embedded in excursions**: The normalized model has a separate `deviation_investigations` table. Here, the investigation lifecycle is stored as a JSONB column on the `excursion` record, reducing joins for the most common query pattern: "show me this excursion with its investigation."

5. **Device metadata as JSONB for hardware diversity**: BLE sensors, cellular trackers, LoRaWAN nodes, passive labels, and wired probes all have different configuration fields. The `device_metadata` JSONB column accommodates this variety without a table-per-device-type strategy.

6. **EPCIS sensorReport stored natively in JSONB**: When sensor readings include EPCIS-formatted data, it is stored in the `sensor_data` JSONB field in its native JSON-LD structure. EPCIS 2.0 export reads directly from this field without transformation.

7. **Audit trail uses JSONB for change tracking**: Rather than `old_value`/`new_value` text columns (which cannot represent JSONB field changes), the audit trail stores a `changes` JSONB object that can capture both relational and JSONB field modifications with full path specificity.

8. **GIN indexes on key JSONB columns**: `regulatory_data`, `sensor_data`, `device_metadata`, and `investigation` all have GIN indexes enabling containment queries (`@>`) for efficient filtering on variable fields.
