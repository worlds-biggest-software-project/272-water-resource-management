# Water Resource Management — Phased Development Plan

> Project: 272-water-resource-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased build. The canonical schema is **Data Model 1 (Entity-Centric Normalized Relational)** — chosen because this platform must serve regulatory environments (SGMA compliance, water-rights adjudication) where data integrity, auditability, and flexible cross-entity reporting outrank raw write throughput. Where vendor/jurisdiction variance demands flexibility, we selectively adopt **Data Model 3's** JSONB pattern (sensor decoders, rate-schedule tiers, alert-rule conditions). The graph capability from **Data Model 4** (flow tracing, curtailment-impact analysis) is deferred to a late phase as an additive `graph_nodes`/`graph_edges` layer.

---

## Product Summary

**What it does.** An AI-native, open-source platform unifying three tiers that incumbents silo: field-level irrigation/agronomy (CropX, Tule, Phytech), district-level water-rights accounting and billing (WaterMaster), and utility-level asset/operations management (OpenGov). It ingests soil/weather/flow sensor streams, computes FAO-56 reference and crop evapotranspiration, generates irrigation prescriptions, tracks water rights against usage with automated compliance checks, and exposes everything via standards-compliant APIs (OGC SensorThings, OGC API — Features, WaterML 2.0, OpenAPI 3.x).

**Primary users.** Irrigation-district managers, commercial growers, municipal utility operators, state regulators, and supply-chain sustainability officers.

**Key differentiators.** (1) Multi-scale unification (field + district + utility); (2) software-first, hardware-agnostic ingestion (no proprietary sensor lock-in); (3) AI-native ET forecasting, continuous water-rights conflict detection, predictive maintenance, and an LLM natural-language interface; (4) standards-compliant data exchange that wins government/regulatory markets where incumbents use proprietary schemas.

**Deployment.** Self-hosted (Docker Compose) and cloud (managed Postgres + object storage), with a web dashboard and a mobile/offline field client.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The domain is ET modelling, time-series analytics, and ML (anomaly detection, forecasting). Python has FAO-56 reference implementations, `pyeto`, `numpy`/`pandas`/`scikit-learn`, the USGS `dataretrieval` SDK, and EPANET toolkits (`EPyT`, `WNTR`) — all directly relevant. |
| API framework | **FastAPI** | Native OpenAPI 3.1 generation (required by standards.md for all public APIs), Pydantic v2 validation for sensor payloads and JSON Schema, async support for high-volume sensor ingestion, first-class background tasks. |
| Database | **PostgreSQL 16 + PostGIS 3.4 + TimescaleDB** | Data Model 1 mandates PostGIS geometry columns and GIST indexes for field boundaries, points of diversion, and asset paths. TimescaleDB hypertables solve the "observations table grows fastest" partitioning problem from Data Model 1's design notes. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic + GeoAlchemy2** | Mature spatial ORM support, explicit migration control for the 31-table normalized schema, async engine for ingestion paths. |
| Task queue | **Celery + Redis** | Async workloads: LoRaWAN/MQTT decoding, weather/satellite API polling, nightly ET batch computation, compliance scans, AI prescription generation, invoice runs. Redis doubles as cache. |
| Realtime ingestion | **MQTT (Eclipse Mosquitto) + paho-mqtt** | standards.md: MQTT (ISO/IEC 20922) is the standard for SensorThings real-time streaming; LoRaWAN network servers bridge via MQTT. |
| LLM integration | **Anthropic Claude (Messages API) via a provider abstraction** | Powers ET-forecast reasoning, compliance-report drafting, and the SMS/chat NL interface. Abstraction layer (`llm/provider.py`) keeps the platform model-agnostic and offline-deployable. |
| Frontend (web) | **Next.js 15 (App Router) + TypeScript + React** | Dashboard SPA with server components; MapLibre GL for GIS visualisation of GeoJSON field/sensor/rights layers. |
| Mapping | **MapLibre GL JS + OGC API — Features tiles** | Open-source (no Esri licence), renders GeoJSON directly, aligns with the OGC API — Features standard the backend serves. |
| Mobile / offline field client | **React Native (Expo) + WatermelonDB** | Ditch-rider / field-operator workflows require offline-first capture (per WaterMaster RideKick and SafetyCulture). WatermelonDB provides local-first sync. |
| Auth | **OAuth 2.0 / OIDC (Authlib) + JWT sessions** | standards.md requires OAuth 2.0 for multi-party data sharing; matches John Deere, Trimble, Leaf integration auth. RBAC via `roles`/`user_org_roles`. |
| Testing (backend) | **pytest + pytest-asyncio + testcontainers + respx** | testcontainers spins a real Postgres+PostGIS for integration tests; respx mocks external HTTP (weather, OpenET, USGS). |
| Testing (frontend) | **Vitest + Playwright** | Unit + e2e for dashboard flows. |
| Code quality | **ruff (lint+format), mypy (types), pre-commit; eslint+prettier for TS** | Single fast toolchain for Python; standard TS toolchain. |
| Package mgmt | **uv (Python), pnpm (JS)** | Fast, reproducible lockfiles. |
| Containerisation | **Docker + docker-compose** | Self-hosted deployment mode from README; compose wires api, worker, postgres, redis, mosquitto, web. |
| Spatial/standards libs | **Shapely, GeoAlchemy2, pyeto (FAO-56), dataretrieval (USGS), paho-mqtt** | Domain-specific per standards.md and research.md. |

### Project Structure

