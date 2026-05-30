# Industrial IoT Edge Platform — Phased Development Plan

> Project: `452-industrial-iot-edge-platform` · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan describes a vendor-agnostic industrial edge platform with two cooperating tiers:

- **Edge agent** — a lightweight runtime deployed on ruggedised ARM/x86 gateways. It speaks native OT protocols (Modbus, OPC-UA, MQTT, DNP3, BACnet), filters/aggregates telemetry locally, evaluates alert rules, runs ML inference, buffers data offline, and syncs to the cloud over mutual TLS.
- **Cloud control plane** — a multi-tenant fleet management, ingestion, configuration, OTA, and dashboard service.

The architectural through-line is **protocol heterogeneity** and **offline resilience**: the edge must work autonomously during WAN outages and reconcile on reconnect, and the data model must absorb wildly different device shapes without schema churn.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Edge agent language | **Go 1.23+** | Single static binary, cross-compiles to `arm64`/`armhf`/`amd64`, runs in <30 MB RAM (well under the 128 MB target — Eclipse Kura needs 512 MB on the JVM). Excellent concurrency for many simultaneous protocol pollers. Mature OT protocol libraries (`gopcua`, `goburrow/modbus`, `eclipse/paho.mqtt.golang`). |
| Cloud control-plane language | **Python 3.12 + FastAPI** | FastAPI auto-generates the OpenAPI 3.1 spec (a standards requirement), Pydantic gives strict request/response validation, and Python is the natural home for the ML training/packaging tooling and analytics. |
| Edge↔cloud transport | **gRPC over mutual TLS (HTTP/2)** + **MQTT 5** fallback | gRPC gives a typed, streaming, bidirectional channel for telemetry sync and command push with built-in flow control. MQTT 5 is offered for cloud-agnostic brokers (AWS IoT Core, Azure IoT Hub, Mosquitto) per the v1.1 requirement. mTLS satisfies IEC 62443 secure-comms. |
| Cloud database | **PostgreSQL 16 + TimescaleDB** | System-of-record for fleet/devices/config/users uses relational tables with **JSONB** for protocol-specific heterogeneity (Data Model Suggestion 3). Telemetry lives in TimescaleDB hypertables with compression + continuous aggregates. One engine, two workloads. |
| Edge-local store | **SQLite 3 (WAL mode) + JSON1** | Zero-process embedded DB for the store-and-forward buffer and last-known-good config cache. Handles a single gateway's write volume and runs on 128 MB hardware. |
| Cloud task/async | **Redis + ARQ (async Redis queue)** | Drives OTA rollout orchestration, certificate rotation jobs, aggregate materialisation triggers, and notification dispatch. Lighter than Celery for an async FastAPI stack. |
| Object storage | **S3-compatible (MinIO self-hosted / any S3)** | Stores OTA release artifacts and ML model files (referenced by URL + SHA-256 in Postgres). Vendor-agnostic via the S3 API. |
| Edge ML runtime | **ONNX Runtime (Go bindings) + TFLite** | ONNX is the portable model format; TFLite covers quantised models for the most constrained gateways. Anomaly/predictive-maintenance models are trained off-platform and shipped as artifacts. |
| Web dashboard | **React 18 + TypeScript + Vite**, **TanStack Query**, **shadcn/ui + Tailwind**, **React Flow** | SPA dashboard. React Flow powers the visual dataflow pipeline editor (the Kura Wire differentiator). TanStack Query handles polling/real-time refresh. |
| Charts | **uPlot** | Renders millions of telemetry points with minimal CPU — important for OT trend views. |
| Edge config format | **YAML (parsed to Go structs) + JSON Schema validation** | Human-editable gateway config, validated against a published JSON Schema so the cloud and edge agree on shape. |
| Telemetry wire format | **Protobuf (gRPC) + InfluxDB Line Protocol export** | Protobuf for the sync channel; Line Protocol export for interop with Telegraf/Grafana ecosystems per standards.md. |
| Containerisation | **Docker + docker-compose (dev)**, **Helm chart (prod)** | Cloud plane is containerised. Edge agent ships as a static binary (systemd unit) OR an optional Docker image for capable gateways. |
| Edge update mechanism | **A/B partition swap + signed artifacts** | OTA with atomic switch and automatic rollback on failed health check, mirroring cloud-native CI/CD reliability. |
| Auth | **OAuth2 password + JWT (cloud users)**, **X.509 client certs (gateways)**, **API keys (programmatic)** | Distinct identity models for humans, machines, and integrations. |
| Edge testing | **Go `testing` + `testify` + `gomock`** | Standard Go stack; protocol adapters tested against in-process simulators. |
| Cloud testing | **pytest + pytest-asyncio + httpx + testcontainers** | testcontainers spins real Postgres/Redis for integration tests. |
| Lint/format | Go: `golangci-lint`, `gofumpt`. Python: `ruff`, `black`, `mypy`. TS: `eslint`, `prettier`, `tsc` | Standard per ecosystem. |
| Migrations | **golang-migrate** (shared SQL) | Versioned migrations; the version table is also mirrored into edge SQLite so OTA can keep edge schema consistent. |

### Project Structure

