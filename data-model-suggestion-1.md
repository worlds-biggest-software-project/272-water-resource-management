# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Water Resource Management · Created: 2026-05-25

## Philosophy

This model follows classical normalized relational design where every domain concept gets its own table, relationships are expressed through foreign keys and junction tables, and referential integrity is enforced at the database level. The schema is structured around the core entities that a water resource management platform must track: organisations (farms, districts, utilities), physical infrastructure (fields, zones, sensors, pipelines), water rights and allocations, observations and measurements, irrigation schedules, and billing.

The normalized approach draws from the HarDWR (Harmonized Database of Western U.S. Water Rights) data model for water rights fields, the OGC SensorThings API entity model for sensor/observation structures, and the FAO-56 Penman-Monteith parameter set for evapotranspiration calculation inputs. Each of these standards maps cleanly to relational tables with well-defined columns and constraints.

This is the safest starting point for a platform that must serve regulatory environments (SGMA compliance, water rights adjudication) where data integrity, auditability, and query flexibility are more important than write throughput. Every relationship is explicit, every constraint is enforced, and ad-hoc reporting across any dimension is straightforward SQL.

**Best for:** Teams prioritising data integrity, regulatory compliance, and flexible cross-entity reporting in a well-understood domain.

**Trade-offs:**
- (+) Maximum referential integrity — broken relationships are impossible
- (+) Standard SQL tooling, ORMs, and reporting tools work out of the box
- (+) Easy to add indexes and optimise for specific query patterns
- (+) Schema is self-documenting — the DDL is the specification
- (-) High table count increases migration complexity and ORM boilerplate
- (-) Schema changes require migrations for every new field or entity
- (-) Sensor observation tables will grow very large and need partitioning strategies
- (-) Many JOIN operations for common queries spanning organisations, fields, sensors, and observations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OGC SensorThings API | Entity model (Thing, Sensor, Datastream, Observation, ObservedProperty, FeatureOfInterest, Location) directly maps to relational tables |
| OGC WaterML 2.0 | Time series structure informs the `observations` and `time_series` table design — result time, phenomenon time, quality codes |
| HarDWR v1 | Water rights fields (priority_date, use_category, water_management_area, water_source, flow_allocation) map to `water_rights` table |
| FAO-56 Penman-Monteith | ET calculation parameters (temperature, humidity, wind speed, solar radiation, crop coefficient) inform `weather_observations` and `crop_coefficients` tables |
| ISO 14046 | Water footprint assessment fields inform `water_usage_records` and `water_footprint` tables |
| ISO 3166 | Country/subdivision codes for jurisdiction modelling in `jurisdictions` table |
| GeoJSON / RFC 7946 | Spatial data stored as PostGIS geometry columns, exportable as GeoJSON |
| OAuth 2.0 / OIDC | Authentication model informs `users`, `organisations`, `roles` tables |

---

## Organisation & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN ('farm', 'irrigation_district', 'utility', 'regulator', 'enterprise')),
    parent_org_id   UUID REFERENCES organisations(id),
    jurisdiction_id UUID REFERENCES jurisdictions(id),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisations_type ON organisations(org_type);
CREATE INDEX idx_organisations_parent ON organisations(parent_org_id);

CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    iso_3166_code   VARCHAR(10),           -- ISO 3166-1 alpha-2 or 3166-2 subdivision
    jurisdiction_type VARCHAR(50) NOT NULL CHECK (jurisdiction_type IN ('country', 'state', 'county', 'water_district', 'basin')),
    parent_id       UUID REFERENCES jurisdictions(id),
    boundary        GEOMETRY(MultiPolygon, 4326),  -- PostGIS
    legal_framework VARCHAR(100),          -- e.g. 'prior_appropriation', 'riparian_rights'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jurisdictions_type ON jurisdictions(jurisdiction_type);