```
water-resource-management/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── README.md
├── backend/
│   ├── wrm/
│   │   ├── __init__.py
│   │   ├── main.py                 # FastAPI app factory, router mounting
│   │   ├── config.py               # Pydantic Settings (env vars)
│   │   ├── db/
│   │   │   ├── session.py          # async engine, session factory
│   │   │   ├── base.py             # declarative base, mixins (TimestampMixin, OrgScopedMixin)
│   │   │   └── migrations/         # Alembic versions
│   │   ├── models/                 # SQLAlchemy ORM models (one module per domain group)
│   │   │   ├── org.py              # organisations, jurisdictions, users, roles, user_org_roles
│   │   │   ├── fields.py           # fields, irrigation_zones, crops, field_crop_seasons, assets
│   │   │   ├── sensors.py          # sensor_devices, observed_properties, datastreams, observations
│   │   │   ├── weather.py          # weather_stations, weather_observations, et_calculations
│   │   │   ├── rights.py           # water_rights, allocations, usage_records, compliance_checks
│   │   │   ├── operations.py       # irrigation_schedules, water_orders
│   │   │   ├── billing.py          # billing_accounts, rate_schedules, invoices, line_items
│   │   │   ├── maintenance.py      # work_orders
│   │   │   ├── alerts.py           # alerts, alert_rules
│   │   │   └── audit.py            # audit_log
│   │   ├── schemas/                # Pydantic request/response models
│   │   ├── api/                    # FastAPI routers
│   │   │   ├── deps.py             # auth, db session, org-scope dependencies
│   │   │   ├── auth.py
│   │   │   ├── organisations.py
│   │   │   ├── fields.py
│   │   │   ├── sensors.py
│   │   │   ├── observations.py
│   │   │   ├── weather.py
│   │   │   ├── irrigation.py
│   │   │   ├── rights.py
│   │   │   ├── billing.py
│   │   │   ├── maintenance.py
│   │   │   ├── alerts.py
│   │   │   ├── ogc/                # SensorThings + OGC API Features routers
│   │   │   └── reports.py
│   │   ├── services/               # business logic (pure, testable)
│   │   │   ├── et.py               # FAO-56 Penman-Monteith
│   │   │   ├── scheduling.py       # irrigation prescription (rule-based + AI)
│   │   │   ├── compliance.py       # water-rights conflict detection
│   │   │   ├── billing.py          # invoice generation
│   │   │   ├── alerts.py           # rule evaluation
│   │   │   └── anomaly.py          # flow/pressure anomaly detection
│   │   ├── ingest/                 # sensor + external data ingestion
│   │   │   ├── mqtt_listener.py
│   │   │   ├── decoders/           # per-vendor LoRaWAN/Cayenne LPP decoders
│   │   │   └── providers/          # weather, OpenET, USGS, SGMA clients
│   │   ├── llm/
│   │   │   ├── provider.py         # model-agnostic LLM interface
│   │   │   ├── prompts.py          # prompt templates
│   │   │   └── nl_interface.py     # SMS/chat command parsing
│   │   ├── workers/
│   │   │   ├── celery_app.py
│   │   │   └── tasks.py
│   │   └── graph/                  # graph_nodes/graph_edges layer (Phase 10)
│   └── tests/
│       ├── conftest.py             # testcontainers Postgres fixture, factory fixtures
│       ├── unit/
│       ├── integration/
│       └── e2e/
├── web/                            # Next.js dashboard
│   ├── package.json
│   ├── app/
│   ├── components/
│   ├── lib/api/                    # generated client from OpenAPI spec
│   └── tests/
└── mobile/                         # React Native (Expo) field client
    ├── package.json
    └── src/
```

The structure is grouped by concern, not by phase; every phase adds files within these directories without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Database, Auth & Multi-Tenancy

### Purpose
Establish the runnable backbone: a FastAPI app, a PostGIS+TimescaleDB database with the organisation/auth tables migrated, OAuth2/OIDC authentication, RBAC, and org-scoped request dependencies. After this phase, a user can authenticate, belong to an organisation with a role, and every subsequent endpoint can rely on a `current_user` + `current_org` context. This is the smallest foundational increment.

### Tasks

#### 1.1 — Project scaffolding & container stack

**What**: A reproducible dev environment with FastAPI, Postgres+PostGIS+TimescaleDB, Redis, and Mosquitto wired via docker-compose.

**Design**:
- `pyproject.toml` declares dependencies (fastapi, uvicorn, sqlalchemy[asyncio], geoalchemy2, alembic, asyncpg, pydantic-settings, authlib, celery[redis], paho-mqtt, shapely, pyeto, anthropic, pytest, ruff, mypy).
- `config.py`:
```python
class Settings(BaseSettings):
    database_url: str            # postgresql+asyncpg://...
    redis_url: str = "redis://redis:6379/0"
    mqtt_broker_url: str = "mqtt://mosquitto:1883"
    jwt_secret: str
    oidc_issuer: str | None = None
    oidc_client_id: str | None = None
    oidc_client_secret: str | None = None
    llm_api_key: str | None = None
    llm_model: str = "claude-opus-4-8"
    environment: Literal["dev", "test", "prod"] = "dev"
    model_config = SettingsConfigDict(env_file=".env")
```
- `main.py` exposes `create_app()` mounting `/health` and `/api/v1`. `/health` returns `{"status":"ok","db":<bool>,"redis":<bool>}`.
- `docker-compose.yml` services: `db` (image `timescale/timescaledb-ha:pg16` with PostGIS), `redis`, `mosquitto`, `api`, `worker`, `web`. Healthchecks on db/redis.
- `Dockerfile` multi-stage (uv install → slim runtime).

**Testing**:
- `Integration: GET /health → 200, body db=true redis=true` (testcontainers up).
- `Unit: Settings loads from env, missing jwt_secret → ValidationError`.
- `E2E: docker-compose up → api container healthy within 60s`.

#### 1.2 — Database base, migrations & spatial/timeseries extensions

**What**: Alembic baseline migration that enables `postgis`, `timescaledb`, `pgcrypto` (for `gen_random_uuid()`), and declarative base with shared mixins.

**Design**:
- `db/base.py`:
```python
class Base(DeclarativeBase): ...
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
class OrgScopedMixin:
    organisation_id: Mapped[UUID] = mapped_column(ForeignKey("organisations.id", ondelete="CASCADE"), index=True)
```
- Alembic env configured for async engine + GeoAlchemy2 type rendering.
- Baseline migration `0001_extensions` runs `CREATE EXTENSION IF NOT EXISTS postgis; timescaledb; pgcrypto;`.