```
industrial-iot-edge-platform/
├── README.md
├── docker-compose.yml                 # cloud plane + simulators for local dev
├── Makefile                           # build edge agent (all arches), run tests, lint
├── proto/                             # shared gRPC/protobuf contracts (edge <-> cloud)
│   ├── sync.proto                     # telemetry/alert upload, command download
│   ├── telemetry.proto
│   └── fleet.proto
│
├── edge/                              # Go edge agent
│   ├── go.mod
│   ├── cmd/edge-agent/main.go
│   ├── internal/
│   │   ├── config/                    # YAML config load + JSON Schema validation
│   │   ├── adapters/                  # protocol adapters (plugin interface)
│   │   │   ├── adapter.go             # Adapter interface
│   │   │   ├── modbus/
│   │   │   ├── opcua/
│   │   │   ├── mqtt/
│   │   │   ├── bacnet/
│   │   │   └── dnp3/
│   │   ├── engine/                    # filtering, aggregation, deadband, rules
│   │   │   ├── pipeline.go            # dataflow graph executor
│   │   │   └── nodes/                 # filter/aggregate/transform/sink node impls
│   │   ├── buffer/                    # SQLite store-and-forward
│   │   ├── sync/                      # gRPC/MQTT cloud sync client
│   │   ├── ml/                        # ONNX/TFLite inference runtime
│   │   ├── ota/                       # A/B update + rollback
│   │   ├── security/                  # mTLS, cert storage/rotation
│   │   └── telemetry/                 # internal health metrics
│   ├── simulators/                    # in-process protocol servers for tests
│   └── tests/
│
├── cloud/                             # Python FastAPI control plane
│   ├── pyproject.toml
│   ├── Dockerfile
│   ├── alembic_disabled_README.md     # (migrations live in /migrations via golang-migrate)
│   ├── app/
│   │   ├── main.py                    # FastAPI app factory
│   │   ├── config.py                  # Pydantic Settings
│   │   ├── db.py                      # async SQLAlchemy + asyncpg
│   │   ├── models/                    # SQLAlchemy ORM models
│   │   ├── schemas/                   # Pydantic request/response models
│   │   ├── routers/                   # fleet, devices, telemetry, alerts, pipelines, ota, ml, auth
│   │   ├── services/                  # business logic
│   │   ├── grpc/                      # gRPC ingestion server (telemetry/command)
│   │   ├── ingest/                    # telemetry ingestion -> TimescaleDB
│   │   ├── ota/                       # rollout orchestration (ARQ jobs)
│   │   ├── security/                  # cert issuance/rotation, JWT, RBAC
│   │   └── workers/                   # ARQ worker entrypoints
│   └── tests/
│
├── migrations/                        # golang-migrate SQL (cloud Postgres)
│   ├── 0001_core_hierarchy.up.sql
│   ├── 0002_fleet.up.sql
│   └── ...
│
├── dashboard/                         # React + TS SPA
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── api/                       # generated OpenAPI client
│       ├── pages/                     # Fleet, Devices, Telemetry, Alerts, Pipelines, OTA, ML
│       ├── components/
│       └── flow/                      # React Flow pipeline editor
│
├── schemas/                           # published JSON Schemas (edge config, telemetry payload)
└── deploy/
    ├── helm/                          # production Helm chart
    └── systemd/                       # edge agent systemd unit + install script
```

The structure is grouped by **concern and tier**, not by phase. Every phase below adds files within these directories without restructuring.

---

## Phase 1: Foundations — Contracts, Schema, and Skeletons

### Purpose
Establish the shared contracts (protobuf, JSON Schema, SQL migrations) and the runnable skeletons of all three tiers. After this phase, the edge agent boots and loads config, the cloud API serves a health check with an auto-generated OpenAPI spec, the database migrates cleanly, and the dashboard renders an empty shell. Nothing does real work yet, but the seams between components are defined and CI is green.

### Tasks

#### 1.1 — Repository, build tooling, and CI

**What**: Monorepo scaffold with reproducible builds and a CI pipeline running lint + tests for all three tiers.

**Design**:
- `Makefile` targets: `make edge ARCH=arm64`, `make cloud`, `make dashboard`, `make test`, `make lint`, `make proto`.
- `make edge` runs `CGO_ENABLED=1 GOARCH=$ARCH go build` (CGO on for SQLite/ONNX) producing `dist/edge-agent-$ARCH`.
- `docker-compose.yml` services: `postgres` (timescale/timescaledb:latest-pg16), `redis`, `minio`, `cloud-api`, `cloud-worker`, plus simulator containers added in later phases.
- GitHub Actions matrix: Go build for 3 arches, `golangci-lint`, `pytest`, `ruff`+`mypy`, `tsc`+`eslint`.

**Testing**:
- `CI: go vet + golangci-lint on edge/ → exit 0`
- `CI: ruff + mypy on cloud/ → exit 0`
- `Smoke: docker-compose up → postgres, redis, minio reach healthy state`
- `E2E: make edge ARCH=arm64 produces a binary; file(1) reports ELF arm aarch64`

#### 1.2 — Shared gRPC/protobuf contracts

**What**: Define the wire contracts between edge and cloud.

**Design**:
```protobuf
// proto/telemetry.proto
message Reading {
  string data_point_id = 1;     // UUID
  google.protobuf.Timestamp recorded_at = 2;
  oneof value { double numeric = 3; string text = 4; bool boolean = 5; }
  int32 quality = 6;            // OPC-UA quality code, 192 = Good
}
message ReadingBatch { string gateway_id = 1; repeated Reading readings = 2; }

// proto/sync.proto
service SyncService {
  rpc UploadTelemetry(stream ReadingBatch) returns (UploadAck);
  rpc UploadAlerts(stream AlertEvent) returns (UploadAck);
  rpc StreamCommands(GatewayIdentity) returns (stream Command);  // cloud -> edge
  rpc Heartbeat(HealthSnapshot) returns (HeartbeatResponse);
}
message Command {
  string command_id = 1;
  enum Type { PUSH_CONFIG=0; DEPLOY_MODEL=1; START_PIPELINE=2; STOP_PIPELINE=3; OTA_UPDATE=4; ROTATE_CERT=5; }
  Type type = 2;
  bytes payload = 3;           // JSON command body
}
message UploadAck { uint64 accepted = 1; uint64 rejected = 2; repeated string errors = 3; }
```
- `make proto` runs `protoc` generating Go (`edge/internal/sync/pb`) and Python (`cloud/app/grpc/pb`) stubs.

