# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Water Resource Management · Created: 2026-05-25

## Philosophy

This model combines a relational backbone for operational CRUD (sensor readings, irrigation schedules, billing) with a property graph layer for relationship-intensive queries that are awkward or slow in pure relational designs. In water resource management, many critical questions are inherently graph problems: "Which fields depend on water from this canal, and what happens if it is curtailed?" "Which water rights downstream of this diversion point are affected by a contamination event?" "Show all infrastructure between the reservoir and this turnout." "Who shares water sources with whom, and are there conflicts?"

The graph layer uses a pair of generic tables (graph_nodes and graph_edges) that form a property graph within PostgreSQL. This avoids the operational complexity of running a separate graph database (Neo4j, Neptune) while enabling multi-hop traversal queries using recursive CTEs. For deployments that outgrow PostgreSQL's graph capabilities, the same node/edge model maps directly to Neo4j's property graph model or Amazon Neptune.

Research shows that graph databases have been successfully applied to water distribution systems, where network topology queries (flow paths between pumps and junctions, leak impact analysis, infrastructure dependency chains) are core requirements. The graph-relational hybrid brings this capability to a broader water resource management platform without abandoning relational strengths for billing, time-series observations, and regulatory compliance.

**Best for:** Deployments where infrastructure network analysis, water flow path tracing, impact assessment, and multi-entity relationship queries are primary use cases — especially irrigation districts managing complex canal/pipeline networks.

**Trade-offs:**
- (+) Multi-hop relationship queries (impact analysis, flow tracing, dependency chains) are natural and fast
- (+) Infrastructure network topology is directly queryable without materialising adjacency lists
- (+) Water rights conflict detection across shared sources is a graph traversal, not a complex JOIN
- (+) Graph model maps directly to Neo4j if PostgreSQL is outgrown
- (+) Relational tables handle operational CRUD and time-series data where graphs add no value
- (-) Recursive CTEs for graph traversal are more complex to write and optimise than simple JOINs
- (-) Two models (relational + graph) increase cognitive load for developers
- (-) Graph queries can be expensive if the graph is dense and unbounded traversal is allowed
- (-) Reporting tools have poor support for graph query patterns
- (-) Graph_nodes/graph_edges tables add indirection compared to direct foreign keys

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EPANET / WNTR | Water distribution network topology (junctions, pipes, pumps, reservoirs, tanks) maps directly to graph nodes and edges |
| OGC SensorThings API | Sensor-to-Thing-to-Location relationships modelled as graph edges; Datastream/Observation stays relational |
| HarDWR v1 | Water rights entities are both relational records and graph nodes connected to water sources, points of diversion, and places of use |
| OGC API — Features | Spatial features (field boundaries, infrastructure locations) are graph nodes with geometry properties |
| ISO 3166 | Jurisdiction hierarchy is a graph: country -> state -> county -> water district -> basin |
| GeoJSON / RFC 7946 | Graph nodes carry geometry properties for spatial + graph combined queries |

---

## Property Graph Layer