**Testing**:
- `Integration: alembic upgrade head on fresh db → extensions present (SELECT from pg_extension)`.
- `Integration: alembic downgrade base then upgrade head → idempotent, no error`.

#### 1.3 — Organisation, jurisdiction, user & RBAC models + migration

**What**: Implement the five Organisation & Auth tables from Data Model 1 exactly (`organisations`, `jurisdictions`, `users`, `roles`, `user_org_roles`).

**Design**: Use the DDL from `data-model-suggestion-1.md` verbatim as the migration target. ORM models in `models/org.py`. Seed default roles: `org_admin`, `field_operator`, `ditch_rider`, `regulator`, `viewer`, each with a `permissions` JSON list (e.g. `org_admin` → `["*"]`, `viewer` → `["read:*"]`). `org_type` and `jurisdiction_type` use the CHECK-constrained enums from the DDL. `parent_org_id` supports district→sub-org hierarchies.

**Testing**:
- `Unit: creating organisation with org_type='invalid' → IntegrityError (CHECK)`.
- `Unit: jurisdiction self-reference (parent_id) resolves hierarchy`.
- `Integration: insert org + user + user_org_role → query user's orgs returns the org with role`.
- `Integration: duplicate (user, org, role) → UNIQUE violation`.

#### 1.4 — OAuth2/OIDC authentication & JWT sessions

**What**: Login via OIDC (Authlib) and a local password fallback for self-hosted; issue JWT access tokens carrying `sub`, `org_ids`, and roles.

**Design**:
- `POST /api/v1/auth/login` (local): `{email, password}` → `{access_token, token_type:"bearer", expires_in}`.
- `GET /api/v1/auth/oidc/login` → redirect; `GET /api/v1/auth/oidc/callback` → exchange code, upsert user by `auth_subject`, issue JWT.
- JWT claims: `{sub: user_id, orgs: [{org_id, role}], exp}`. Signed HS256 with `jwt_secret`.
- Passwords (local mode) hashed with `argon2`.

**Testing**:
- `Unit: valid credentials → JWT with correct claims; expired token → 401`.
- `Integration (mocked OIDC via respx): callback with valid code → user upserted, JWT returned`.
- `Integration: login wrong password → 401, no token`.

#### 1.5 — Auth & org-scope dependencies + audit log

**What**: FastAPI dependencies `current_user`, `require_role(...)`, and `current_org` that enforce membership; plus the `audit_log` table and a write helper.

**Design**:
- `deps.py`:
```python
async def current_user(token=Depends(oauth2_scheme), db=Depends(get_db)) -> User: ...
def require_role(*roles: str) -> Callable: ...   # 403 if user lacks role in target org
async def current_org(org_id: UUID, user=Depends(current_user)) -> Organisation: ...
```
- `audit_log` table per Data Model 1. `services/audit.py: record(action, entity_type, entity_id, old, new, request)` called from mutating endpoints.
- Org isolation: queries on org-scoped tables always filter `organisation_id == current_org.id`.

**Testing**:
- `Unit: require_role('org_admin') with viewer token → 403`.
- `Integration: user A cannot read org B's organisation record → 404/403`.
- `Integration: updating a record writes an audit_log row with old/new JSON`.

---

## Phase 2: Spatial Core — Fields, Zones, Crops & Infrastructure

### Purpose
Add the physical and agronomic backbone: field boundaries (PostGIS polygons), irrigation zones, crops with FAO-56 coefficients, crop seasons, and infrastructure assets. After this phase the platform models *what is being irrigated and with what equipment*, and can serve/accept GeoJSON. This is the spatial foundation every later tier (sensors, rights, scheduling) attaches to.

### Tasks

#### 2.1 — Fields & irrigation zones (PostGIS + GeoJSON I/O)

**What**: CRUD for `fields` and `irrigation_zones` with GeoJSON in/out and computed area.

**Design**: DDL from Data Model 1 (`fields`, `irrigation_zones`). Pydantic schemas accept/emit GeoJSON (RFC 7946) via Shapely⇄GeoAlchemy2 conversion. `area_hectares` auto-computed from boundary using `ST_Area(geography)` on insert/update.
- `POST /api/v1/orgs/{org_id}/fields` body: `{name, boundary: <GeoJSON Polygon>, soil_type?, elevation_m?}` → field with computed `area_hectares`.
- `GET .../fields?bbox=minx,miny,maxx,maxy` → GeoJSON FeatureCollection (spatial filter via `ST_Intersects`).

**Testing**:
- `Unit: GeoJSON Polygon → Shapely → WKB round-trips unchanged`.
- `Integration: create field with 1 km² boundary → area_hectares ≈ 100 (±0.5%)`.
- `Integration: bbox query returns only intersecting fields`.
- `Unit: invalid GeoJSON (open ring) → 422 with descriptive error`.

#### 2.2 — Crops & crop seasons (FAO-56 coefficients)

**What**: Crop catalogue with `kc_ini/kc_mid/kc_end`, root depth, growing days; per-field/zone planting seasons with growth-stage tracking.

**Design**: DDL for `crops` and `field_crop_seasons`. Seed a starter crop library (alfalfa, almond, corn, grape, tomato, wheat) with published FAO-56 Kc values. `growth_stage` derived helper computes the current stage from `planting_date`, `growing_days`, and today.
```python
def current_growth_stage(planting_date: date, growing_days: int, today: date) -> Literal["initial","development","mid_season","late_season"]
def current_kc(crop: Crop, stage: str) -> float   # interpolates between kc_ini/mid/end
```

**Testing**:
- `Unit: day 5 of 120-day corn → 'initial'; day 70 → 'mid_season'`.
- `Unit: current_kc interpolates linearly in development stage`.
- `Integration: assign crop season to field → query returns active season for season_year`.