CREATE INDEX idx_jurisdictions_boundary ON jurisdictions USING GIST(boundary);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',  -- 'local', 'oidc', 'saml'
    auth_subject    VARCHAR(255),          -- external IdP subject identifier
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,  -- 'org_admin', 'field_operator', 'ditch_rider', 'regulator', 'viewer'
    description     TEXT,
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_org_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    UNIQUE(user_id, organisation_id, role_id)
);
CREATE INDEX idx_user_org_roles_user ON user_org_roles(user_id);
CREATE INDEX idx_user_org_roles_org ON user_org_roles(organisation_id);
```

---

## Fields, Zones & Infrastructure

```sql
CREATE TABLE fields (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,  -- PostGIS field boundary
    area_hectares   NUMERIC(10,4),
    soil_type       VARCHAR(100),
    elevation_m     NUMERIC(8,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fields_org ON fields(organisation_id);
CREATE INDEX idx_fields_boundary ON fields USING GIST(boundary);

CREATE TABLE irrigation_zones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    zone_type       VARCHAR(50) NOT NULL CHECK (zone_type IN ('drip', 'sprinkler', 'flood', 'pivot', 'micro_sprinkler')),
    boundary        GEOMETRY(Polygon, 4326),
    area_hectares   NUMERIC(10,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_zones_field ON irrigation_zones(field_id);

CREATE TABLE crops (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    variety         VARCHAR(255),
    kc_ini          NUMERIC(4,2),  -- FAO-56 crop coefficient: initial stage
    kc_mid          NUMERIC(4,2),  -- FAO-56 crop coefficient: mid-season
    kc_end          NUMERIC(4,2),  -- FAO-56 crop coefficient: end of late season
    root_depth_m    NUMERIC(4,2),
    growing_days    INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE field_crop_seasons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id) ON DELETE CASCADE,
    zone_id         UUID REFERENCES irrigation_zones(id),
    crop_id         UUID NOT NULL REFERENCES crops(id),
    planting_date   DATE NOT NULL,
    harvest_date    DATE,
    growth_stage    VARCHAR(50),  -- 'initial', 'development', 'mid_season', 'late_season'
    season_year     INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_crops_field ON field_crop_seasons(field_id);
CREATE INDEX idx_field_crops_season ON field_crop_seasons(season_year);

CREATE TABLE infrastructure_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    asset_type      VARCHAR(50) NOT NULL CHECK (asset_type IN ('pipe', 'pump', 'valve', 'canal', 'reservoir', 'well', 'meter', 'gate')),
    name            VARCHAR(255) NOT NULL,
    location        GEOMETRY(Point, 4326),
    path            GEOMETRY(LineString, 4326),  -- for linear assets like pipes/canals
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    install_date    DATE,
    specifications  JSONB DEFAULT '{}',
    -- specifications example: {"diameter_mm": 150, "material": "PVC", "capacity_lps": 25.0}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assets_org ON infrastructure_assets(organisation_id);
CREATE INDEX idx_assets_type ON infrastructure_assets(asset_type);
CREATE INDEX idx_assets_location ON infrastructure_assets USING GIST(location);
```

---

## Sensors & Observations (OGC SensorThings-aligned)

```sql
CREATE TABLE sensor_devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    device_type     VARCHAR(50) NOT NULL CHECK (device_type IN ('soil_moisture', 'weather_station', 'flow_meter', 'pressure', 'canopy_temp', 'stem_sensor', 'water_level', 'et_sensor')),
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    serial_number   VARCHAR(255),
    connectivity    VARCHAR(50) CHECK (connectivity IN ('lorawan', 'cellular', 'wifi', 'satellite', 'manual')),
    location        GEOMETRY(Point, 4326),
    field_id        UUID REFERENCES fields(id),
    zone_id         UUID REFERENCES irrigation_zones(id),
    asset_id        UUID REFERENCES infrastructure_assets(id),
    install_depth_m NUMERIC(4,2),          -- depth for soil sensors
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    battery_pct     NUMERIC(5,2),
    last_seen_at    TIMESTAMPTZ,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensors_org ON sensor_devices(organisation_id);
CREATE INDEX idx_sensors_field ON sensor_devices(field_id);
CREATE INDEX idx_sensors_type ON sensor_devices(device_type);
CREATE INDEX idx_sensors_status ON sensor_devices(status);

CREATE TABLE observed_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,  -- 'soil_moisture_vwc', 'temperature_air', 'wind_speed', etc.
    description     TEXT,
    unit_of_measure VARCHAR(50) NOT NULL,   -- 'm3/m3', '°C', 'm/s', 'mm', 'kPa', '%'
    property_type   VARCHAR(50),            -- 'soil', 'weather', 'flow', 'plant', 'water_quality'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE datastreams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensor_devices(id) ON DELETE CASCADE,
    observed_property_id UUID NOT NULL REFERENCES observed_properties(id),
    name            VARCHAR(255),
    observation_type VARCHAR(50) NOT NULL DEFAULT 'measurement',
    unit_of_measure VARCHAR(50) NOT NULL,
    observed_area   GEOMETRY(Polygon, 4326),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(sensor_id, observed_property_id)
);
CREATE INDEX idx_datastreams_sensor ON datastreams(sensor_id);

CREATE TABLE observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    datastream_id   UUID NOT NULL REFERENCES datastreams(id) ON DELETE CASCADE,
    phenomenon_time TIMESTAMPTZ NOT NULL,   -- OGC: when the observation occurred
    result_time     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- OGC: when the result was recorded
    result_value    NUMERIC(15,6) NOT NULL,
    quality_code    VARCHAR(20) DEFAULT 'good',  -- WaterML 2.0: 'good', 'suspect', 'missing', 'estimated'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_observations_datastream_time ON observations(datastream_id, phenomenon_time DESC);
CREATE INDEX idx_observations_time ON observations(phenomenon_time DESC);

-- Partition observations by month for performance on large datasets
-- ALTER TABLE observations PARTITION BY RANGE (phenomenon_time);
```

---

## Weather & Evapotranspiration

```sql
CREATE TABLE weather_stations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    source          VARCHAR(100) NOT NULL,  -- 'on_site', 'noaa', 'openweather', 'davis', 'openet'
    location        GEOMETRY(Point, 4326) NOT NULL,
    elevation_m     NUMERIC(8,2),
    organisation_id UUID REFERENCES organisations(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_weather_stations_location ON weather_stations USING GIST(location);

CREATE TABLE weather_observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES weather_stations(id) ON DELETE CASCADE,
    observed_at     TIMESTAMPTZ NOT NULL,
    temp_max_c      NUMERIC(5,2),     -- FAO-56 input
    temp_min_c      NUMERIC(5,2),     -- FAO-56 input
    temp_mean_c     NUMERIC(5,2),     -- FAO-56 input
    humidity_pct    NUMERIC(5,2),     -- FAO-56 input: relative humidity
    wind_speed_ms   NUMERIC(6,2),     -- FAO-56 input: m/s at 2m height
    solar_rad_mjm2  NUMERIC(8,4),     -- FAO-56 input: MJ/m²/day
    precipitation_mm NUMERIC(8,2),
    pressure_kpa    NUMERIC(7,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_weather_obs_station_time ON weather_observations(station_id, observed_at DESC);

CREATE TABLE et_calculations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id),
    calculation_date DATE NOT NULL,
    et_reference_mm NUMERIC(8,4),     -- FAO-56 ETo (reference ET)
    crop_coefficient NUMERIC(4,2),    -- Kc for current growth stage
    et_crop_mm      NUMERIC(8,4),     -- ETc = ETo * Kc
    method          VARCHAR(50) NOT NULL DEFAULT 'fao56_penman_monteith',
    source_station_id UUID REFERENCES weather_stations(id),
    source_satellite VARCHAR(100),    -- 'openet_ensemble', 'modis', 'landsat'
    ndvi_value      NUMERIC(5,4),
    confidence      NUMERIC(4,2),     -- 0-1 confidence score
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_et_field_date ON et_calculations(field_id, calculation_date DESC);
```

---

## Water Rights & Allocations

```sql
CREATE TABLE water_rights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    right_number    VARCHAR(100) NOT NULL,
    right_type      VARCHAR(50) NOT NULL CHECK (right_type IN ('appropriative', 'riparian', 'prescriptive', 'pueblo', 'federal_reserved', 'groundwater')),
    priority_date   DATE,                            -- HarDWR: critical for prior appropriation
    use_category    VARCHAR(50) NOT NULL,             -- HarDWR: 'irrigation', 'municipal', 'industrial', 'stock', 'domestic', 'environmental', 'other'
    water_source    VARCHAR(255),                     -- HarDWR: name of waterbody or well
    source_type     VARCHAR(50) CHECK (source_type IN ('surface', 'groundwater', 'recycled', 'mixed')),
    max_flow_cfs    NUMERIC(12,4),                    -- cubic feet per second
    max_volume_af   NUMERIC(12,4),                    -- acre-feet per year
    point_of_diversion GEOMETRY(Point, 4326),         -- HarDWR: PoD coordinates
    place_of_use    GEOMETRY(MultiPolygon, 4326),     -- authorised use area
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    adjudication_status VARCHAR(50),
    permit_number   VARCHAR(100),
    expiry_date     DATE,
    conditions      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_water_rights_org ON water_rights(organisation_id);
CREATE INDEX idx_water_rights_jurisdiction ON water_rights(jurisdiction_id);
CREATE INDEX idx_water_rights_priority ON water_rights(priority_date);
CREATE INDEX idx_water_rights_pod ON water_rights USING GIST(point_of_diversion);

CREATE TABLE water_allocations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    water_right_id  UUID NOT NULL REFERENCES water_rights(id),
    allocation_year INTEGER NOT NULL,
    allocated_volume_af NUMERIC(12,4) NOT NULL,
    used_volume_af  NUMERIC(12,4) DEFAULT 0,
    remaining_af    NUMERIC(12,4) GENERATED ALWAYS AS (allocated_volume_af - used_volume_af) STORED,
    allocation_date DATE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_allocations_right_year ON water_allocations(water_right_id, allocation_year);

CREATE TABLE water_usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    allocation_id   UUID NOT NULL REFERENCES water_allocations(id),
    field_id        UUID REFERENCES fields(id),
    zone_id         UUID REFERENCES irrigation_zones(id),
    meter_id        UUID REFERENCES sensor_devices(id),
    usage_date      DATE NOT NULL,
    volume_af       NUMERIC(12,6) NOT NULL,
    flow_rate_cfs   NUMERIC(10,4),
    reading_type    VARCHAR(50) NOT NULL DEFAULT 'metered',  -- 'metered', 'estimated', 'manual'
    source          VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_usage_allocation_date ON water_usage_records(allocation_id, usage_date);
CREATE INDEX idx_usage_field ON water_usage_records(field_id);

CREATE TABLE compliance_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    water_right_id  UUID NOT NULL REFERENCES water_rights(id),
    check_date      DATE NOT NULL,
    check_type      VARCHAR(50) NOT NULL,  -- 'usage_limit', 'flow_rate', 'seasonal_restriction', 'curtailment'
    status          VARCHAR(50) NOT NULL,  -- 'compliant', 'warning', 'violation', 'pending'
    allocated_limit NUMERIC(12,4),
    actual_value    NUMERIC(12,4),
    details         TEXT,
    auto_generated  BOOLEAN NOT NULL DEFAULT false,
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_compliance_right ON compliance_checks(water_right_id);
CREATE INDEX idx_compliance_status ON compliance_checks(status);
```

---

## Irrigation Scheduling & Orders

```sql
CREATE TABLE irrigation_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id),
    zone_id         UUID REFERENCES irrigation_zones(id),
    schedule_type   VARCHAR(50) NOT NULL CHECK (schedule_type IN ('ai_recommendation', 'manual', 'rule_based', 'calendar')),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ,
    duration_minutes INTEGER,
    volume_target_mm NUMERIC(8,2),        -- target application depth
    flow_rate_lps   NUMERIC(10,4),
    status          VARCHAR(50) NOT NULL DEFAULT 'scheduled',  -- 'scheduled', 'approved', 'in_progress', 'completed', 'cancelled'
    ai_confidence   NUMERIC(4,2),          -- 0-1 for AI-generated schedules
    reasoning       TEXT,                  -- AI explanation for recommendation
    created_by      UUID REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_schedules_field ON irrigation_schedules(field_id);
CREATE INDEX idx_schedules_status ON irrigation_schedules(status);
CREATE INDEX idx_schedules_start ON irrigation_schedules(scheduled_start);

CREATE TABLE water_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    order_number    VARCHAR(50) NOT NULL,
    ordered_by      UUID NOT NULL REFERENCES users(id),
    delivery_date   DATE NOT NULL,
    delivery_start  TIME,
    delivery_end    TIME,
    volume_af       NUMERIC(12,4),
    flow_rate_cfs   NUMERIC(10,4),
    delivery_point  VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'approved', 'delivering', 'completed', 'cancelled'
    assigned_rider  UUID REFERENCES users(id),   -- ditch rider for field delivery
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_org ON water_orders(organisation_id);
CREATE INDEX idx_orders_status ON water_orders(status);
CREATE INDEX idx_orders_delivery ON water_orders(delivery_date);
```

---

## Billing & Accounts

```sql
CREATE TABLE billing_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    account_number  VARCHAR(100) NOT NULL UNIQUE,
    account_holder  VARCHAR(255) NOT NULL,
    contact_email   VARCHAR(255),
    billing_address TEXT,
    rate_schedule_id UUID REFERENCES rate_schedules(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rate_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    rate_type       VARCHAR(50) NOT NULL,  -- 'flat', 'tiered', 'seasonal', 'volumetric'
    rate_per_af     NUMERIC(10,2),
    tiers           JSONB,
    -- tiers example: [{"min_af": 0, "max_af": 100, "rate": 25.00}, {"min_af": 100, "max_af": null, "rate": 35.00}]
    effective_date  DATE NOT NULL,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    billing_account_id UUID NOT NULL REFERENCES billing_accounts(id),
    invoice_number  VARCHAR(100) NOT NULL UNIQUE,
    billing_period_start DATE NOT NULL,
    billing_period_end   DATE NOT NULL,
    total_volume_af NUMERIC(12,4),
    subtotal        NUMERIC(12,2) NOT NULL,
    tax             NUMERIC(12,2) DEFAULT 0,
    total           NUMERIC(12,2) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',  -- 'draft', 'issued', 'paid', 'overdue', 'cancelled'
    issued_at       TIMESTAMPTZ,
    due_date        DATE,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_invoices_account ON invoices(billing_account_id);
CREATE INDEX idx_invoices_status ON invoices(status);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,
    water_right_id  UUID REFERENCES water_rights(id),
    volume_af       NUMERIC(12,4),
    rate            NUMERIC(10,2),
    amount          NUMERIC(12,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Maintenance & Work Orders

```sql
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    asset_id        UUID REFERENCES infrastructure_assets(id),
    sensor_id       UUID REFERENCES sensor_devices(id),
    work_order_number VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',  -- 'critical', 'high', 'medium', 'low'
    status          VARCHAR(50) NOT NULL DEFAULT 'open',     -- 'open', 'assigned', 'in_progress', 'completed', 'cancelled'
    work_type       VARCHAR(50) NOT NULL,  -- 'preventive', 'corrective', 'inspection', 'replacement'
    assigned_to     UUID REFERENCES users(id),
    scheduled_date  DATE,
    completed_date  DATE,
    ai_generated    BOOLEAN NOT NULL DEFAULT false,  -- true if created by predictive maintenance AI
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_work_orders_org ON work_orders(organisation_id);
CREATE INDEX idx_work_orders_status ON work_orders(status);
CREATE INDEX idx_work_orders_asset ON work_orders(asset_id);
```

---

## Alerts & Notifications

```sql
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    alert_type      VARCHAR(50) NOT NULL,  -- 'soil_threshold', 'flow_anomaly', 'compliance_warning', 'sensor_offline', 'frost_risk', 'maintenance_due'
    severity        VARCHAR(20) NOT NULL,  -- 'critical', 'warning', 'info'
    title           VARCHAR(255) NOT NULL,
    message         TEXT,
    source_type     VARCHAR(50),           -- 'sensor', 'compliance', 'weather', 'ai_model'
    source_id       UUID,                  -- polymorphic reference to source entity
    field_id        UUID REFERENCES fields(id),
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alerts_org ON alerts(organisation_id);
CREATE INDEX idx_alerts_type ON alerts(alert_type);
CREATE INDEX idx_alerts_unresolved ON alerts(organisation_id) WHERE resolved_at IS NULL;

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,
    condition       JSONB NOT NULL,
    -- condition example: {"property": "soil_moisture_vwc", "operator": "lt", "value": 0.15, "duration_minutes": 60}
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    notify_roles    JSONB DEFAULT '[]',    -- ["org_admin", "field_operator"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(50) NOT NULL,   -- 'create', 'update', 'delete', 'approve', 'login'
    entity_type     VARCHAR(100) NOT NULL,  -- 'water_right', 'irrigation_schedule', 'water_order', etc.
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Auth | 5 | organisations, jurisdictions, users, roles, user_org_roles |
| Fields & Infrastructure | 5 | fields, irrigation_zones, crops, field_crop_seasons, infrastructure_assets |
| Sensors & Observations | 4 | sensor_devices, observed_properties, datastreams, observations |
| Weather & ET | 3 | weather_stations, weather_observations, et_calculations |
| Water Rights | 4 | water_rights, water_allocations, water_usage_records, compliance_checks |
| Irrigation Operations | 2 | irrigation_schedules, water_orders |
| Billing | 4 | billing_accounts, rate_schedules, invoices, invoice_line_items |
| Maintenance | 1 | work_orders |
| Alerts | 2 | alerts, alert_rules |
| Audit | 1 | audit_log |
| **Total** | **31** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, merging datasets from multiple districts, and avoiding sequential ID enumeration attacks.

2. **PostGIS geometry columns for all spatial data** — field boundaries, sensor locations, water right points of diversion, and infrastructure paths are stored as native PostGIS types with GIST indexes, enabling spatial queries (containment, proximity, intersection) at the database level.

3. **OGC SensorThings entity alignment** — the sensor_devices/datastreams/observations structure directly mirrors the Thing/Datastream/Observation pattern from OGC SensorThings API, making it straightforward to expose a standards-compliant API layer.

4. **HarDWR-aligned water rights fields** — priority_date, use_category, water_source, max_flow_cfs, max_volume_af, and point_of_diversion match the harmonised field set used across 11 Western US states, enabling regulatory data exchange.

5. **FAO-56 parameters as explicit columns** — weather observations include the exact fields required for Penman-Monteith ET calculation (temp_max, temp_min, humidity, wind_speed, solar_radiation), and crop coefficients (Kc_ini, Kc_mid, Kc_end) are stored per crop rather than buried in JSON.

6. **Observations table designed for partitioning** — the observations table will grow fastest and should be range-partitioned by phenomenon_time (monthly or quarterly) for query performance and data lifecycle management.

7. **Generated column for water allocation remaining** — `remaining_af` is a stored generated column derived from allocated minus used, ensuring consistency without application-level calculation.

8. **Polymorphic alert source** — alerts reference their trigger source via source_type + source_id rather than separate foreign keys per entity type, keeping the alerts table flexible as new alert sources are added.

9. **JSONB used sparingly** — only for genuinely variable data (rate schedule tiers, alert rule conditions, infrastructure specifications, role permissions) where the set of keys varies by record. All core domain fields are explicit columns.

10. **Row-level security ready** — organisation_id is present on every tenant-scoped table, enabling PostgreSQL RLS policies for multi-tenant isolation without schema-per-tenant complexity.