```sql
-- Generic node table: every domain entity that participates in relationships
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,
    -- node_type values:
    -- Infrastructure: 'reservoir', 'canal', 'pipe', 'pump', 'valve', 'gate', 'well', 'turnout', 'junction', 'meter'
    -- Land:           'field', 'irrigation_zone', 'parcel'
    -- Water:          'water_source', 'aquifer', 'watershed', 'water_right', 'allocation'
    -- Organisation:   'organisation', 'user'
    -- Sensor:         'sensor_device', 'weather_station'
    -- Administrative: 'jurisdiction', 'water_district', 'basin'
    label           VARCHAR(255) NOT NULL,         -- human-readable name
    organisation_id UUID,                          -- tenant scoping (NULL for shared reference data)
    entity_id       UUID,                          -- FK to the corresponding relational table record
    entity_table    VARCHAR(100),                   -- which relational table: 'fields', 'sensors', 'water_rights', etc.
    location        GEOMETRY(Point, 4326),          -- spatial position
    boundary        GEOMETRY(Polygon, 4326),        -- spatial extent
    properties      JSONB NOT NULL DEFAULT '{}',    -- node-specific attributes
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gn_type ON graph_nodes(node_type);
CREATE INDEX idx_gn_org ON graph_nodes(organisation_id);
CREATE INDEX idx_gn_entity ON graph_nodes(entity_table, entity_id);
CREATE INDEX idx_gn_location ON graph_nodes USING GIST(location);
CREATE INDEX idx_gn_boundary ON graph_nodes USING GIST(boundary);
CREATE INDEX idx_gn_properties ON graph_nodes USING GIN(properties);

-- Generic edge table: typed, directed relationships between nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    edge_type       VARCHAR(50) NOT NULL,
    -- edge_type values:
    -- Water flow:     'flows_to', 'diverts_from', 'feeds', 'drains_to'
    -- Infrastructure: 'connects_to', 'controlled_by', 'monitored_by', 'maintained_by'
    -- Water rights:   'authorised_by', 'diverts_from_source', 'applies_to_field', 'allocated_to'
    -- Organisation:   'owns', 'operates', 'manages', 'employs'
    -- Administrative: 'within_jurisdiction', 'part_of_basin', 'regulated_by'
    -- Sensor:         'installed_at', 'measures'
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    organisation_id UUID,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- flows_to:       {"capacity_cfs": 15.0, "length_m": 450, "friction_loss": 0.02}
    -- authorised_by:  {"permit_number": "A031234", "priority_date": "1952-06-15"}
    -- monitors:       {"datastream_ids": ["uuid-1", "uuid-2"], "install_date": "2024-03-15"}
    weight          NUMERIC(10,4),                  -- optional: distance, capacity, priority, or cost for graph algorithms
    is_active       BOOLEAN NOT NULL DEFAULT true,
    valid_from      TIMESTAMPTZ DEFAULT now(),       -- temporal validity
    valid_to        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ge_type ON graph_edges(edge_type);
CREATE INDEX idx_ge_source ON graph_edges(source_node_id);
CREATE INDEX idx_ge_target ON graph_edges(target_node_id);
CREATE INDEX idx_ge_org ON graph_edges(organisation_id);
CREATE INDEX idx_ge_active ON graph_edges(is_active) WHERE is_active = true;
CREATE INDEX idx_ge_properties ON graph_edges USING GIN(properties);
```

---

## Relational Backbone: Operational Tables

```sql
-- Organisations (also graph nodes)
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    parent_org_id   UUID REFERENCES organisations(id),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    organisation_id UUID REFERENCES organisations(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Fields (also graph nodes with boundary geometry)
CREATE TABLE fields (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    boundary        GEOMETRY(Polygon, 4326) NOT NULL,
    area_hectares   NUMERIC(10,4),
    soil_type       VARCHAR(100),
    current_crop    VARCHAR(255),
    growth_stage    VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fields_org ON fields(organisation_id);
CREATE INDEX idx_fields_boundary ON fields USING GIST(boundary);

-- Infrastructure assets (also graph nodes — the primary graph entities)
CREATE TABLE infrastructure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    asset_type      VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    location        GEOMETRY(Point, 4326),
    path            GEOMETRY(LineString, 4326),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    specifications  JSONB NOT NULL DEFAULT '{}',
    install_date    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_infra_org ON infrastructure(organisation_id);
CREATE INDEX idx_infra_type ON infrastructure(asset_type);
CREATE INDEX idx_infra_location ON infrastructure USING GIST(location);

-- Sensors (also graph nodes connected to fields and infrastructure via edges)
CREATE TABLE sensors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    device_type     VARCHAR(50) NOT NULL,
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    serial_number   VARCHAR(255),
    connectivity    VARCHAR(50),
    location        GEOMETRY(Point, 4326),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    battery_pct     NUMERIC(5,2),
    last_seen_at    TIMESTAMPTZ,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensors_org ON sensors(organisation_id);
CREATE INDEX idx_sensors_type ON sensors(device_type);

-- Water rights (also graph nodes connected to water sources and fields)
CREATE TABLE water_rights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    right_number    VARCHAR(100) NOT NULL,
    right_type      VARCHAR(50) NOT NULL,
    priority_date   DATE,
    use_category    VARCHAR(50) NOT NULL,
    water_source    VARCHAR(255),
    source_type     VARCHAR(50),
    max_flow_cfs    NUMERIC(12,4),
    max_volume_af   NUMERIC(12,4),
    point_of_diversion GEOMETRY(Point, 4326),
    place_of_use    GEOMETRY(MultiPolygon, 4326),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    jurisdiction_data JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rights_org ON water_rights(organisation_id);
CREATE INDEX idx_rights_pod ON water_rights USING GIST(point_of_diversion);
```