#### 2.3 — Infrastructure assets

**What**: CRUD for `infrastructure_assets` (pipe, pump, valve, canal, reservoir, well, meter, gate) with point/line geometry and JSONB `specifications`.

**Design**: DDL from Data Model 1. Linear assets (pipe/canal) use `path` LineString; point assets use `location`. `specifications` JSONB validated per asset_type by a Pydantic discriminated union (e.g. pipe → `{diameter_mm, material, capacity_lps}`; pump → `{power_kw, max_flow_lps}`).

**Testing**:
- `Unit: pipe spec missing diameter_mm → 422`.
- `Integration: create canal with LineString path → GIST index used in proximity query`.
- `Integration: assets filtered by asset_type and bbox`.

---

## Phase 3: Sensor Ingestion & Observations (OGC SensorThings)

### Purpose
The data heart of the field tier. Implement the OGC SensorThings-aligned `sensor_devices`/`observed_properties`/`datastreams`/`observations` model, a TimescaleDB hypertable for observations, MQTT ingestion with pluggable per-vendor decoders, and sensor-health tracking. After this phase the platform continuously ingests soil/weather/flow data and can answer "what is field X's soil moisture right now and over time" — the table-stakes capability every competitor has.

### Tasks

#### 3.1 — Sensor & observation schema + hypertable

**What**: Implement the four sensor tables; convert `observations` to a TimescaleDB hypertable partitioned by `phenomenon_time`.

**Design**: DDL from Data Model 1 for `sensor_devices`, `observed_properties`, `datastreams`, `observations`. Migration calls `SELECT create_hypertable('observations','phenomenon_time', chunk_time_interval => INTERVAL '7 days')` and adds a compression policy after 30 days. Seed `observed_properties` with the standard set: `soil_moisture_vwc (m3/m3)`, `soil_temp (°C)`, `soil_ec (dS/m)`, `air_temp (°C)`, `humidity (%)`, `wind_speed (m/s)`, `solar_rad (MJ/m2)`, `flow_rate (L/s)`, `pressure (kPa)`, `canopy_temp (°C)`, `water_level (m)`.

**Testing**:
- `Integration: observations is a hypertable (timescaledb_information.hypertables)`.
- `Integration: insert 10k observations across 3 weeks → spread over ≥3 chunks`.
- `Unit: duplicate (sensor_id, observed_property_id) datastream → UNIQUE violation`.

#### 3.2 — Observation write API & query API

**What**: Endpoints to push observations and query time-series with aggregation.

**Design**:
- `POST /api/v1/datastreams/{id}/observations` body: `{phenomenon_time, result_value, quality_code?}` (single or batch array). Validates quality_code ∈ WaterML 2.0 set `good|suspect|missing|estimated`.
- `GET /api/v1/datastreams/{id}/observations?start=&end=&interval=1h&agg=mean` → uses TimescaleDB `time_bucket`. `agg ∈ {mean,min,max,sum,last}`.
- Latest-value endpoint `GET /api/v1/sensors/{id}/latest` returns last observation per datastream.

**Testing**:
- `Integration: batch insert 1000 obs → all stored, returns count`.
- `Integration: hourly mean aggregation over a day → 24 buckets with correct means`.
- `Unit: quality_code='bogus' → 422`.

#### 3.3 — MQTT listener & pluggable vendor decoders

**What**: A long-running MQTT subscriber that decodes vendor payloads (Cayenne LPP + per-vendor binary) into observations, keeping the raw payload in JSONB (Data Model 3 pattern).

**Design**:
- `ingest/mqtt_listener.py` subscribes to `sensors/+/up`. Topic carries device EUI; payload is base64/hex.
- Decoder registry:
```python
class SensorDecoder(Protocol):
    vendor: str
    def decode(self, raw: bytes, device: SensorDevice) -> list[DecodedReading]: ...

@dataclass
class DecodedReading:
    observed_property: str
    value: float
    phenomenon_time: datetime
    quality_code: str = "good"
```
- Implement `CayenneLPPDecoder` and one example vendor binary decoder. Raw payloads stored in `sensor_devices.metadata->'last_raw'` and (optionally) a `raw_payloads` audit table for reprocessing. Updates `last_seen_at` and `battery_pct`.
- Unknown device EUI → log + dead-letter, do not crash.

**Testing**:
- `Unit: CayenneLPP frame for temp+humidity → two DecodedReadings with correct units`.
- `Unit: unknown EUI → dead-lettered, listener continues`.
- `Integration (embedded MQTT): publish frame → observation row created, last_seen_at updated`.

#### 3.4 — Sensor-health monitoring

**What**: Track connectivity/battery and surface offline/low-battery state.

**Design**: A Celery beat task `check_sensor_health` (every 15 min) marks `status='offline'` where `last_seen_at < now() - expected_interval` and emits an alert (consumed in Phase 7). `GET /api/v1/orgs/{org_id}/sensors/health` returns counts by status + battery distribution.

**Testing**:
- `Unit: sensor with last_seen 2h ago, 1h interval → flagged offline`.
- `Integration: health endpoint returns correct online/offline/low-battery counts`.

---

## Phase 4: Weather, Evapotranspiration & FAO-56 Engine

### Purpose
Implement the agronomic compute core: weather ingestion (on-site + external APIs), the FAO-56 Penman-Monteith reference ET calculation, crop ET (ETc = ETo × Kc), and OpenET satellite integration. This is the first piece of genuine domain value and the input to irrigation scheduling. After this phase the platform produces daily ETo/ETc per field.

### Tasks

#### 4.1 — Weather stations & observations

**What**: `weather_stations` and `weather_observations` tables with the exact FAO-56 input columns.

**Design**: DDL from Data Model 1 (note the explicit `temp_max_c, temp_min_c, temp_mean_c, humidity_pct, wind_speed_ms, solar_rad_mjm2, precipitation_mm, pressure_kpa` columns — designed for Penman-Monteith). CRUD plus a `nearest_station(field)` helper using `ST_Distance`.