**Testing**:
- `Unit (Go): marshal/unmarshal ReadingBatch round-trips losslessly`
- `Unit (Py): generated stub imports; UploadAck fields accessible`
- `CI: protoc lint (buf) → no breaking changes vs. committed descriptor`

#### 1.3 — Cloud database schema & migrations (hybrid relational + JSONB)

**What**: Author golang-migrate SQL implementing the hybrid model (Data Model Suggestion 3) with the ISA-95 hierarchy, fleet, device registry, and JSONB heterogeneity columns. Telemetry tables become TimescaleDB hypertables.

**Design** (key tables — see Suggestion 3 for full detail):
```sql
-- 0001_core_hierarchy: organizations, sites, areas (ISA-95 levels 3-4)
-- 0002_fleet:
CREATE TABLE edge_gateways (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id UUID NOT NULL REFERENCES sites(id) ON DELETE RESTRICT,
    serial_number VARCHAR(100) NOT NULL UNIQUE,
    hardware_model VARCHAR(100) NOT NULL,
    cpu_architecture VARCHAR(20) NOT NULL,        -- 'arm64','armhf','x86_64'
    ram_mb INTEGER NOT NULL,
    agent_version VARCHAR(50),
    status VARCHAR(20) NOT NULL DEFAULT 'provisioning',
    capabilities JSONB NOT NULL DEFAULT '{}',     -- HETEROGENEITY: protocols, ml support, storage class
    last_heartbeat_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_gw_status CHECK (status IN
        ('provisioning','online','offline','degraded','decommissioned'))
);
-- 0003_devices: protocol_adapter_instances, devices, data_points
--   protocol_config JSONB on adapter (host/port OR serial_device/baud OR opcua endpoint)
--   metadata JSONB on devices (manufacturer-specific fields)
--   calibration JSONB on data_points (scaling/offset/deadband/valid bounds)
-- 0004_telemetry:
CREATE TABLE telemetry_readings (
    recorded_at TIMESTAMPTZ NOT NULL,
    data_point_id UUID NOT NULL,
    gateway_id UUID NOT NULL,
    value_numeric DOUBLE PRECISION,
    value_string  VARCHAR(500),
    value_boolean BOOLEAN,
    quality SMALLINT NOT NULL DEFAULT 192
);
SELECT create_hypertable('telemetry_readings','recorded_at',chunk_time_interval=>INTERVAL '1 day');
ALTER TABLE telemetry_readings SET (timescaledb.compress,
    timescaledb.compress_segmentby='data_point_id');
SELECT add_compression_policy('telemetry_readings', INTERVAL '7 days');
-- 0005_alerts, 0006_pipelines, 0007_ml, 0008_ota, 0009_security, 0010_users
```
- JSONB columns carry `pg_jsonschema` CHECK constraints (or app-level Pydantic validation) so heterogeneous fields stay disciplined.
- A `schema_migrations` table version is the value mirrored into edge SQLite (task 4.1).

**Testing**:
- `Integration (testcontainers): migrate up then down → no errors, clean teardown`
- `Integration: insert gateway with capabilities JSONB; query WHERE capabilities @> '{"protocols":["modbus_tcp"]}' returns it`
- `Integration: create_hypertable succeeds; inserting 10k readings then compressing a chunk reduces relation size`
- `Unit: chk_gw_status rejects status='banana' with constraint violation`

#### 1.4 — Cloud API skeleton + OpenAPI

**What**: FastAPI app with health/readiness endpoints, settings, DB session, and auto OpenAPI 3.1.

**Design**:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str; s3_bucket: str = "iiot-artifacts"
    jwt_secret: SecretStr; jwt_ttl_seconds: int = 3600
    grpc_port: int = 50051