---

## Time-Series & Operational Tables (Pure Relational)

```sql
-- Observations (high-volume, not in graph)
CREATE TABLE observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensors(id) ON DELETE CASCADE,
    observed_at     TIMESTAMPTZ NOT NULL,
    property        VARCHAR(100) NOT NULL,
    value           NUMERIC(15,6) NOT NULL,
    unit            VARCHAR(50) NOT NULL,
    quality         VARCHAR(20) DEFAULT 'good',
    depth_m         NUMERIC(4,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_obs_sensor_time ON observations(sensor_id, observed_at DESC);
CREATE INDEX idx_obs_time ON observations(observed_at DESC);

-- Weather data
CREATE TABLE weather_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          VARCHAR(100) NOT NULL,
    source_id       VARCHAR(255),
    location        GEOMETRY(Point, 4326),
    observed_at     TIMESTAMPTZ NOT NULL,
    temp_max_c      NUMERIC(5,2),
    temp_min_c      NUMERIC(5,2),
    humidity_pct    NUMERIC(5,2),
    wind_speed_ms   NUMERIC(6,2),
    solar_rad_mjm2  NUMERIC(8,4),
    precipitation_mm NUMERIC(8,2),
    extended        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_weather_time ON weather_data(observed_at DESC);

-- ET calculations
CREATE TABLE et_calculations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id),
    calc_date       DATE NOT NULL,
    et_reference_mm NUMERIC(8,4),
    crop_coefficient NUMERIC(4,2),
    et_crop_mm      NUMERIC(8,4),
    method          VARCHAR(50) NOT NULL DEFAULT 'fao56_penman_monteith',
    inputs          JSONB NOT NULL,
    confidence      NUMERIC(4,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_et_field ON et_calculations(field_id, calc_date DESC);

-- Water usage
CREATE TABLE water_usage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    water_right_id  UUID NOT NULL REFERENCES water_rights(id),
    field_id        UUID REFERENCES fields(id),
    usage_date      DATE NOT NULL,
    volume_af       NUMERIC(12,6) NOT NULL,
    flow_rate_cfs   NUMERIC(10,4),
    reading_type    VARCHAR(50) NOT NULL DEFAULT 'metered',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_usage_right ON water_usage(water_right_id, usage_date);

-- Irrigation events
CREATE TABLE irrigation_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES fields(id),
    zone_id         UUID,
    event_type      VARCHAR(50) NOT NULL,
    schedule_type   VARCHAR(50),
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    volume_mm       NUMERIC(8,2),
    duration_minutes INTEGER,
    details         JSONB DEFAULT '{}',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_irrig_field ON irrigation_events(field_id);

-- Water orders
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
    details         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_org ON water_orders(organisation_id);

-- Alerts
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    message         TEXT,
    source_node_id  UUID REFERENCES graph_nodes(id),  -- link to graph for impact analysis
    field_id        UUID REFERENCES fields(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alerts_org ON alerts(organisation_id);
CREATE INDEX idx_alerts_unresolved ON alerts(organisation_id) WHERE resolved_at IS NULL;

-- Work orders
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    asset_node_id   UUID REFERENCES graph_nodes(id),  -- link to graph for impact analysis
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    assigned_to     UUID REFERENCES users(id),
    scheduled_date  DATE,
    completed_date  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_org ON work_orders(organisation_id);
CREATE INDEX idx_wo_status ON work_orders(status);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Graph Queries

### Trace water flow path from reservoir to field

```sql
-- Find all infrastructure and fields downstream of a reservoir
WITH RECURSIVE flow_path AS (
    -- Start from the reservoir node
    SELECT
        gn.id AS node_id,
        gn.node_type,
        gn.label,
        ge.edge_type,
        ge.properties AS edge_props,
        1 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_node_id = gn.id
    WHERE gn.entity_id = 'uuid-reservoir'  -- starting reservoir
      AND gn.node_type = 'reservoir'
      AND ge.is_active = true

    UNION ALL

    -- Follow flow edges downstream
    SELECT
        gn.id,
        gn.node_type,
        gn.label,
        ge.edge_type,
        ge.properties,
        fp.depth + 1,
        fp.path || gn.id
    FROM flow_path fp
    JOIN graph_edges ge ON ge.source_node_id = fp.node_id
    JOIN graph_nodes gn ON gn.id = ge.target_node_id
    WHERE ge.edge_type IN ('flows_to', 'feeds', 'diverts_from')
      AND ge.is_active = true
      AND gn.id != ALL(fp.path)  -- prevent cycles
      AND fp.depth < 20           -- safety limit
)
SELECT node_type, label, depth, edge_type,
       edge_props->>'capacity_cfs' AS capacity_cfs