**Testing**:
- `Integration: nearest_station returns closest by geography distance`.
- `Unit: weather obs with all FAO-56 fields persists and reads back`.

#### 4.2 — FAO-56 Penman-Monteith ETo service

**What**: Pure-function reference ET calculation following FAO Irrigation & Drainage Paper 56.

**Design**: `services/et.py` (uses `pyeto` where suitable, with explicit implementation for auditability):
```python
def eto_penman_monteith(
    temp_max_c: float, temp_min_c: float, humidity_pct: float,
    wind_speed_ms: float, solar_rad_mjm2: float,
    latitude_deg: float, elevation_m: float, day_of_year: int,
) -> float:   # returns ETo in mm/day
```
Implements: saturation vapour pressure, slope of vapour curve (Δ), psychrometric constant (γ), net radiation (Rn from solar + extraterrestrial Ra), and the combined FAO-56 equation. Wind adjusted to 2 m height. Returns mm/day.

**Testing**:
- `Unit: FAO-56 Example 18 (Allen et al. 1998) inputs → ETo within ±0.05 mm of published 3.9 mm/day`.
- `Unit: zero wind / extreme inputs → no NaN, bounded output`.
- `Property: ETo monotonically increases with solar radiation, all else equal`.

#### 4.3 — Crop ET & daily batch + et_calculations

**What**: Compute ETc per field and persist to `et_calculations`; a nightly Celery task runs all active fields.

**Design**: DDL for `et_calculations`. `services/et.py: etc(eto, kc) -> float`. Daily task `compute_daily_et`: for each field with an active crop season, fetch nearest-station weather (or OpenET), compute ETo, derive Kc from growth stage (Phase 2.2), store `et_reference_mm`, `crop_coefficient`, `et_crop_mm`, `method='fao56_penman_monteith'`, source station, and optional `ndvi_value`. Stores `confidence` (lower when interpolating missing inputs).

**Testing**:
- `Integration: field with corn at mid-season + weather → et_calculations row with ETc = ETo × kc_mid`.
- `Integration: batch over 50 fields completes; fields lacking weather get confidence<1 and a flag`.

#### 4.4 — External provider clients (weather + OpenET + USGS)

**What**: Pluggable clients for OpenET (satellite actual ET), a weather forecast API, and USGS Water Data (OGC API — Features) for streamflow/groundwater context.

**Design**: `ingest/providers/` with a common interface:
```python
class ETProvider(Protocol):
    async def actual_et(self, geometry: dict, start: date, end: date) -> list[ETPoint]: ...
```
- `OpenETClient` (API key, GeoJSON field query, daily/monthly products) per standards.md.
- `USGSClient` wraps `dataretrieval` / OGC API — Features.
- Provider results stored as `et_calculations` rows with `source_satellite='openet_ensemble'`. Retry with backoff; rate-limit aware.

**Testing**:
- `Integration (respx mock OpenET): field GeoJSON → ETPoints parsed and stored`.
- `Unit: provider 429 → retried with backoff, then surfaced after N attempts`.

---

## Phase 5: Irrigation Scheduling & Prescriptions (rule-based core, AI-ready)

### Purpose
Convert ET + soil + weather into actionable irrigation prescriptions — the MVP "basic irrigation scheduling" feature and the seam where AI plugs in (v1.1). Implements a soil-water-balance model, rule-based scheduling, multi-zone support, and the `irrigation_schedules` lifecycle. After this phase operators receive recommended watering amount and timing per field/zone.

### Tasks

#### 5.1 — Soil water balance & rule-based scheduler

**What**: A daily soil-water-balance model that recommends irrigation depth/timing.

**Design**: `services/scheduling.py`:
```python
@dataclass
class WaterBalanceState:
    available_water_mm: float       # current soil water in root zone
    field_capacity_mm: float
    mad_fraction: float = 0.5       # management allowable depletion
    def deficit_mm(self) -> float: ...

def recommend_irrigation(field, season, etc_mm, rain_mm, soil_obs) -> IrrigationRecommendation
```
Logic: update balance with ETc (out) and rain + last irrigation (in); when depletion ≥ MAD × total available water, recommend refill depth = deficit; pick timing avoiding forecast rain. Output includes `volume_target_mm`, `scheduled_start`, `duration_minutes` (from zone flow rate), and human-readable `reasoning`.

**Testing**:
- `Unit: depletion below MAD → no irrigation recommended`.
- `Unit: depletion at MAD with no forecast rain → recommends depth == deficit`.
- `Unit: forecast rain ≥ deficit within 24h → defers irrigation`.

#### 5.2 — irrigation_schedules lifecycle & multi-zone

**What**: Persist schedules with state machine and approval; support per-zone schedules for complex layouts.

**Design**: DDL for `irrigation_schedules`. State machine: `scheduled → approved → in_progress → completed` (+ `cancelled`). `schedule_type ∈ {ai_recommendation, manual, rule_based, calendar}`. Endpoints:
- `POST .../fields/{id}/schedules/generate` → runs scheduler for field (all zones), creates `rule_based` schedules in `scheduled`.
- `POST .../schedules/{id}/approve` (requires `field_operator`+) → `approved`.
- `PATCH .../schedules/{id}/status` enforces legal transitions.

**Testing**:
- `Unit: illegal transition completed→scheduled → 409`.
- `Integration: generate for 3-zone field → 3 schedules`.
- `Integration: approve writes approved_by + audit_log`.

#### 5.3 — AI prescription generation (LLM-assisted)

**What**: An AI path that produces prescriptions enriched with crop stage, disease/salinity risk, and reasoning, with confidence scoring.

