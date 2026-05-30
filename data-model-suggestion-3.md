# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Water Resource Management · Created: 2026-05-25

## Philosophy

This model uses a reduced set of relational tables for core entities and relationships, but pushes jurisdiction-specific, sensor-vendor-specific, and configuration-variant data into JSONB columns. The key insight is that water resource management spans many jurisdictions (California prior appropriation differs from Colorado's, which differs from Australian riparian systems), many sensor vendors (each with proprietary payload formats), and many organisation types (farms vs. districts vs. utilities). A fully normalised schema would either need hundreds of columns with most being NULL, or dozens of junction tables for each variant — both of which hurt developer productivity and query performance.

The hybrid approach keeps the "what is always true" in typed, indexed relational columns and the "what varies by context" in JSONB. PostgreSQL's JSONB supports GIN indexes, containment queries (@>), path expressions (->>, #>>), and partial indexing, making the flexible fields queryable without sacrificing performance. This pattern is used by Stripe (payment metadata), Shopify (product metafields), and most modern multi-tenant SaaS platforms.

For a water resource management MVP, this approach lets the team ship fast: a new sensor vendor is supported by adding a decoder that writes its specific fields into the JSONB column, not by running a migration. A new jurisdiction's water rights fields are modelled in JSONB with a JSON Schema validator, not by adding nullable columns. The trade-off is that the JSONB fields require discipline — without schemas and documentation, they can become an unstructured mess.

**Best for:** Rapid MVP development, multi-jurisdiction deployments where regulatory fields vary, and teams that need to integrate diverse sensor vendors without per-vendor schema migrations.

**Trade-offs:**
- (+) Fewer tables (roughly half the normalised model) — simpler ORM mapping and fewer migrations
- (+) New jurisdictions, sensor types, and organisation variants require code changes, not schema changes
- (+) JSONB columns handle the long tail of vendor-specific and jurisdiction-specific fields
- (+) Fast to prototype and iterate on feature scope
- (+) PostgreSQL JSONB operators and GIN indexes provide good query performance on flexible fields
- (-) JSONB fields lack database-level type enforcement — validation must happen in application code or JSON Schema
- (-) JSONB data is harder to report on with standard BI tools
- (-) Risk of schema drift: different records can have different JSONB shapes if validation is lax
- (-) Partial indexes on JSONB paths are more complex to maintain than column indexes
- (-) Joins involving JSONB fields are slower than joins on typed columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OGC SensorThings API | Core entities (sensor, datastream) are relational; vendor-specific metadata and payload variations go into JSONB |
| OGC WaterML 2.0 | Observation core fields are relational columns; extended quality metadata and interpolation types go into JSONB |
| HarDWR v1 | Common water rights fields (priority_date, use_category, max_flow, max_volume) are relational; state-specific fields go into a `jurisdiction_data` JSONB column |
| FAO-56 Penman-Monteith | Standard ET parameters are relational columns; alternative methods and extended parameters go into JSONB |
| ISO 3166 | Jurisdiction codes are a relational column; jurisdiction-specific legal requirements are JSONB |
| LoRaWAN / Cayenne LPP | Raw sensor payloads from different vendors stored in `raw_payload` JSONB for debugging and reprocessing |
| GeoJSON / RFC 7946 | Spatial data uses PostGIS columns (not JSONB) for proper spatial indexing |

---

## Organisation & Auth

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN ('farm', 'irrigation_district', 'utility', 'regulator', 'enterprise')),
    parent_org_id   UUID REFERENCES organisations(id),
    jurisdiction_code VARCHAR(10),         -- ISO 3166
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example for irrigation_district:
    -- {
    --   "billing_enabled": true,
    --   "water_ordering_enabled": true,
    --   "rate_type": "tiered",
    --   "fiscal_year_start_month": 10,
    --   "regulatory_agency": "SWRCB",
    --   "sgma_gsp_id": "GSP-2022-0042"
    -- }
    -- config example for farm:
    -- {
    --   "default_crop_model": "fao56",
    --   "sensor_vendors": ["cropx", "aquaspy"],
    --   "irrigation_method": "drip",
    --   "ndvi_provider": "openet"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_orgs_type ON organisations(org_type);
CREATE INDEX idx_orgs_parent ON organisations(parent_org_id);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    organisation_id UUID REFERENCES organisations(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    -- role: 'admin', 'manager', 'field_operator', 'ditch_rider', 'regulator', 'viewer'
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "units": "imperial",
    --   "language": "es",
    --   "notification_channels": ["sms", "email"],
    --   "dashboard_layout": "compact"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_org ON users(organisation_id);
```

---

## Fields & Infrastructure

```sql
CREATE TABLE fields (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,
    area_hectares   NUMERIC(10,4),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "soil_type": "clay_loam",
    --   "soil_texture": {"sand_pct": 30, "silt_pct": 35, "clay_pct": 35},
    --   "elevation_m": 125.0,
    --   "slope_pct": 2.5,
    --   "available_water_capacity_mm_m": 150,
    --   "field_capacity_vwc": 0.35,
    --   "wilting_point_vwc": 0.15,
    --   "management_allowed_depletion": 0.50
    -- }
    current_crop    JSONB,
    -- current_crop example:
    -- {
    --   "crop_name": "Almonds",
    --   "variety": "Nonpareil",
    --   "planting_date": "2020-02-15",
    --   "growth_stage": "mid_season",
    --   "kc_current": 1.05,
    --   "kc_ini": 0.40, "kc_mid": 1.05, "kc_end": 0.65,
    --   "root_depth_m": 1.5,
    --   "season_year": 2025
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fields_org ON fields(organisation_id);
CREATE INDEX idx_fields_boundary ON fields USING GIST(boundary);
CREATE INDEX idx_fields_crop ON fields USING GIN(current_crop);

CREATE TABLE irrigation_zones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    zone_type       VARCHAR(50) NOT NULL,  -- 'drip', 'sprinkler', 'flood', 'pivot', 'micro_sprinkler'
    boundary        GEOMETRY(Polygon, 4326),
    area_hectares   NUMERIC(10,4),
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "emitter_flow_lph": 3.5,
    --   "emitter_spacing_m": 0.60,
    --   "row_spacing_m": 5.0,
    --   "system_efficiency": 0.90,
    --   "max_application_rate_mmh": 6.5,
    --   "controller_id": "RC-2025-042",
    --   "valve_ids": ["V-01", "V-02", "V-03"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_zones_field ON irrigation_zones(field_id);

CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    asset_type      VARCHAR(50) NOT NULL,  -- 'pipe', 'pump', 'valve', 'canal', 'reservoir', 'well', 'meter', 'gate'
    name            VARCHAR(255) NOT NULL,
    location        GEOMETRY(Point, 4326),
    path            GEOMETRY(LineString, 4326),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by asset_type:
    -- pump: {"capacity_lps": 50, "motor_kw": 15, "fuel_type": "electric", "install_date": "2018-05-20"}
    -- pipe: {"diameter_mm": 200, "material": "HDPE", "length_m": 450, "pressure_rating_bar": 10}
    -- well: {"depth_m": 85, "casing_diameter_mm": 300, "static_water_level_m": 25, "pump_depth_m": 60}
    -- canal: {"width_m": 3.0, "depth_m": 1.5, "lined": true, "capacity_cfs": 15}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assets_org ON assets(organisation_id);
CREATE INDEX idx_assets_type ON assets(asset_type);
CREATE INDEX idx_assets_location ON assets USING GIST(location);
```

---

## Sensors & Observations

```sql
CREATE TABLE sensors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    device_type     VARCHAR(50) NOT NULL,
    vendor          VARCHAR(100),
    model           VARCHAR(100),
    serial_number   VARCHAR(255),
    connectivity    VARCHAR(50),
    location        GEOMETRY(Point, 4326),
    field_id        UUID REFERENCES fields(id),
    zone_id         UUID REFERENCES irrigation_zones(id),
    asset_id        UUID REFERENCES assets(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    battery_pct     NUMERIC(5,2),
    last_seen_at    TIMESTAMPTZ,
    vendor_config   JSONB NOT NULL DEFAULT '{}',
    -- vendor_config examples:
    -- CropX: {"device_eui": "A8404117C1829B45", "app_key": "...", "depths_cm": [15, 30, 60], "firmware": "3.2.1"}
    -- AquaSpy: {"probe_id": "AS-2025-442", "segments": 12, "segment_spacing_cm": 10}
    -- Davis: {"station_id": "DVS-001", "transmitter_id": 1, "retransmit": false}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensors_org ON sensors(organisation_id);
CREATE INDEX idx_sensors_field ON sensors(field_id);
CREATE INDEX idx_sensors_type ON sensors(device_type);

CREATE TABLE observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensors(id) ON DELETE CASCADE,
    observed_at     TIMESTAMPTZ NOT NULL,
    property        VARCHAR(100) NOT NULL,  -- 'soil_moisture_vwc', 'soil_temp_c', 'ec_dsm', etc.
    value           NUMERIC(15,6) NOT NULL,
    unit            VARCHAR(50) NOT NULL,
    quality         VARCHAR(20) DEFAULT 'good',
    depth_m         NUMERIC(4,2),           -- for multi-depth soil sensors
    raw_payload     JSONB,                  -- original vendor payload for debugging/reprocessing
    -- raw_payload example (LoRaWAN Cayenne LPP decoded):
    -- {
    --   "rx_time": "2025-07-15T14:30:00Z",
    --   "port": 2,
    --   "rssi": -95,
    --   "snr": 7.5,
    --   "decoded": {"channel_1": {"type": "analog_input", "value": 0.22}}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_obs_sensor_time ON observations(sensor_id, observed_at DESC);
CREATE INDEX idx_obs_property_time ON observations(property, observed_at DESC);
CREATE INDEX idx_obs_time ON observations(observed_at DESC);

-- Partition by month for performance
-- ALTER TABLE observations PARTITION BY RANGE (observed_at);
```

---

## Weather & ET

```sql
CREATE TABLE weather_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          VARCHAR(100) NOT NULL,  -- 'on_site_station', 'noaa', 'openweather', 'openet'
    source_id       VARCHAR(255),           -- station ID or API reference
    location        GEOMETRY(Point, 4326),
    observed_at     TIMESTAMPTZ NOT NULL,
    -- FAO-56 core parameters as typed columns:
    temp_max_c      NUMERIC(5,2),
    temp_min_c      NUMERIC(5,2),
    humidity_pct    NUMERIC(5,2),
    wind_speed_ms   NUMERIC(6,2),
    solar_rad_mjm2  NUMERIC(8,4),
    precipitation_mm NUMERIC(8,2),
    -- Extended data in JSONB for non-FAO-56 parameters:
    extended        JSONB DEFAULT '{}',
    -- extended examples:
    -- {
    --   "dew_point_c": 12.5,
    --   "pressure_hpa": 1013.2,
    --   "uv_index": 8,
    --   "wind_direction_deg": 225,
    --   "wind_gust_ms": 5.2,
    --   "cloud_cover_pct": 15,
    --   "visibility_m": 16000,
    --   "forecast_confidence": 0.85
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_weather_source_time ON weather_data(source, observed_at DESC);
CREATE INDEX idx_weather_location ON weather_data USING GIST(location);

CREATE TABLE et_calculations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id),
    calc_date       DATE NOT NULL,
    et_reference_mm NUMERIC(8,4),
    crop_coefficient NUMERIC(4,2),
    et_crop_mm      NUMERIC(8,4),
    method          VARCHAR(50) NOT NULL DEFAULT 'fao56_penman_monteith',
    inputs          JSONB NOT NULL,
    -- inputs example (preserves all calculation inputs for reproducibility):
    -- {
    --   "weather_data_id": "uuid",
    --   "temp_max_c": 38.2, "temp_min_c": 18.5,
    --   "humidity_pct": 35.0, "wind_speed_ms": 2.1,
    --   "solar_rad_mjm2": 28.5,
    --   "elevation_m": 125.0, "latitude_deg": 38.5,
    --   "crop": "Almonds", "growth_stage": "mid_season",
    --   "satellite_source": "openet_ensemble",
    --   "ndvi": 0.72,
    --   "soil_moisture_deficit_mm": 12.5
    -- }
    confidence      NUMERIC(4,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_et_field_date ON et_calculations(field_id, calc_date DESC);
```

---

## Water Rights & Allocations

```sql
CREATE TABLE water_rights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    right_number    VARCHAR(100) NOT NULL,
    right_type      VARCHAR(50) NOT NULL,   -- HarDWR: 'appropriative', 'riparian', 'groundwater', etc.
    priority_date   DATE,                   -- HarDWR: critical for prior appropriation
    use_category    VARCHAR(50) NOT NULL,   -- HarDWR: 'irrigation', 'municipal', 'industrial', etc.
    water_source    VARCHAR(255),           -- HarDWR: waterbody or well name
    source_type     VARCHAR(50),            -- 'surface', 'groundwater', 'recycled', 'mixed'
    max_flow_cfs    NUMERIC(12,4),
    max_volume_af   NUMERIC(12,4),
    point_of_diversion GEOMETRY(Point, 4326),
    place_of_use    GEOMETRY(MultiPolygon, 4326),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    -- jurisdiction_data varies by state/country:
    -- California: {
    --   "board": "SWRCB",
    --   "permit_type": "appropriative_post1914",
    --   "application_number": "A031234",
    --   "face_value_af": 200,
    --   "reported_diversions_required": true,
    --   "sgma_basin": "5-022.01",
    --   "gsp_compliance": "adequate"
    -- }
    -- Colorado: {
    --   "division": 1,
    --   "water_district": 6,
    --   "decree_case_number": "W-7890",
    --   "adjudication_date": "1965-03-15",
    --   "augmentation_plan": "AP-2020-123"
    -- }
    -- Australia: {
    --   "entitlement_type": "high_reliability",
    --   "trading_zone": "Goulburn_1A",
    --   "carryover_pct": 50,
    --   "seasonal_determination_pct": 100
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rights_org ON water_rights(organisation_id);
CREATE INDEX idx_rights_priority ON water_rights(priority_date);
CREATE INDEX idx_rights_pod ON water_rights USING GIST(point_of_diversion);
CREATE INDEX idx_rights_jurisdiction ON water_rights USING GIN(jurisdiction_data);

CREATE TABLE water_usage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    water_right_id  UUID NOT NULL REFERENCES water_rights(id),
    field_id        UUID REFERENCES fields(id),
    meter_sensor_id UUID REFERENCES sensors(id),
    usage_date      DATE NOT NULL,
    volume_af       NUMERIC(12,6) NOT NULL,
    flow_rate_cfs   NUMERIC(10,4),
    reading_type    VARCHAR(50) NOT NULL DEFAULT 'metered',
    details         JSONB DEFAULT '{}',
    -- details example:
    -- {
    --   "meter_reading_start": 45231.5,
    --   "meter_reading_end": 45232.1,
    --   "delivery_point": "Turnout 14A",
    --   "water_order_id": "uuid-order"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_usage_right_date ON water_usage(water_right_id, usage_date);
CREATE INDEX idx_usage_field ON water_usage(field_id);
```

---

## Irrigation & Water Orders

```sql
CREATE TABLE irrigation_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id),
    zone_id         UUID REFERENCES irrigation_zones(id),
    event_type      VARCHAR(50) NOT NULL,  -- 'scheduled', 'started', 'completed', 'cancelled'
    schedule_type   VARCHAR(50),           -- 'ai_recommendation', 'manual', 'rule_based'
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    volume_mm       NUMERIC(8,2),
    duration_minutes INTEGER,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details for AI recommendation:
    -- {
    --   "ai_confidence": 0.87,
    --   "reasoning": "Soil moisture at 0.15 VWC, below threshold of 0.20...",
    --   "model_version": "irrigation_advisor_v3",
    --   "inputs": {
    --     "soil_moisture_vwc": 0.15,
    --     "et_forecast_mm": 6.2,
    --     "rain_forecast_mm": 0.0,
    --     "crop_coefficient": 1.05,
    --     "growth_stage": "mid_season"
    --   }
    -- }
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_irrigation_field ON irrigation_events(field_id);
CREATE INDEX idx_irrigation_type ON irrigation_events(event_type);
CREATE INDEX idx_irrigation_scheduled ON irrigation_events(scheduled_at);

CREATE TABLE water_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    order_number    VARCHAR(50) NOT NULL,
    ordered_by      UUID NOT NULL REFERENCES users(id),
    delivery_date   DATE NOT NULL,
    volume_af       NUMERIC(12,4),
    flow_rate_cfs   NUMERIC(10,4),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    assigned_rider  UUID REFERENCES users(id),
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "delivery_point": "Lateral 12, Turnout 4",
    --   "delivery_window": {"start": "06:00", "end": "18:00"},
    --   "priority": "normal",
    --   "rider_notes": "Gate needs repair, use temporary",
    --   "billing_account_id": "uuid-account"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_org ON water_orders(organisation_id);
CREATE INDEX idx_orders_status ON water_orders(status);
CREATE INDEX idx_orders_date ON water_orders(delivery_date);
```

---

## Billing

```sql
CREATE TABLE billing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    record_type     VARCHAR(50) NOT NULL,  -- 'account', 'invoice', 'payment', 'credit'
    account_number  VARCHAR(100),
    reference_number VARCHAR(100),
    amount          NUMERIC(12,2),
    status          VARCHAR(50),
    billing_period  DATERANGE,
    data            JSONB NOT NULL,
    -- data for 'account':
    -- {
    --   "holder_name": "Smith Ranch LLC",
    --   "contact_email": "billing@smithranch.com",
    --   "address": "1234 Farm Rd, Fresno, CA 93701",
    --   "rate_schedule": {"type": "tiered", "tiers": [...]},
    --   "water_right_ids": ["uuid-1", "uuid-2"]
    -- }
    -- data for 'invoice':
    -- {
    --   "account_id": "uuid-account",
    --   "line_items": [
    --     {"description": "Water usage Jul 2025", "volume_af": 12.5, "rate": 25.00, "amount": 312.50}
    --   ],
    --   "subtotal": 312.50,
    --   "tax": 0,
    --   "total": 312.50,
    --   "due_date": "2025-08-15"
    -- }
    -- data for 'payment':
    -- {
    --   "invoice_id": "uuid-invoice",
    --   "payment_method": "check",
    --   "check_number": "4521",
    --   "paid_at": "2025-08-10T00:00:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_billing_org ON billing(organisation_id);
CREATE INDEX idx_billing_type ON billing(record_type);
CREATE INDEX idx_billing_account ON billing(account_number);
CREATE INDEX idx_billing_status ON billing(status);
```

---

## Alerts & Compliance

```sql
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    message         TEXT,
    source_entity   VARCHAR(50),           -- 'sensor', 'water_right', 'field', 'asset'
    source_id       UUID,
    field_id        UUID REFERENCES fields(id),
    context         JSONB DEFAULT '{}',
    -- context examples:
    -- soil_threshold: {"sensor_id": "uuid", "property": "soil_moisture_vwc", "threshold": 0.15, "actual": 0.12}
    -- compliance:     {"water_right_id": "uuid", "allocated_af": 120, "used_af": 118, "pct_used": 98.3}
    -- sensor_offline: {"sensor_id": "uuid", "last_seen": "2025-07-14T08:00:00Z", "offline_hours": 30}
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alerts_org ON alerts(organisation_id);
CREATE INDEX idx_alerts_unresolved ON alerts(organisation_id) WHERE resolved_at IS NULL;
CREATE INDEX idx_alerts_type ON alerts(alert_type);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    -- changes example:
    -- {
    --   "before": {"status": "scheduled", "volume_mm": 20},
    --   "after":  {"status": "completed", "volume_mm": 22.5}
    -- }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);
```

---

## Example Queries

### Query water rights with jurisdiction-specific fields

```sql
-- Find all California water rights with SGMA basin compliance issues
SELECT
    wr.right_number,
    wr.max_volume_af,
    wr.jurisdiction_data->>'sgma_basin' AS sgma_basin,
    wr.jurisdiction_data->>'gsp_compliance' AS gsp_compliance,
    wr.jurisdiction_data->>'permit_type' AS permit_type
FROM water_rights wr
WHERE wr.organisation_id = 'uuid-org'
  AND wr.jurisdiction_data @> '{"gsp_compliance": "inadequate"}'::jsonb;
```

### Query sensor data with vendor-specific config

```sql
-- Find all CropX sensors with firmware below 3.0
SELECT
    s.serial_number,
    s.vendor_config->>'firmware' AS firmware,
    s.vendor_config->'depths_cm' AS depths
FROM sensors s
WHERE s.vendor = 'CropX'
  AND (s.vendor_config->>'firmware')::numeric < 3.0;
```

### Dashboard query combining relational and JSONB fields

```sql
-- Field status dashboard
SELECT
    f.name,
    f.area_hectares,
    f.current_crop->>'crop_name' AS crop,
    f.current_crop->>'growth_stage' AS stage,
    f.properties->>'soil_type' AS soil,
    latest_obs.soil_moisture,
    latest_et.et_crop_mm
FROM fields f
LEFT JOIN LATERAL (
    SELECT o.value AS soil_moisture
    FROM observations o
    WHERE o.sensor_id IN (SELECT id FROM sensors WHERE field_id = f.id)
      AND o.property = 'soil_moisture_vwc'
    ORDER BY o.observed_at DESC LIMIT 1
) latest_obs ON true
LEFT JOIN LATERAL (
    SELECT et.et_crop_mm
    FROM et_calculations et
    WHERE et.field_id = f.id
    ORDER BY et.calc_date DESC LIMIT 1
) latest_et ON true
WHERE f.organisation_id = 'uuid-org';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Auth | 2 | organisations, users |
| Fields & Infrastructure | 3 | fields, irrigation_zones, assets |
| Sensors & Observations | 2 | sensors, observations |
| Weather & ET | 2 | weather_data, et_calculations |
| Water Rights | 2 | water_rights, water_usage |
| Irrigation Operations | 2 | irrigation_events, water_orders |
| Billing | 1 | billing (multi-type via record_type + JSONB data) |
| Alerts | 1 | alerts |
| Audit | 1 | audit_log |
| **Total** | **16** | Roughly half the normalised model's table count |

---

## Key Design Decisions

1. **Typed columns for always-present fields, JSONB for variant fields** — every table has a clear boundary: priority_date, max_volume_af, and use_category are always columns because they exist for every water right. State-specific fields like sgma_basin or augmentation_plan go into jurisdiction_data JSONB because they only apply to specific jurisdictions.

2. **jurisdiction_data JSONB on water_rights** — this is the most impactful design choice. Water rights vary dramatically between California (SWRCB permits, SGMA compliance), Colorado (divisions, decrees, augmentation plans), and Australia (entitlement types, trading zones, seasonal determinations). Rather than NULL-heavy columns or per-jurisdiction tables, JSONB with GIN indexes handles all variants.

3. **vendor_config JSONB on sensors** — each sensor vendor has unique configuration fields (CropX depths and firmware versions, AquaSpy probe segments, Davis transmitter IDs). The vendor_config JSONB avoids per-vendor tables while keeping the core sensor identity in typed columns.

4. **raw_payload on observations** — storing the original LoRaWAN/vendor payload alongside the normalised value enables reprocessing when decoder bugs are found or new properties need to be extracted from historical data.

5. **Unified billing table** — billing accounts, invoices, payments, and credits share one table differentiated by record_type. This is a deliberate choice for MVP velocity: billing is complex to model relationally, and the JSONB data column handles the structural differences between record types. This should be refactored to separate tables if billing becomes a primary feature.

6. **current_crop as JSONB on fields** — rather than a separate crops reference table with a junction table for field-crop-season associations, the current crop state is embedded directly on the field. This simplifies the most common query (dashboard showing fields with current crops) at the cost of making historical crop rotations harder to query.

7. **PostGIS geometry columns are NOT JSONB** — despite the hybrid approach, spatial data always uses native PostGIS types with GIST indexes. GeoJSON in JSONB would lose spatial indexing and query capabilities.

8. **GIN indexes on key JSONB columns** — jurisdiction_data, vendor_config, and current_crop all have GIN indexes to support containment queries (@>) without full table scans.

9. **Observations table stays lean** — the observation table uses typed columns for the core fields (sensor_id, observed_at, property, value, unit, quality, depth_m) and only uses JSONB for the raw vendor payload. This keeps the highest-volume table as queryable and compressible as possible.

10. **JSONB fields require application-level JSON Schema validation** — the database does not enforce the structure of JSONB columns. The application layer must validate against documented JSON Schemas for each context (jurisdiction type, sensor vendor, billing record type) to prevent schema drift.
