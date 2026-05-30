# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Water Resource Management · Created: 2026-05-25

## Philosophy

This model treats every state change in the system as an immutable event appended to an event store. The current state of any entity — a water right, a field's irrigation status, an allocation balance — is derived by replaying the events that affected it. Materialised read models (views) are projected from the event stream using CQRS (Command Query Responsibility Segregation), allowing the write path (event append) and read path (denormalised projections) to be optimised independently.

This approach is drawn from patterns used in financial ledger systems, SCADA event logging, and regulatory compliance platforms where the question "what was the state on date X?" is as important as "what is the state now?". In water resource management, this is directly relevant: regulators need to audit historical usage against allocations, operators need to understand what irrigation decisions were made and why, and compliance officers need to prove that thresholds were never exceeded during a specific period.

The event store also provides a natural foundation for AI analytics. Machine learning models can train on the full history of irrigation decisions, water usage patterns, and compliance outcomes. The event stream can feed real-time anomaly detection for flow/pressure data and power continuous compliance monitoring against water rights entitlements.

**Best for:** Deployments where full audit trails are mandatory, temporal queries are frequent, and AI models need rich historical data for training and inference.

**Trade-offs:**
- (+) Complete audit trail — every change is preserved, nothing is lost
- (+) Temporal queries are natural — replay to any point in time
- (+) Event stream feeds AI/ML pipelines, anomaly detection, and compliance engines
- (+) Write path is simple and fast — append-only
- (+) Schema evolution is easier — new event types don't require migrations on existing data
- (-) Read queries require materialised views that must be kept in sync
- (-) Higher infrastructure complexity — event store + projection engine + read store
- (-) Eventual consistency between write and read models
- (-) Debugging requires understanding event replay, not just inspecting current state
- (-) Storage grows faster than a mutable model (events are never deleted)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OGC WaterML 2.0 | Time series observations naturally map to events with phenomenon_time and result_time |
| OGC SensorThings API | Sensor observations are a natural event type; the observation entity IS an event |
| HarDWR v1 | Water rights state changes (creation, modification, transfer, curtailment) become events with HarDWR-aligned fields |
| FAO-56 Penman-Monteith | ET calculations become events with full input parameters preserved, enabling recalculation with improved models |
| ISO 14046 | Water footprint calculations are events, enabling temporal footprint tracking |
| OCSF | Event schema structure draws from Open Cybersecurity Schema Framework patterns for structured audit events |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,         -- aggregate root identifier
    stream_type     VARCHAR(100) NOT NULL,  -- 'WaterRight', 'Field', 'Allocation', 'IrrigationSchedule', 'Sensor', 'WaterOrder', 'BillingAccount'
    event_type      VARCHAR(100) NOT NULL,  -- 'WaterRightCreated', 'AllocationUsageRecorded', 'IrrigationScheduled', etc.
    event_version   INTEGER NOT NULL,       -- monotonically increasing per stream
    organisation_id UUID NOT NULL,          -- tenant scoping
    actor_id        UUID,                   -- user or system that caused the event
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',  -- 'user', 'system', 'ai_agent', 'sensor'
    occurred_at     TIMESTAMPTZ NOT NULL,   -- when the real-world event happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when it was written to the store
    payload         JSONB NOT NULL,         -- event-specific data
    metadata        JSONB DEFAULT '{}',     -- correlation IDs, causation IDs, IP address, etc.
    UNIQUE(stream_id, event_version)
);

-- Primary query pattern: replay all events for a specific aggregate
CREATE INDEX idx_events_stream ON events(stream_id, event_version);

-- Query pattern: all events of a type across the system (for projections)
CREATE INDEX idx_events_type ON events(event_type, recorded_at);

-- Query pattern: all events for a tenant (for tenant-wide analytics)
CREATE INDEX idx_events_org ON events(organisation_id, recorded_at);

-- Query pattern: time-range scans for compliance audits
CREATE INDEX idx_events_occurred ON events(occurred_at);