**Design**: `services/scheduling.py: ai_recommend(...)` calls `llm/provider.py`. Prompt template in `llm/prompts.py`:
```
System: You are an irrigation agronomist. Given field, crop stage, recent ETc,
soil moisture trend, weather forecast, and water-rights remaining allocation,
recommend watering depth (mm) and timing. Respond as JSON:
{depth_mm, start_iso, reasoning, confidence_0_1, risks:[...]}.
Never recommend exceeding remaining allocation.
User: <structured field/crop/weather/soil/allocation context JSON>
```
Output validated against a Pydantic schema; stored as `schedule_type='ai_recommendation'` with `ai_confidence` and `reasoning`. Falls back to rule-based scheduler if LLM unavailable.

**Testing**:
- `Integration (mocked LLM): valid JSON response → ai_recommendation schedule created`.
- `Unit: LLM returns depth exceeding remaining allocation → clamped + risk flag added`.
- `Unit: LLM unreachable → falls back to rule_based, no error`.

---

## Phase 6: Water Rights, Allocations, Usage & Compliance

### Purpose
The district/regulatory tier — the strongest differentiator versus field-only competitors. Implements HarDWR-aligned water rights, annual allocations, metered usage, and an automated compliance-check engine (the AI-native "continuous conflict detection"). After this phase the platform tracks legal entitlements against actual usage and flags violations.

### Tasks

#### 6.1 — Water rights & allocations (HarDWR-aligned)

**What**: Implement `water_rights`, `water_allocations`, `water_usage_records` from Data Model 1.

**Design**: DDL verbatim — `priority_date`, `use_category`, `water_source`, `source_type`, `max_flow_cfs`, `max_volume_af`, `point_of_diversion` (PostGIS), `place_of_use` align with HarDWR's harmonised fields across 11 Western US states. `water_allocations.remaining_af` is the stored generated column. Usage records link allocation↔field↔meter.

**Testing**:
- `Unit: right_type='invalid' → CHECK violation`.
- `Integration: insert allocation 1000 AF, usage records summing 300 AF rolled into used_volume → remaining_af = 700`.
- `Integration: point_of_diversion spatial query within jurisdiction boundary`.

#### 6.2 — Usage rollup & priority-date conflict ordering

**What**: Aggregate usage into allocation balances and order rights by priority for curtailment scenarios.

**Design**: `services/compliance.py: rollup_usage(allocation_year)` sums `water_usage_records.volume_af` into `water_allocations.used_volume_af`. `prior_appropriation_order(jurisdiction)` returns rights sorted by `priority_date` (first in time = first in right) for curtailment lists.

**Testing**:
- `Unit: rollup sums only the target year's usage`.
- `Unit: priority ordering — earliest priority_date ranked senior`.

#### 6.3 — Automated compliance-check engine

**What**: A scheduled scan that compares usage to entitlements and writes `compliance_checks` rows; AI-drafted explanations.

**Design**: DDL for `compliance_checks`. Celery beat `run_compliance_scan` (daily) per active right evaluates: `usage_limit` (used_volume_af vs max_volume_af), `flow_rate` (peak flow vs max_flow_cfs), `seasonal_restriction`, `curtailment`. Status ∈ `compliant|warning|violation|pending`; `warning` at ≥90% of limit, `violation` at >100%. `auto_generated=true`. For violations, `llm/provider.py` drafts a plain-language `details` explanation. Emits an alert (Phase 7).

**Testing**:
- `Unit: used 95% of limit → 'warning'; 105% → 'violation'`.
- `Integration: scan over rights produces correct statuses; violations create alerts`.
- `Integration (mocked LLM): violation → details populated`.

#### 6.4 — Water orders (field-to-office workflow)

**What**: `water_orders` table and ditch-rider assignment workflow (WaterMaster RideKick parity).

**Design**: DDL for `water_orders`. Lifecycle `pending → approved → delivering → completed` (+`cancelled`). Endpoints to create order (grower), approve + assign rider (district admin), and rider status updates. Order volume validated against remaining allocation.

**Testing**:
- `Unit: order volume > remaining allocation → 422`.
- `Integration: assign rider → status approved, assigned_rider set`.

---

## Phase 7: Alerts, Notifications & Rule Engine

### Purpose
Cross-cutting alerting consumed by sensors (Phase 3), compliance (Phase 6), weather (frost), and AI models. Implements `alerts`/`alert_rules`, a configurable rule evaluator, and notification delivery (email/SMS/in-app). After this phase the platform proactively surfaces conditions needing attention.

### Tasks

#### 7.1 — Alerts & alert rules

**What**: Implement `alerts` and `alert_rules` with a JSONB condition evaluator (Data Model 3 pattern for conditions).

**Design**: DDL from Data Model 1. `alert_rules.condition` JSONB e.g. `{"property":"soil_moisture_vwc","operator":"lt","value":0.15,"duration_minutes":60}`. `services/alerts.py: evaluate_rules(org)` checks recent observations/compliance against active rules; creates `alerts` (deduped by source_type+source_id+open state). Polymorphic `source_type`+`source_id`.

**Testing**:
- `Unit: soil_moisture 0.12 < 0.15 for 60min → alert created`.
- `Unit: condition met once but not for duration → no alert`.
- `Integration: duplicate condition while alert open → not re-created`.

#### 7.2 — Notification delivery & acknowledgement

**What**: Deliver alerts to roles via channels; acknowledge/resolve workflow.

**Design**: `notify_roles` JSON drives recipients. Channel adapters: `EmailChannel`, `SMSChannel` (Twilio-style), `InAppChannel`. `POST /api/v1/alerts/{id}/ack` and `/resolve` set `acknowledged_by`/`resolved_at`. `GET .../alerts?status=unresolved`.

**Testing**:
- `Integration (mocked SMS/email): critical alert → recipients per notify_roles`.
- `Integration: ack sets acknowledged_by; resolve sets resolved_at`.

---

## Phase 8: Standards-Compliant APIs (OGC SensorThings, OGC API — Features, OpenAPI)

### Purpose
Expose the data via the OGC standards from standards.md — the differentiator for government/regulatory markets where incumbents use proprietary schemas. After this phase external systems and an MCP server can consume sensor and spatial data through standard interfaces.