# GET /healthz -> {"status":"ok"}
# GET /readyz  -> checks DB + Redis -> 200/503
# /openapi.json served automatically; /docs Swagger UI
```

**Testing**:
- `Unit: GET /healthz → 200 {"status":"ok"}`
- `Integration: /readyz with DB down → 503`
- `Unit: /openapi.json is valid OpenAPI 3.1 (validate against meta-schema)`

#### 1.5 — Edge agent skeleton + config loader

**What**: Go binary that loads/validates a YAML config and logs a startup banner.

**Design**:
```go
type Config struct {
    GatewayID   string         `yaml:"gateway_id"`
    Cloud       CloudConfig    `yaml:"cloud"`     // endpoint, mtls cert paths
    Adapters    []AdapterConfig`yaml:"adapters"`
    Buffer      BufferConfig   `yaml:"buffer"`    // sqlite path, max_size_mb, retention_hours
    LogLevel    string         `yaml:"log_level"`
}
func Load(path string) (*Config, error)   // reads YAML, validates against schemas/edge-config.schema.json
```
- Validation uses `santhosh-tekuri/jsonschema` against the published JSON Schema. Unknown adapter type → error naming the field.

**Testing**:
- `Unit: valid YAML → Config with defaults populated (poll_interval_ms=1000)`
- `Unit: missing gateway_id → error mentioning "gateway_id"`
- `Unit: adapters[0].type="frobnicator" → schema validation error`
- `E2E: run binary with sample config → logs "edge-agent <version> starting", exit 0`

---

## Phase 2: Protocol Adapters & Edge Polling Core

### Purpose
Build the heart of the edge value proposition: native OT protocol collection. After this phase the agent connects to real (simulated) industrial devices over Modbus, OPC-UA, and MQTT, polls data points on schedule, applies scaling/deadband, and emits normalised `Reading` structs onto an internal channel. This is where protocol heterogeneity is tamed behind a single interface.

### Tasks

#### 2.1 — Adapter interface & registry

**What**: A plugin contract every protocol adapter implements, plus a registry that instantiates adapters from config.

**Design**:
```go
type Reading struct {
    DataPointID string
    RecordedAt  time.Time
    Value       any        // float64 | string | bool
    Quality     int32      // 192 = Good
}
type Adapter interface {
    Connect(ctx context.Context) error
    Poll(ctx context.Context, points []DataPoint) ([]Reading, error)
    Write(ctx context.Context, point DataPoint, value any) error   // for writable points
    Close() error
    Health() AdapterHealth
}
type Factory func(cfg AdapterConfig) (Adapter, error)
var registry = map[string]Factory{}          // "modbus_tcp" -> factory
func Register(protocol string, f Factory)
```
- `DataPoint` carries `ProtocolAddress`, `DataType`, `ScalingFactor`, `ScalingOffset`, `Deadband`, `Writable`.
- Normalisation: raw register value → `value*scale + offset`, type-coerced to declared `DataType`.

**Testing**:
- `Unit: Register then look up "modbus_tcp" → factory returned`
- `Unit: unknown protocol → registry returns error`
- `Unit: scaling 100 raw, factor 0.1, offset 5 → normalised 15.0`

#### 2.2 — Modbus TCP/RTU adapter

**What**: Poll holding/input registers and coils over TCP and serial.

**Design**:
- Wrap `goburrow/modbus`. `protocol_config` JSON: `{host,port,unit_id}` (TCP) or `{serial_device,baud_rate,parity,unit_id}` (RTU).
- Address parsing: `"HR:40001:uint16"`, `"IR:30005:float32"`, `"COIL:1:bool"`. Float32 spans two registers (big/little-endian word order configurable).
- Batches contiguous register reads into a single Modbus request to minimise round-trips.

**Testing**:
- `Unit: parse "HR:40001:float32" → {table:holding, addr:40000, width:2}`
- `Integration (sim): start in-process Modbus TCP server with known registers; Poll → expected normalised values`
- `Integration (sim): server returns illegal-address exception → Reading quality set to Bad (0), no crash`
- `Integration (sim): contiguous addresses coalesced into one request (assert request count)`

#### 2.3 — OPC-UA adapter

**What**: Connect to an OPC-UA server, read NodeIds, optionally subscribe to monitored items.

**Design**:
- Wrap `gopcua/opcua`. `protocol_config`: `{endpoint, security_policy, security_mode, username?, cert_path?}`.
- `protocol_address` = OPC-UA NodeId string `ns=2;s=Temperature`. Map OPC-UA `StatusCode` → quality.
- Support both polled reads and `Subscribe` mode (monitored items push changes) selected per adapter.
- **Licence note**: native OPC-UA for commercial use requires the OPC Foundation Vendor License Agreement (per standards.md); document this in the adapter README and gate the build behind a `opcua` build tag.

**Testing**:
- `Integration (sim): open62541-based test server in docker; read ns=2;s=Temp → value`
- `Integration (sim): subscription mode → callback fires on simulated value change within poll window`
- `Unit: OPC-UA StatusCode Bad_NodeIdUnknown → quality Bad`

#### 2.4 — MQTT adapter (client + embedded broker)

**What**: Subscribe to device topics and (optionally) run a local broker for device-to-device messaging.

**Design**:
- Client via `paho.mqtt.golang`. `protocol_config`: `{broker_url, topics:[{topic, json_path, data_point_id}], qos}`.
- Extract values from JSON payloads via JSONPath into `Reading`s.
- Optional embedded broker (`mochi-mqtt/server`) bound to LAN for local pub/sub without cloud round-trips (mirrors Greengrass local broker).

**Testing**:
- `Integration (sim): publish to test broker topic → Reading emitted with extracted value`
- `Unit: JSONPath "$.sensors.temp" on sample payload → 21.5`
- `Integration: embedded broker accepts a local client connect + publish`

#### 2.5 — Poller scheduler

**What**: Per-adapter goroutine pool that polls data points at their configured interval and priority, feeding a central readings channel.

**Design**:
- One ticker per distinct `poll_interval_ms`; high-priority points (`poll_priority=1`) get a dedicated faster loop.
- Backpressure: bounded channel; if the downstream engine is slow, drop low-priority readings first and increment a dropped-reading metric.
- Adapter reconnect with exponential backoff on `Connect`/`Poll` errors.

**Testing**:
- `Unit: 3 points at 1000ms, 1 at 200ms → over 1s the 200ms point polled ~5x`
- `Integration (sim): kill sim mid-run → adapter reconnects, polling resumes`
- `Unit: full channel → low-priority readings dropped, dropped_total metric increments`

---

## Phase 3: Edge Processing Engine & Store-and-Forward

### Purpose
Turn raw readings into actionable, bandwidth-efficient output and guarantee no data loss during WAN outages. After this phase the agent filters/aggregates per configurable tiering policy, evaluates threshold alert rules locally (sub-100 ms decisions), and persists everything to a durable SQLite buffer with retention management. This delivers the headline "80% backhaul reduction" and "offline-first" capabilities.

### Tasks

#### 3.1 — SQLite store-and-forward buffer

**What**: Durable local buffer for telemetry and alerts awaiting cloud sync, plus a config cache.

**Design** (from Suggestion 1 edge schema):
```sql
CREATE TABLE telemetry_buffer (
  id INTEGER PRIMARY KEY AUTOINCREMENT, data_point_id TEXT NOT NULL,
  recorded_at TEXT NOT NULL, value_numeric REAL, value_string TEXT,
  value_boolean INTEGER, quality INTEGER NOT NULL DEFAULT 192,
  sync_status TEXT NOT NULL DEFAULT 'pending', retry_count INTEGER NOT NULL DEFAULT 0);