-- Partition by month for lifecycle management and query performance
-- CREATE TABLE events PARTITION BY RANGE (recorded_at);
```

---

## Event Type Catalogue

### Water Rights Events

```sql
-- Event: WaterRightCreated
-- payload example:
-- {
--   "right_number": "WR-2024-00451",
--   "right_type": "appropriative",
--   "priority_date": "1952-06-15",
--   "use_category": "irrigation",
--   "water_source": "Bear Creek",
--   "source_type": "surface",
--   "max_flow_cfs": 2.5,
--   "max_volume_af": 150.0,
--   "point_of_diversion": {"type": "Point", "coordinates": [-121.4, 38.5]},
--   "jurisdiction_iso": "US-CA",
--   "legal_framework": "prior_appropriation"
-- }

-- Event: WaterRightTransferred
-- payload example:
-- {
--   "from_organisation_id": "uuid-old-owner",
--   "to_organisation_id": "uuid-new-owner",
--   "transfer_type": "permanent",
--   "transfer_volume_af": 150.0,
--   "regulatory_approval_number": "DWR-T-2025-1234"
-- }

-- Event: WaterRightCurtailed
-- payload example:
-- {
--   "curtailment_order": "SWRCB-2025-0042",
--   "curtailment_pct": 25,
--   "effective_date": "2025-07-01",
--   "reason": "drought_emergency"
-- }
```

### Allocation & Usage Events

```sql
-- Event: AllocationGranted
-- payload example:
-- {
--   "water_right_id": "uuid-right",
--   "allocation_year": 2025,
--   "allocated_volume_af": 120.0,
--   "allocation_date": "2025-03-01"
-- }

-- Event: WaterUsageRecorded
-- payload example:
-- {
--   "allocation_id": "uuid-allocation",
--   "field_id": "uuid-field",
--   "zone_id": "uuid-zone",
--   "meter_id": "uuid-meter",
--   "usage_date": "2025-07-15",
--   "volume_af": 0.35,
--   "flow_rate_cfs": 1.2,
--   "reading_type": "metered"
-- }

-- Event: ComplianceViolationDetected
-- payload example:
-- {
--   "water_right_id": "uuid-right",
--   "violation_type": "usage_exceeds_allocation",
--   "allocated_limit_af": 120.0,
--   "actual_usage_af": 125.3,
--   "detection_method": "ai_continuous_monitor"
-- }
```

### Sensor & Observation Events

```sql
-- Event: SensorRegistered
-- payload example:
-- {
--   "device_type": "soil_moisture",
--   "manufacturer": "CropX",
--   "model": "Apex",
--   "serial_number": "CX-2025-00891",
--   "connectivity": "lorawan",
--   "location": {"type": "Point", "coordinates": [-121.4, 38.5]},
--   "field_id": "uuid-field",
--   "install_depth_m": 0.30
-- }

-- Event: SensorObservationReceived
-- payload example:
-- {
--   "sensor_id": "uuid-sensor",
--   "observed_property": "soil_moisture_vwc",
--   "phenomenon_time": "2025-07-15T14:30:00Z",
--   "result_value": 0.22,
--   "unit": "m3/m3",
--   "quality_code": "good"
-- }

-- Event: SensorAnomalyDetected
-- payload example:
-- {
--   "sensor_id": "uuid-sensor",
--   "anomaly_type": "value_out_of_range",
--   "expected_range": {"min": 0.05, "max": 0.55},
--   "observed_value": 0.92,
--   "detection_model": "isolation_forest_v2",
--   "confidence": 0.94
-- }
```

### Irrigation Events

```sql
-- Event: IrrigationScheduleCreated
-- payload example:
-- {
--   "field_id": "uuid-field",
--   "zone_id": "uuid-zone",
--   "schedule_type": "ai_recommendation",
--   "scheduled_start": "2025-07-16T06:00:00Z",
--   "duration_minutes": 120,
--   "volume_target_mm": 25.0,
--   "ai_confidence": 0.87,
--   "reasoning": "Soil moisture at 0.15 VWC (below 0.20 threshold), ET forecast 6.2mm, no rain expected 5 days",
--   "et_reference_mm": 6.2,
--   "crop_coefficient": 1.05,
--   "soil_moisture_current": 0.15
-- }