### Tasks

#### 8.1 — OGC SensorThings API (read)

**What**: SensorThings-compliant endpoints projecting the existing entities.

**Design**: `api/ogc/sensorthings.py`. Map `sensor_devices→Things/Sensors`, `datastreams→Datastreams`, `observations→Observations`, `observed_properties→ObservedProperties`, locations→`Locations`. Support OData query options `$filter`, `$select`, `$expand`, `$top/$skip`, `$resultFormat=GeoJSON`. Base path `/api/v1/ogc/v1.1/`.

**Testing**:
- `Integration: GET /Datastreams(<id>)/Observations?$top=10 → SensorThings JSON`.
- `Integration: $filter=phenomenonTime gt <iso> → filtered set`.
- `Conformance: response validates against SensorThings JSON shape fixtures`.

#### 8.2 — OGC API — Features (field/sensor/rights spatial layers)

**What**: Serve fields, sensor locations, and water-rights places-of-use as OGC API — Features collections.

**Design**: `api/ogc/features.py`. Collections: `fields`, `sensors`, `water_rights`. Endpoints `/collections`, `/collections/{id}/items` (GeoJSON, bbox + property filters), `/collections/{id}/items/{fid}`. Aligns with USGS NWIS migration noted in standards.md.

**Testing**:
- `Integration: /collections lists three collections`.
- `Integration: items?bbox= returns GeoJSON FeatureCollection within bbox`.

#### 8.3 — OpenAPI 3.1 spec & generated clients

**What**: Ensure the full API publishes a clean OpenAPI 3.1 doc; generate the web TS client.

**Design**: FastAPI auto-spec at `/openapi.json`; tag/operation-id hygiene pass. `web/lib/api` generated via `openapi-typescript`. Document OAuth2 security scheme.

**Testing**:
- `Integration: /openapi.json validates as OpenAPI 3.1`.
- `CI: client generation succeeds; type-checks`.

---

## Phase 9: Billing, Maintenance & Reporting

### Purpose
Complete the district/utility tier: volumetric/tiered billing, predictive-ready work orders, and regulatory/operational report generation. After this phase the platform can invoice water usage and produce compliance reports — closing the WaterMaster/OpenGov feature gap.

### Tasks

#### 9.1 — Billing (rate schedules, invoices)

**What**: Implement `billing_accounts`, `rate_schedules`, `invoices`, `invoice_line_items` with tiered-rate computation.

**Design**: DDL from Data Model 1. `rate_schedules.tiers` JSONB (tiered pricing). `services/billing.py: generate_invoice(account, period)` aggregates usage (from `water_usage_records`/allocations), applies the rate schedule (flat/tiered/seasonal/volumetric), writes invoice + line items. Invoice lifecycle `draft→issued→paid|overdue|cancelled`.

**Testing**:
- `Unit: tiered rate 0-100 AF @ $25, 100+ @ $35; 150 AF usage → $2500 + $1750 = $4250`.
- `Integration: generate_invoice → invoice + line items totalling subtotal+tax`.

#### 9.2 — Work orders & predictive-maintenance hooks

**What**: `work_orders` CRUD plus an anomaly-detection service flagging assets for maintenance.

**Design**: DDL for `work_orders`. `services/anomaly.py` runs statistical/ML anomaly detection (e.g. rolling-z-score / IsolationForest) on flow/pressure datastreams; sustained anomalies create `ai_generated=true` work orders + alerts (predictive maintenance from research.md).

**Testing**:
- `Unit: injected flow spike beyond N sigma → anomaly flagged`.
- `Integration: sustained pressure anomaly → ai_generated work order created`.

#### 9.3 — Report generation (compliance & irrigation)

**What**: Generate regulatory and operational reports (PDF/CSV) and a WaterML 2.0 export.

**Design**: `api/reports.py`. Reports: `water_rights_compliance` (usage vs allocation per right, statuses), `irrigation_summary` (ETc, applied, schedules per field/season), `usage_export` (CSV + WaterML 2.0 JSON binding for regulators). Async generation via Celery → object storage; `GET .../reports/{id}` returns status/download URL.

**Testing**:
- `Integration: compliance report for org → PDF + CSV with correct totals`.
- `Integration: usage_export produces WaterML 2.0-shaped JSON validating against fixture schema`.

---

## Phase 10: AI-Native Differentiators — NL Interface, ET Forecasting, Graph Layer & MCP

### Purpose
Ship the headline AI-native capabilities that no incumbent unifies: a natural-language SMS/chat interface, hyper-local ET forecasting, infrastructure-graph impact analysis (Data Model 4), and an MCP server exposing the platform to LLM agents. After this phase the platform delivers its full strategic promise.

### Tasks

#### 10.1 — Natural-language interface (SMS/chat)

**What**: Let operators query status and issue irrigation commands via SMS/chat in natural language.

**Design**: `llm/nl_interface.py`. Inbound SMS webhook → LLM with tool-calling over a constrained toolset: `get_field_status(field)`, `get_soil_moisture(field)`, `create_irrigation_schedule(field, depth, when)`, `get_remaining_allocation(right)`. Tool calls execute through existing services with the sender's RBAC. Replies are concise SMS-formatted text. All actions audit-logged; write actions require a confirmation reply.

**Testing**:
- `Integration (mocked LLM+SMS): "how wet is North field?" → tool get_soil_moisture → SMS reply with value`.
- `Unit: write command without confirmation → asks for confirmation, no schedule created`.
- `Unit: sender lacking role → action refused`.

#### 10.2 — Hyper-local ET forecasting

**What**: Forecast field-by-field ETc several days ahead by fusing forecast weather, satellite NDVI/OpenET history, and soil-sensor trends.