FROM flow_path
ORDER BY depth;
```

### Impact analysis: what is affected if a canal segment fails?

```sql
-- Find all downstream entities affected by a canal outage
WITH RECURSIVE downstream AS (
    SELECT
        gn.id,
        gn.node_type,
        gn.label,
        gn.entity_id,
        gn.entity_table,
        1 AS depth
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_node_id = (
        SELECT id FROM graph_nodes WHERE entity_id = 'uuid-failed-canal' AND node_type = 'canal'
    )
    WHERE gn.id = ge.target_node_id
      AND ge.edge_type IN ('flows_to', 'feeds')
      AND ge.is_active = true

    UNION ALL

    SELECT
        gn.id,
        gn.node_type,
        gn.label,
        gn.entity_id,
        gn.entity_table,
        d.depth + 1
    FROM downstream d
    JOIN graph_edges ge ON ge.source_node_id = d.id
    JOIN graph_nodes gn ON gn.id = ge.target_node_id
    WHERE ge.edge_type IN ('flows_to', 'feeds')
      AND ge.is_active = true
      AND gn.id NOT IN (SELECT id FROM downstream)
      AND d.depth < 20
)
SELECT
    node_type,
    label,
    depth,
    -- Join to relational tables for operational details
    CASE
        WHEN entity_table = 'fields' THEN (SELECT name FROM fields WHERE id = entity_id)
        WHEN entity_table = 'water_rights' THEN (SELECT right_number FROM water_rights WHERE id = entity_id)
    END AS detail
FROM downstream
ORDER BY depth, node_type;
```

### Water rights conflict detection: shared source analysis

```sql
-- Find all water rights that share the same water source
SELECT
    gn_source.label AS water_source,
    gn_right.label AS water_right,
    wr.right_number,
    wr.priority_date,
    wr.max_volume_af,
    wr.use_category,
    org.name AS organisation
FROM graph_nodes gn_source
JOIN graph_edges ge ON ge.source_node_id = gn_source.id
    AND ge.edge_type = 'diverts_from_source'
JOIN graph_nodes gn_right ON gn_right.id = ge.target_node_id
    AND gn_right.node_type = 'water_right'
JOIN water_rights wr ON wr.id = gn_right.entity_id
JOIN organisations org ON org.id = wr.organisation_id
WHERE gn_source.node_type = 'water_source'
  AND gn_source.label = 'Bear Creek'