-- Event: IrrigationStarted
-- Event: IrrigationCompleted
-- Event: IrrigationCancelled
```

### Weather & ET Events

```sql
-- Event: WeatherObservationRecorded
-- payload example:
-- {
--   "station_id": "uuid-station",
--   "observed_at": "2025-07-15T00:00:00Z",
--   "temp_max_c": 38.2,
--   "temp_min_c": 18.5,
--   "temp_mean_c": 28.3,
--   "humidity_pct": 35.0,
--   "wind_speed_ms": 2.1,
--   "solar_rad_mjm2": 28.5,
--   "precipitation_mm": 0.0
-- }

-- Event: ETCalculationCompleted
-- payload example:
-- {
--   "field_id": "uuid-field",
--   "calculation_date": "2025-07-15",
--   "method": "fao56_penman_monteith",
--   "et_reference_mm": 7.8,
--   "crop_coefficient": 1.05,
--   "et_crop_mm": 8.19,
--   "source_station_id": "uuid-station",
--   "ndvi_value": 0.72,
--   "confidence": 0.91,
--   "input_params": {
--     "temp_max_c": 38.2, "temp_min_c": 18.5,
--     "humidity_pct": 35.0, "wind_speed_ms": 2.1,
--     "solar_rad_mjm2": 28.5, "elevation_m": 125.0,
--     "latitude": 38.5
--   }
-- }
```

---

## Materialised Read Models (Projections)

These tables are derived from the event store and can be rebuilt from scratch by replaying events. They are the query-side of CQRS.

```sql
-- Current state of water rights (projected from WaterRight* events)
CREATE TABLE rm_water_rights (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    right_number    VARCHAR(100) NOT NULL,
    right_type      VARCHAR(50) NOT NULL,
    priority_date   DATE,
    use_category    VARCHAR(50) NOT NULL,
    water_source    VARCHAR(255),
    source_type     VARCHAR(50),
    max_flow_cfs    NUMERIC(12,4),
    max_volume_af   NUMERIC(12,4),
    current_curtailment_pct NUMERIC(5,2) DEFAULT 0,
    status          VARCHAR(50) NOT NULL,
    point_of_diversion GEOMETRY(Point, 4326),
    last_event_id   UUID NOT NULL,         -- last event that updated this projection
    last_event_at   TIMESTAMPTZ NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_rights_org ON rm_water_rights(organisation_id);

-- Current allocation balances (projected from Allocation* and Usage* events)
CREATE TABLE rm_allocation_balances (
    id              UUID PRIMARY KEY,
    water_right_id  UUID NOT NULL,
    allocation_year INTEGER NOT NULL,
    allocated_volume_af NUMERIC(12,4),
    used_volume_af  NUMERIC(12,4),
    remaining_af    NUMERIC(12,4),
    usage_pct       NUMERIC(5,2),
    last_usage_date DATE,
    compliance_status VARCHAR(50),          -- 'on_track', 'approaching_limit', 'exceeded'
    last_event_id   UUID NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_balances_right ON rm_allocation_balances(water_right_id);

-- Current field status (projected from sensor, irrigation, crop events)
CREATE TABLE rm_field_status (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    field_name      VARCHAR(255),
    boundary        GEOMETRY(Polygon, 4326),
    area_hectares   NUMERIC(10,4),
    current_crop    VARCHAR(255),
    growth_stage    VARCHAR(50),
    latest_soil_moisture NUMERIC(6,4),
    latest_soil_moisture_at TIMESTAMPTZ,
    latest_et_mm    NUMERIC(8,4),
    latest_et_date  DATE,
    irrigation_status VARCHAR(50),          -- 'idle', 'scheduled', 'irrigating', 'recently_irrigated'
    next_irrigation_at TIMESTAMPTZ,
    active_sensor_count INTEGER DEFAULT 0,
    alert_count     INTEGER DEFAULT 0,
    last_event_id   UUID NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_field_org ON rm_field_status(organisation_id);
CREATE INDEX idx_rm_field_boundary ON rm_field_status USING GIST(boundary);

-- Sensor latest readings (projected from SensorObservationReceived events)
CREATE TABLE rm_sensor_latest (
    sensor_id       UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    device_type     VARCHAR(50),
    field_id        UUID,
    location        GEOMETRY(Point, 4326),
    status          VARCHAR(50),
    battery_pct     NUMERIC(5,2),
    last_observation_at TIMESTAMPTZ,
    readings        JSONB NOT NULL DEFAULT '{}',
    -- readings example: {
    --   "soil_moisture_vwc": {"value": 0.22, "unit": "m3/m3", "at": "2025-07-15T14:30:00Z"},
    --   "soil_temperature_c": {"value": 24.5, "unit": "°C", "at": "2025-07-15T14:30:00Z"},
    --   "electrical_conductivity": {"value": 1.2, "unit": "dS/m", "at": "2025-07-15T14:30:00Z"}
    -- }
    last_event_id   UUID NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_sensor_org ON rm_sensor_latest(organisation_id);
CREATE INDEX idx_rm_sensor_field ON rm_sensor_latest(field_id);
```

---

## Time Series Projection (for dashboards and analytics)

```sql
-- Observations time series — optimised for range queries and dashboards
-- This table can use TimescaleDB hypertable for automatic partitioning
CREATE TABLE rm_observations_ts (
    time            TIMESTAMPTZ NOT NULL,
    sensor_id       UUID NOT NULL,
    organisation_id UUID NOT NULL,
    field_id        UUID,
    property_name   VARCHAR(100) NOT NULL,
    value           NUMERIC(15,6) NOT NULL,
    unit            VARCHAR(50) NOT NULL,
    quality_code    VARCHAR(20) DEFAULT 'good',
    event_id        UUID NOT NULL
);
-- SELECT create_hypertable('rm_observations_ts', 'time');  -- TimescaleDB
CREATE INDEX idx_rm_obs_sensor_time ON rm_observations_ts(sensor_id, time DESC);
CREATE INDEX idx_rm_obs_field_time ON rm_observations_ts(field_id, time DESC);
CREATE INDEX idx_rm_obs_org_time ON rm_observations_ts(organisation_id, time DESC);

-- Daily aggregates (continuous aggregate or materialised by projection engine)
CREATE TABLE rm_daily_aggregates (
    date            DATE NOT NULL,
    sensor_id       UUID NOT NULL,
    field_id        UUID,
    organisation_id UUID NOT NULL,
    property_name   VARCHAR(100) NOT NULL,
    min_value       NUMERIC(15,6),
    max_value       NUMERIC(15,6),
    avg_value       NUMERIC(15,6),
    count           INTEGER,
    PRIMARY KEY(date, sensor_id, property_name)
);
CREATE INDEX idx_rm_daily_field ON rm_daily_aggregates(field_id, date DESC);
```

---

## Projection Tracking

```sql
-- Tracks the position of each projection in the event stream
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'running',  -- 'running', 'paused', 'rebuilding', 'error'
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed projection
CREATE TABLE projection_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL REFERENCES events(event_id),
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Snapshot Store (for aggregate performance)

```sql
-- Periodic snapshots of aggregate state to avoid replaying all events
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,     -- event_version at time of snapshot
    state           JSONB NOT NULL,        -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY(stream_id, snapshot_version)
);
```

---

## Reference Data (non-event-sourced)

```sql
-- These tables are static reference data, not event-sourced
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    organisation_id UUID REFERENCES organisations(id),
    role            VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    iso_3166_code   VARCHAR(10),
    jurisdiction_type VARCHAR(50) NOT NULL,
    parent_id       UUID REFERENCES jurisdictions(id),
    boundary        GEOMETRY(MultiPolygon, 4326),
    legal_framework VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE crops (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    variety         VARCHAR(255),
    kc_ini          NUMERIC(4,2),
    kc_mid          NUMERIC(4,2),
    kc_end          NUMERIC(4,2),
    root_depth_m    NUMERIC(4,2),
    growing_days    INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE observed_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT,
    unit_of_measure VARCHAR(50) NOT NULL,
    property_type   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Temporal query: water right state on a specific date

```sql
-- What was the state of water right WR-2024-00451 on 2025-06-01?
SELECT payload
FROM events
WHERE stream_id = (
    SELECT event_id FROM events
    WHERE stream_type = 'WaterRight'
      AND payload->>'right_number' = 'WR-2024-00451'
    LIMIT 1
)
  AND stream_type = 'WaterRight'
  AND occurred_at <= '2025-06-01T23:59:59Z'
ORDER BY event_version ASC;
-- Application code replays these events to reconstruct state at that date
```

### Compliance audit: total usage vs allocation for a period

```sql
-- Sum all usage events for a water right in July 2025
SELECT
    SUM((payload->>'volume_af')::numeric) AS total_usage_af
FROM events
WHERE event_type = 'WaterUsageRecorded'
  AND payload->>'allocation_id' = 'uuid-allocation'
  AND occurred_at BETWEEN '2025-07-01' AND '2025-07-31';
```

### Event stream analytics: irrigation decision patterns

```sql
-- All irrigation decisions for a field with their reasoning
SELECT
    occurred_at,
    payload->>'schedule_type' AS schedule_type,
    payload->>'volume_target_mm' AS volume_mm,
    payload->>'ai_confidence' AS confidence,
    payload->>'reasoning' AS reasoning,
    payload->>'soil_moisture_current' AS soil_moisture
FROM events
WHERE event_type = 'IrrigationScheduleCreated'
  AND payload->>'field_id' = 'uuid-field'
  AND occurred_at >= '2025-01-01'
ORDER BY occurred_at;
```

### Read model query: current field dashboard (from projection)

```sql
-- Fast dashboard query — no event replay needed
SELECT
    f.field_name,
    f.current_crop,
    f.growth_stage,
    f.latest_soil_moisture,
    f.latest_et_mm,
    f.irrigation_status,
    f.next_irrigation_at,
    f.active_sensor_count,
    f.alert_count
FROM rm_field_status f
WHERE f.organisation_id = 'uuid-org'
ORDER BY f.field_name;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by time) |
| Projections — Entity State | 4 | rm_water_rights, rm_allocation_balances, rm_field_status, rm_sensor_latest |
| Projections — Time Series | 2 | rm_observations_ts (hypertable), rm_daily_aggregates |
| Projection Infrastructure | 2 | projection_checkpoints, projection_dead_letters |
| Snapshots | 1 | snapshots |
| Reference Data | 5 | organisations, users, jurisdictions, crops, observed_properties |
| **Total** | **15** | Plus partitions for events and time series |

---

## Key Design Decisions

1. **Single events table as source of truth** — all domain state changes flow through one append-only table. This guarantees a complete, ordered history of everything that happened and eliminates the possibility of "silent" data mutations.

2. **Event payload is JSONB** — each event type has its own payload schema documented in the event catalogue, but stored as JSONB to allow schema evolution without table migrations. New event types are added by deploying new code, not by running ALTER TABLE.

3. **stream_id + event_version for optimistic concurrency** — the UNIQUE constraint on (stream_id, event_version) prevents concurrent conflicting writes to the same aggregate. Application code reads the current version, appends at version + 1, and retries on conflict.

4. **Materialised read models are disposable** — every rm_* table can be dropped and rebuilt from the event store. This means read model schema changes are zero-risk: deploy new projection code, rebuild the projection, and switch reads to the new table.

5. **TimescaleDB for observation time series** — the rm_observations_ts projection uses a TimescaleDB hypertable for automatic time-based partitioning, compression, and continuous aggregates. This handles the high-volume sensor data that would otherwise overwhelm a regular table.

6. **Reference data is not event-sourced** — organisations, users, jurisdictions, and crops change infrequently and don't need temporal history. Keeping them as regular CRUD tables reduces complexity without losing auditability (the events table still records who did what).

7. **Snapshots for aggregate performance** — long-lived aggregates (water rights with decades of history) would be expensive to rebuild from events. Periodic snapshots store the current state, and replay starts from the snapshot rather than event #1.

8. **FAO-56 input parameters preserved in ET events** — every ET calculation event includes all input parameters (temperatures, humidity, wind speed, solar radiation, elevation, latitude), enabling recalculation with improved models without re-collecting data.

9. **AI reasoning captured in events** — irrigation schedule events include the AI model's reasoning, confidence score, and input signals. This creates a searchable corpus for evaluating AI performance and explaining recommendations to operators and regulators.

10. **Projection dead-letter queue** — events that fail projection processing are captured rather than blocking the pipeline, ensuring the system remains available even when individual events have data quality issues.