**Design**: `services/et.py: forecast_etc(field, horizon_days)` builds a feature set (forecast weather, recent ETo, NDVI, soil-moisture trend) and produces daily ETc forecasts with uncertainty. Baseline: FAO-56 on forecast weather × Kc; enhanced: gradient-boosted residual model trained on historical actual-vs-FAO ET. Forecasts feed the scheduler (Phase 5) for proactive timing.

**Testing**:
- `Unit: forecast horizon 5 → 5 daily ETc values with confidence intervals`.
- `Integration: forecast feeds scheduler producing pre-emptive schedule before forecast deficit`.

#### 10.3 — Graph layer for flow tracing & curtailment impact

**What**: Add the `graph_nodes`/`graph_edges` layer (Data Model 4) for infrastructure-network and water-source relationship queries.

**Design**: Generic `graph_nodes(id, node_type, ref_id, properties JSONB, geom)` and `graph_edges(id, from_node, to_node, edge_type, properties)`. Sync infrastructure assets, fields, water sources, and rights into the graph. Recursive-CTE traversals: `downstream_fields(asset)`, `curtailment_impact(water_right)`, `shared_source_conflicts(source)`. Maps cleanly to Neo4j if outgrown.

**Testing**:
- `Unit: recursive CTE returns all fields downstream of a canal node`.
- `Integration: curtailing a senior right lists junior rights/fields affected via shared source`.

#### 10.4 — MCP server for water-domain agents

**What**: An MCP server exposing read tools over sensor data, water rights, and ET — the open-source gap noted in standards.md.

**Design**: `backend/wrm/mcp/` implements an MCP server (Python SDK) with tools: `query_observations`, `get_field_status`, `list_water_rights`, `get_compliance_status`, `get_et_forecast`. Auth via API key mapped to a service user + org scope. Read-only by default.

**Testing**:
- `Integration: MCP tool query_observations returns time-series for a datastream`.
- `Unit: MCP request without valid key → unauthorized`.

---

## Phase 11: Web Dashboard & Mobile Field Client

### Purpose
Deliver the human-facing surfaces: a GIS-centric web dashboard for managers/regulators and an offline-first mobile client for field operators and ditch riders. After this phase the platform is usable end-to-end without API calls.

### Tasks

#### 11.1 — Web dashboard (Next.js + MapLibre)

**What**: Authenticated dashboard with map, field status, sensor charts, schedules, rights/compliance, alerts, and billing.

**Design**: Next.js App Router; auth via OIDC/JWT. Views: Map (MapLibre rendering OGC API — Features layers: fields, sensors, rights), Field detail (soil-moisture/ETc charts from observations API), Irrigation (review/approve schedules), Rights & Compliance (status table + violations), Alerts, Billing. Uses the generated OpenAPI client.

**Testing**:
- `E2E (Playwright): login → map renders fields → open field → chart loads`.
- `E2E: approve a schedule → status updates`.
- `Component: compliance table renders violation badges`.

#### 11.2 — Mobile field client (Expo + offline sync)

**What**: Offline-capable app for field status review, water-order updates (ditch rider), and data capture.

**Design**: Expo + WatermelonDB local store; sync queue posts to API on connectivity. Screens: assigned water orders, order status update, field status, photo/note capture, offline observation entry. Conflict resolution: last-write-wins with server timestamp + audit trail.

**Testing**:
- `E2E: offline order status change → queued → synced on reconnect`.
- `Unit: sync conflict resolves to server-authoritative with audit entry`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, db, RBAC)           ─── required by everything
    │
Phase 2: Spatial Core (fields/zones/crops)     ─── requires 1
    │
    ├── Phase 3: Sensor Ingestion              ─── requires 2
    │       │
    │       └── Phase 4: Weather & ET (FAO-56) ─── requires 3 (soil) + 2 (crops)
    │               │
    │               └── Phase 5: Scheduling    ─── requires 4
    │
    ├── Phase 6: Water Rights & Compliance     ─── requires 2 (can parallel with 3-5)
    │
    └── Phase 7: Alerts & Rule Engine          ─── requires 3 + 6 (consumes their events)

Phase 8: Standards APIs (OGC/OpenAPI)          ─── requires 3 + 2 + 6
Phase 9: Billing/Maintenance/Reporting         ─── requires 6 (billing) + 3 (anomaly) 
Phase 10: AI Differentiators                   ─── requires 4,5,6 (NL/ET/graph/MCP)
Phase 11: Web + Mobile                          ─── requires 8 (APIs) ; mobile needs 6 (orders)
```

**Parallelism opportunities:**
- After **Phase 2**, the field/agronomy track (3 → 4 → 5) and the rights/billing track (6 → 9) can be developed concurrently by separate contributors.
- **Phase 7 (Alerts)** can begin as soon as Phase 3 lands and be extended when Phase 6 lands.
- **Phase 8 (Standards APIs)** can start once Phase 3 entities exist; its Features collections grow as 6 lands.
- **Phase 11 (Web/Mobile)** front-end scaffolding can begin against the OpenAPI spec from Phase 8 in parallel with Phase 9/10 backend work.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented per their Design sections.
2. All unit and integration tests pass (`pytest`); new tests cover happy-path and edge cases listed.
3. Linting and formatting pass (`ruff check`, `ruff format --check`; `eslint`/`prettier` for TS).
4. Type checking passes (`mypy backend/`; `tsc --noEmit` for web/mobile).
5. Alembic migrations created, and `upgrade head` / `downgrade` run cleanly on a fresh DB.
6. Docker build succeeds and `docker-compose up` brings the affected services healthy.
7. The phase's headline capability works end-to-end (demonstrated by an integration or e2e test).
8. New endpoints appear in the auto-generated OpenAPI 3.1 spec with correct schemas and security.
9. New config/env options added to `.env.example` and documented in the README.
10. Standards conformance verified where applicable (FAO-56 reference values, SensorThings/OGC API — Features/WaterML 2.0 fixture validation, GeoJSON RFC 7946, OAuth2).
11. Any new external-provider calls are mocked in tests (respx) and degrade gracefully when unavailable.
```