CREATE INDEX idx_buffer_sync ON telemetry_buffer(sync_status) WHERE sync_status='pending';
-- alert_buffer, config_cache as in Suggestion 1
```
- WAL mode, `PRAGMA synchronous=NORMAL`. `Buffer` interface: `Append([]Reading)`, `NextBatch(n)`, `MarkSynced(ids)`, `Prune(olderThan, maxSizeMB)`.
- Ring-buffer eviction: when `maxSizeMB` reached, drop oldest non-critical synced rows first, then oldest pending non-critical.

**Testing**:
- `Unit: Append 1000 then NextBatch(100) → 100 oldest pending rows`
- `Unit: MarkSynced → rows move to 'confirmed', excluded from next batch`
- `Unit: exceed maxSizeMB → oldest synced pruned, critical rows retained`
- `Integration: kill process mid-write (WAL) → reopen, no corruption, committed rows present`

#### 3.2 — Dataflow pipeline executor

**What**: Execute a directed graph of processing nodes (source → filter → aggregate → transform → sink) defined by config.

**Design**:
```go
type Node interface { Process(in []Reading) ([]Reading, error); Type() string }
// node types: filter_threshold, filter_deadband, aggregate_window, transform_scale,
//             transform_unit, ml_inference, sink_cloud, sink_local_store, sink_mqtt
type Pipeline struct { nodes []Node; edges map[string][]string }
func (p *Pipeline) Run(readings []Reading) error   // topological execution
```
- `aggregate_window`: tumbling window (e.g. 5-min) producing min/max/avg/stddev/count per point.
- `filter_deadband`: suppress readings whose change < deadband unless `is_critical`.
- Sinks: `sink_cloud` writes to buffer with `pending`; `sink_local_store` keeps locally only.

**Testing**:
- `Unit: deadband 0.5, sequence [10.0,10.2,10.8] → emits 10.0 and 10.8 (10.2 suppressed)`
- `Unit: 5-min window over 300 readings → one aggregate with correct avg/stddev`
- `Unit: cyclic graph → Run returns error before executing`
- `Integration: source→deadband→sink_cloud → only passing readings reach buffer`

#### 3.3 — Local alert rule engine

**What**: Evaluate threshold/range/rate-of-change/flatline rules on the edge in real time.

**Design**:
```go
type Rule struct {
    ID string; DataPointID string
    ConditionType string  // threshold_high|threshold_low|range_outside|rate_of_change|flatline|quality_bad
    ThresholdHigh, ThresholdLow float64
    DurationMs int          // must persist this long
    CooldownMs int; Severity string
}
func (e *Engine) Evaluate(r Reading) []AlertEvent
```
- Stateful: tracks how long a condition has held (debounce via `DurationMs`) and last-fired time (`CooldownMs`).
- Emits `AlertEvent` to the alert buffer immediately; latency target <100 ms from reading to alert decision.

**Testing**:
- `Unit: threshold_high=80, reading 85 held > DurationMs → AlertTriggered`
- `Unit: 85 then 70 before DurationMs elapses → no alert`
- `Unit: within cooldown → no duplicate alert`
- `Unit: flatline (N identical readings) → alert`
- `Bench: Evaluate p99 latency < 1 ms per reading`

#### 3.4 — Data tiering policy

**What**: Decide per data point whether to forward raw, aggregates, or alerts-only based on policy and bandwidth.

**Design**:
- `tier ∈ {raw_all, raw_critical_only, aggregate_5min, aggregate_1hr, alerts_only}` resolved per data point (point policy overrides gateway default).
- A bandwidth governor measures recent sync throughput; if `bandwidth_limit_kbps` would be exceeded, automatically downgrades non-critical points one tier and logs the decision.

**Testing**:
- `Unit: tier=alerts_only → telemetry readings not buffered for cloud, alerts still buffered`
- `Unit: tier=aggregate_5min → raw suppressed, 5-min aggregates buffered`
- `Unit: bandwidth exceeded → non-critical point downgraded raw_all→aggregate_5min`
- `Unit: critical point never downgraded`

---

## Phase 4: Edge↔Cloud Sync, Ingestion & Security

### Purpose
Connect the two tiers securely and reliably. After this phase the edge authenticates with a client certificate over mutual TLS, streams buffered telemetry and alerts to the cloud, receives commands, and reconciles on reconnect after an outage. The cloud ingests readings into TimescaleDB and tracks sync sessions. This realises offline-first sync and IEC 62443-aligned secure communication.

### Tasks

#### 4.1 — mTLS provisioning & certificate store (edge + cloud)

**What**: Issue per-gateway X.509 client certs from a platform CA; edge stores and presents them; cloud verifies.

**Design**:
- Cloud `security` service: internal CA (root + intermediate) using Python `cryptography`. `POST /v1/gateways/{id}/certificates` issues a client cert (CN = gateway serial), stores metadata in `certificates` table (subject, issuer, not_after, fingerprint).
- Edge stores cert/key under `0600` perms; presents on gRPC dial. Cloud gRPC server requires and verifies client cert against the CA, maps CN → gateway identity.
- Rotation: cloud job flags certs within 30 days of expiry as `pending_rotation`; pushes `ROTATE_CERT` command; edge fetches and atomically swaps.

**Testing**:
- `Integration: issue cert → stored row with correct not_after and SHA-256 fingerprint`
- `Integration (mTLS): edge dials with valid cert → handshake ok, identity = serial`
- `Integration (mTLS): expired/unknown cert → handshake rejected, audit_log entry`
- `Integration: rotation job marks cert <30d as pending_rotation`

#### 4.2 — Telemetry sync client (edge)

**What**: Stream pending buffer rows to the cloud and mark confirmed on ack.

**Design**:
- Loop: `NextBatch(500)` → `UploadTelemetry` stream → on `UploadAck` `MarkSynced(accepted)`; rejected rows get `retry_count++`, exponential backoff, dead-letter after N retries.
- On reconnect after outage, drains backlog oldest-first while respecting bandwidth governor.
- Heartbeat goroutine sends `HealthSnapshot` (cpu/mem/disk/wan_latency/temp) every 30 s.

**Testing**:
- `Integration (mock server): 500 readings → all marked confirmed`
- `Integration: server rejects 5 → those stay pending, retry_count incremented`
- `Integration: simulate 10-min outage with buffering → on reconnect, full backlog drains in order`
- `Unit: ack accepted=N → exactly first N batch rows marked synced`

#### 4.3 — gRPC ingestion server (cloud)

**What**: Accept streamed telemetry/alerts, validate, write to TimescaleDB, record sync sessions.

**Design**:
- `UploadTelemetry`: per-reading validation (data_point exists, value within `min/max_valid_value` from calibration JSONB, timestamp not in future); valid → batched `COPY` into `telemetry_readings`; invalid → counted in `UploadAck.rejected` with reason.
- Opens/closes a `sync_sessions` row (records_received, bytes, status).
- Latest-value cache updated for dashboard (`rm_data_points.latest_value` style materialised row).

**Testing**:
- `Integration (testcontainers): stream 1000 readings → 1000 rows in hypertable, session completed`
- `Integration: value above max_valid_value → rejected with "out_of_bounds", row not inserted`
- `Integration: future timestamp → rejected`
- `Load: 50k readings/s sustained for 60s → no backlog growth (COPY path)`

#### 4.4 — Command channel (cloud → edge)

**What**: Push config/model/pipeline/OTA commands to connected gateways over the `StreamCommands` server stream.

**Design**:
- Cloud holds a per-gateway command queue (Redis list). On gateway connect, server drains queued commands; new commands pushed live.
- Edge ACKs each command (separate unary RPC `AckCommand{command_id, status, error}`); cloud updates command state. Commands received while edge briefly offline persist until delivered (mirrors `pending_commands` edge table).

**Testing**:
- `Integration: enqueue PUSH_CONFIG → connected edge receives it, ACKs success`
- `Integration: command enqueued while edge offline → delivered on reconnect`
- `Unit: ACK failure status → command marked failed, surfaced in API`

---

## Phase 5: Cloud Fleet Management API & REST Surface

### Purpose
Expose the full management surface as a documented REST API so the dashboard, CLI, and third-party integrations can drive the platform. After this phase, operators can register sites/gateways/devices/data points, define alert rules and tiering policies, and query telemetry — all via OpenAPI 3.1 endpoints with RBAC. This is parallelisable with Phase 6 (dashboard) once the contracts exist.

### Tasks

#### 5.1 — AuthN/AuthZ (users, JWT, RBAC, API keys)

**What**: Human login + role-based access and programmatic API keys.

**Design**:
- `POST /v1/auth/login` (OAuth2 password) → JWT (role claim + org_id). Roles: `admin|engineer|operator|viewer`; site-scoped grants via `user_site_access`.
- API keys: `Authorization: Bearer iiot_<prefix>_<secret>`; stored as SHA-256 hash with `scopes[]` (`telemetry:read`, `devices:write`, `fleet:admin`). Dependency `require_scope(...)` guards routes.

**Testing**:
- `Unit: viewer calls POST /v1/devices → 403`
- `Unit: engineer with site grant → 200; without grant for that site → 403`
- `Unit: API key wrong scope → 403; correct scope → 200`
- `Integration: expired JWT → 401`

#### 5.2 — Fleet & hierarchy endpoints

**What**: CRUD for organizations, sites, areas, gateways; gateway provisioning workflow.

**Design**:
- `POST /v1/gateways` (provision: validates serial unique, site exists, capabilities JSONB) → creates gateway, issues cert (5.1/4.1), returns an enrolment bundle (cert + bootstrap config).
- `GET /v1/gateways?site_id=&status=&tag=` paginated; `GET /v1/gateways/{id}` includes latest health snapshot.
- `POST /v1/gateways/{id}:decommission`.

**Testing**:
- `Integration: provision gateway → 201 with cert bundle; duplicate serial → 409`
- `Integration: list filtered by status=online → only online gateways`
- `Integration: decommission → status=decommissioned, cert revoked`

#### 5.3 — Device, data point, adapter endpoints

**What**: Register adapters/devices/data points with heterogeneous JSONB config.

**Design**:
- `POST /v1/gateways/{id}/adapters` body includes `protocol` + `protocol_config` JSONB (validated against per-protocol JSON Schema).
- `POST /v1/devices`, `POST /v1/data-points` with `calibration` JSONB. Creating these enqueues a `PUSH_CONFIG` command (Phase 4.4) so the edge starts polling.

**Testing**:
- `Integration: create modbus adapter with valid protocol_config → 201; missing host → 422`
- `Integration: add data point → PUSH_CONFIG command enqueued for gateway`
- `Integration: opcua adapter config validated against opcua schema`

#### 5.4 — Telemetry, alerts & tiering endpoints

**What**: Query telemetry/aggregates, manage alert rules + acknowledgements, manage tiering policies.

**Design**:
- `GET /v1/telemetry?data_point_id=&from=&to=&agg=raw|5min|1hr` → reads hypertable or continuous aggregate; supports Line Protocol export (`Accept: text/plain`).
- `GET /v1/alerts?status=active&severity=` , `POST /v1/alerts/{id}:acknowledge`.
- `POST /v1/alert-rules`, `POST /v1/tiering-policies` → both enqueue PUSH_CONFIG so edge enforces locally.

**Testing**:
- `Integration: query agg=5min → returns continuous-aggregate buckets`
- `Integration: Accept text/plain → valid InfluxDB Line Protocol body`
- `Integration: acknowledge alert → acknowledged_by/at set`
- `Integration: create alert rule → PUSH_CONFIG enqueued`

---

## Phase 6: Operations Dashboard & Visual Pipeline Editor

### Purpose
Deliver the operator-facing UI: a unified fleet dashboard with real-time and historical views, plus the no-code visual dataflow editor that lets OT engineers configure routing/filtering without writing code (the Kura Wire differentiator). Parallelisable with Phase 5 after contracts are fixed.

### Tasks

#### 6.1 — Dashboard shell, auth, generated API client

**What**: React SPA with login, routing, and a typed client generated from the OpenAPI spec.

**Design**:
- `openapi-typescript` generates `dashboard/src/api/types.ts`; TanStack Query hooks wrap each endpoint.
- Layout: sidebar (Fleet, Devices, Telemetry, Alerts, Pipelines, OTA, Models, Settings). JWT in memory + refresh; route guards by role.

**Testing**:
- `Component (RTL): login form submits → token stored, redirect to Fleet`
- `Component: viewer role → Pipelines edit controls disabled`
- `E2E (Playwright): login → fleet list renders rows from mocked API`

#### 6.2 — Fleet & device views

**What**: Fleet map/list with health, drill-down to device + data point detail with live values.

**Design**:
- Fleet table: status badge, last heartbeat, cpu/mem/temp sparklines (uPlot), filter by site/tag/status.
- Device detail: data point table with latest value + quality; auto-refresh via TanStack Query polling (configurable interval).

**Testing**:
- `Component: gateway offline > heartbeat TTL → rendered as "offline" badge`
- `E2E: open device → data points list with latest values`

#### 6.3 — Telemetry trend explorer

**What**: Interactive time-series charting with aggregation selection.

**Design**:
- uPlot chart; selectors for data point(s), time range, agg level (raw/5min/1hr). Lazy-load via range queries; downsample client-side beyond a point threshold.

**Testing**:
- `Component: switch agg raw→5min → refetches with agg=5min param`
- `E2E: select 24h range → chart renders aggregated series`

#### 6.4 — Alerts console

**What**: Active alerts list, acknowledgement, history, severity filtering.

**Design**:
- Active alerts sorted by severity/time; acknowledge action (optimistic update). History view reads resolved alerts with duration.

**Testing**:
- `Component: acknowledge → row shows acknowledged_by, mutation called`
- `E2E: filter severity=critical → only critical rows`

#### 6.5 — Visual dataflow pipeline editor (React Flow)

**What**: Drag-and-drop canvas to build source→filter→aggregate→transform→sink pipelines, persisted and deployed to a gateway.

**Design**:
- React Flow canvas; node palette mirrors edge node types (3.2). Each node has a config form (e.g. deadband threshold, window size, sink target).
- Save → `POST /v1/pipelines` storing nodes/edges (relational `pipeline_nodes`/`pipeline_edges` + `config_json`). Deploy → enqueues `START_PIPELINE` PUSH_CONFIG.
- Client-side validation: no cycles, every sink reachable from a source, required configs present.

**Testing**:
- `Component: connect source→sink, save → POST body has matching nodes/edges`
- `Component: create cycle → save blocked with validation error`
- `E2E: build deadband pipeline, deploy → START_PIPELINE command enqueued (mocked API asserts call)`

---

## Phase 7: OTA Updates & Edge ML Inference

### Purpose
Add fleet-scale software lifecycle management and edge intelligence. After this phase, operators stage OTA rollouts with canary/staged strategies and automatic rollback, and deploy pre-trained anomaly/predictive-maintenance models that run locally with sub-100 ms inference. These are the v1.1 differentiators.

### Tasks

#### 7.1 — OTA release & rollout orchestration (cloud)

**What**: Manage signed release artifacts and staged rollout campaigns with auto-halt.

**Design**:
- `software_releases` (artifact in S3, SHA-256, target_arch[], signature). `rollout_campaigns` with `strategy ∈ {immediate,staged,canary}`, `stage_pct[]={10,50,100}`, `max_failure_pct`.
- ARQ orchestrator: per stage, select gateway cohort, enqueue `OTA_UPDATE` commands, watch `rollout_gateway_status`; if failure rate > `max_failure_pct` → pause campaign + alert.

**Testing**:
- `Integration: create campaign staged → stage 1 targets ~10% of gateways`
- `Integration: failures exceed max_failure_pct → campaign auto-paused`
- `Unit: artifact signature mismatch → release rejected at upload`

#### 7.2 — Edge OTA apply with A/B + rollback

**What**: Edge downloads, verifies, installs to inactive slot, swaps, and self-rolls-back on failed health check.

**Design**:
- On `OTA_UPDATE`: download artifact → verify SHA-256 + signature → write to inactive A/B slot → update bootloader pointer → restart into new slot.
- Post-boot health gate: if new agent fails to reach `online` + pass self-check within `health_timeout`, bootloader reverts to previous slot; edge reports `RolloutGatewayFailed{rolled_back:true}`.

**Testing**:
- `Integration (sim): valid artifact → installed to inactive slot, pointer swapped`
- `Integration: corrupted artifact (bad hash) → install aborted, current slot untouched`
- `Integration: new agent fails health gate → automatic rollback to previous version, status reported`

#### 7.3 — ML model registry & deployment (cloud)

**What**: Register ONNX/TFLite models, version artifacts, deploy to compatible gateways.

**Design**:
- `ml_models`/`ml_model_versions` (artifact_url, hash, target_arch[], min_ram_mb, input/output schema). Deploy validates gateway `capabilities` JSONB meets `min_ram_mb` + arch, then enqueues `DEPLOY_MODEL`.

**Testing**:
- `Integration: deploy model needing 256MB to 128MB gateway → 422 incompatible`
- `Integration: compatible gateway → DEPLOY_MODEL enqueued`

#### 7.4 — Edge ML inference runtime

**What**: Run deployed models as an `ml_inference` pipeline node producing anomaly scores/predictions locally.

**Design**:
- ONNX Runtime (Go) / TFLite loads model; `ml_inference` node feeds a feature window (e.g. last N readings) → output tensor → emits derived `Reading`(s) (e.g. `anomaly_score`) or triggers an alert when score > threshold.
- Inference time recorded; target p99 < 100 ms on reference ARM hardware.

**Testing**:
- `Unit: feed fixture window to bundled test ONNX model → expected output within tolerance`
- `Integration: anomaly score > threshold → AlertTriggered via rule engine`
- `Bench: inference p99 < 100 ms on amd64 CI runner (proxy for target)`

---

## Phase 8: Cloud-Agnostic Connectors, Hardening & Advanced (Backlog)

### Purpose
Broaden interoperability and harden for production, then seed the differentiating backlog. After this phase the platform forwards to any MQTT cloud (AWS IoT Core, Azure IoT Hub, generic), passes security review, and has the foundations for protocol auto-discovery and OT security monitoring.

### Tasks

#### 8.1 — Cloud-agnostic MQTT egress connector (edge)

**What**: A `sink_cloud` variant that publishes to external MQTT endpoints instead of (or in addition to) the native gRPC plane.

**Design**:
- Config: `{provider: aws_iot|azure_iot|generic, broker_url, topic_template, qos, tls}`. Topic templating per provider conventions; payload as JSON or Line Protocol.

**Testing**:
- `Integration (sim broker): aws_iot config → publishes to templated topic with correct payload`
- `Unit: topic_template renders {site}/{gateway}/{point} correctly`

#### 8.2 — Secure remote tunnel for edge management

**What**: On-demand reverse tunnel from cloud to the edge gateway's local management interface.

**Design**:
- Cloud issues a `OPEN_TUNNEL` command; edge dials out (no inbound firewall change) establishing a multiplexed mTLS tunnel (yamux over the gRPC connection) exposing the gateway's local admin port to an authorised cloud-side session, time-boxed and audit-logged.

**Testing**:
- `Integration: open tunnel → cloud reaches edge admin endpoint; close → connection torn down`
- `Integration: tunnel session recorded in audit_log with operator identity`

#### 8.3 — Security hardening & IEC 62443 alignment

**What**: Production security pass.

**Design**:
- Enforce TLS 1.3 minimum; secrets via env/secret manager (never in config files); rate limiting on auth + ingestion; full `audit_log` coverage of privileged actions; network-zoning guidance doc; dependency + container CVE scanning in CI.

**Testing**:
- `Integration: TLS 1.1 handshake → rejected`
- `Integration: 100 failed logins → rate-limited 429`
- `CI: trivy scan of cloud image → no HIGH/CRITICAL unpatched CVEs`
- `Integration: privileged action (cert.rotate) → audit_log row written`

#### 8.4 — Protocol auto-discovery (backlog, AI-augmented)

**What**: Passively observe network traffic to suggest device types and adapter configuration.

**Design**:
- Edge sniffer module classifies observed traffic (port/payload heuristics → Modbus/OPC-UA/BACnet/DNP3) and emits discovery candidates; optional LLM call (cloud-side) maps observed register patterns to likely device types, surfaced as suggested config in the dashboard for operator confirmation.

**Testing**:
- `Unit: replay captured Modbus TCP pcap → classified as modbus_tcp with unit IDs`
- `Integration: discovery candidates surface in dashboard as suggestions (not auto-applied)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundations (contracts, schema, skeletons)   ─── required by everything
    │