ORDER BY wr.priority_date;
```

### Find all sensors monitoring infrastructure along a flow path

```sql
-- Sensors installed on infrastructure in the delivery path to a specific field
WITH RECURSIVE delivery_path AS (
    SELECT gn.id, gn.node_type, gn.label, 1 AS depth, ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = 'uuid-target-field' AND gn.node_type = 'field'

    UNION ALL

    SELECT gn.id, gn.node_type, gn.label, dp.depth + 1, dp.path || gn.id
    FROM delivery_path dp
    JOIN graph_edges ge ON ge.target_node_id = dp.id  -- traverse upstream
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    WHERE ge.edge_type IN ('flows_to', 'feeds')
      AND ge.is_active = true
      AND gn.id != ALL(dp.path)
      AND dp.depth < 20
)
SELECT
    dp.label AS infrastructure,
    dp.node_type,
    s.serial_number,
    s.device_type,
    s.status
FROM delivery_path dp
JOIN graph_edges ge_sensor ON ge_sensor.target_node_id = dp.id
    AND ge_sensor.edge_type = 'installed_at'
JOIN graph_nodes gn_sensor ON gn_sensor.id = ge_sensor.source_node_id
    AND gn_sensor.node_type = 'sensor_device'
JOIN sensors s ON s.id = gn_sensor.entity_id
ORDER BY dp.depth DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Organisation & Auth | 2 | organisations, users |
| Fields & Infrastructure | 2 | fields, infrastructure |
| Sensors & Observations | 2 | sensors, observations |
| Weather & ET | 2 | weather_data, et_calculations |
| Water Rights & Usage | 2 | water_rights, water_usage |
| Irrigation Operations | 2 | irrigation_events, water_orders |
| Alerts & Maintenance | 2 | alerts, work_orders |
| Audit | 1 | audit_log |
| **Total** | **17** | Plus the graph layer provides implicit relationships replacing many junction tables |

---

## Key Design Decisions

1. **Graph nodes reference relational entities** — each graph_node has entity_id and entity_table columns that link back to the corresponding relational record. The graph layer adds relationship navigation; the relational tables remain the source of truth for entity attributes. This avoids data duplication while enabling graph traversal.

2. **Directed, typed edges** — edges have a source, target, and type, enabling directional queries ("what flows downstream from here?") and type-filtered traversals ("follow only 'flows_to' edges"). The edge properties JSONB column carries relationship-specific attributes like pipe capacity or permit number.

3. **Temporal edges with valid_from/valid_to** — infrastructure connections and water rights associations change over time. Temporal validity on edges enables historical network analysis ("what was the flow path in 2020?") without deleting edges.

4. **Weight column for graph algorithms** — the optional weight on edges supports shortest-path, maximum-flow, and minimum-cost algorithms. For water networks, weight can represent pipe length (shortest path), capacity (maximum flow), or friction loss (minimum cost routing).

5. **EPANET topology mapping** — the graph node types (reservoir, junction, pipe, pump, valve, tank) align with EPANET's network model, enabling export of the graph to EPANET INP format for hydraulic simulation. This bridges the gap between the management platform and engineering analysis tools.

6. **Alerts and work orders link to graph nodes** — when a sensor detects an anomaly or an asset needs maintenance, the alert/work order references the graph node, enabling immediate impact analysis ("if this pump fails, which fields lose water?") directly from the alert.

7. **High-volume time-series data stays relational** — observations, weather data, and ET calculations are NOT in the graph. These are append-heavy, range-query-optimised tables that gain nothing from graph representation. The graph connects sensors to infrastructure and fields; the relational tables store what those sensors measure.

8. **Cycle prevention in recursive CTEs** — all graph traversal queries include path tracking (ARRAY of visited node IDs) and depth limits to prevent infinite loops in cyclic infrastructure networks (e.g., recirculation loops).

9. **Graph maintains its own indexes** — GIN indexes on graph_nodes.properties and graph_edges.properties enable filtered traversals ("find all pipes with capacity > 10 CFS in this path") without joining to relational tables during graph walks.

10. **Migration path to dedicated graph database** — if the PostgreSQL graph layer becomes a bottleneck, the graph_nodes and graph_edges tables can be exported to Neo4j (node labels = node_type, relationship types = edge_type, properties = JSONB) with minimal transformation. The relational tables continue to handle CRUD and time-series regardless of graph backend.