Phase 2: Protocol Adapters & Polling                  ─── requires P1
    │
Phase 3: Processing Engine & Store-and-Forward        ─── requires P2
    │
Phase 4: Edge↔Cloud Sync, Ingestion, Security         ─── requires P1, P3
    │
    ├── Phase 5: Fleet Management REST API             ─── requires P4 ──┐
    │                                                                    │ (5 & 6 parallel
    └── Phase 6: Dashboard & Pipeline Editor           ─── requires P5 contracts ┘  after 5's contracts)
         │
Phase 7: OTA & Edge ML                                ─── requires P4 (commands) + P5 (API)
    │
Phase 8: Connectors, Hardening, Backlog               ─── requires P4–P7
```

**Parallelism opportunities**
- Within Phase 2, the Modbus/OPC-UA/MQTT/BACnet/DNP3 adapters (2.2–2.4 + later) are independent once the adapter interface (2.1) lands — split across developers.
- Phase 5 (API) and Phase 6 (dashboard) proceed concurrently once Phase 5's OpenAPI contracts are frozen; the dashboard develops against the generated client + mocked responses.
- Phase 7.1/7.3 (cloud OTA/ML registry) and 7.2/7.4 (edge apply/inference) are independent given the Phase 4 command channel.
- Phase 8 tasks are largely independent of one another.

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; new tests cover happy-path and the named edge cases above.
3. Linting/formatting passes: `golangci-lint`/`gofumpt` (edge), `ruff`/`black` (cloud), `eslint`/`prettier` (dashboard).
4. Type checking passes: `go vet` (edge), `mypy` (cloud), `tsc --noEmit` (dashboard).
5. Cloud Docker image builds; edge agent cross-compiles for `arm64`, `armhf`, and `amd64`.
6. The phase's headline capability works end-to-end against simulators (no manual data fixup).
7. New configuration options are documented in the relevant README and reflected in `schemas/edge-config.schema.json`.
8. New/changed REST endpoints appear in the auto-generated `/openapi.json` and validate as OpenAPI 3.1.
9. New gRPC messages are reflected in `proto/` and pass `buf` breaking-change checks.
10. Database changes ship as forward + rollback golang-migrate scripts that apply cleanly on a fresh and an existing database.
11. Security-relevant actions added in the phase emit `audit_log` entries.
```
