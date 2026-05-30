# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Industrial IoT Edge Platform (452) · Generated: 2026-05-25

## Overview

This model uses a fully normalized relational schema in PostgreSQL to represent the entire domain: edge gateway fleet, industrial protocol adapters, device/sensor hierarchies, telemetry data, ML model deployments, OTA updates, security credentials, and data pipeline configurations. Normalization is carried to 3NF throughout, with referential integrity enforced by foreign keys and domain constraints.

The schema follows the ISA-95 equipment hierarchy (Enterprise > Site > Area > Work Cell > Equipment > Sensor) to align with industry standards and ensure the data model is immediately comprehensible to OT engineers.

---

## Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Primary RDBMS | PostgreSQL 16+ | Mature, extensible, strong indexing, partitioning, LISTEN/NOTIFY for real-time |
| Time-series extension | TimescaleDB (optional layer) | Hypertables for telemetry if volume demands it — keeps everything in one engine |
| Edge-local DB | SQLite 3 | Store-and-forward buffer on constrained gateways (128 MB RAM target) |
| Migration tool | Flyway or golang-migrate | Versioned schema migrations for fleet-wide consistency |
| Connection pooling | PgBouncer | High-concurrency telemetry ingest from hundreds of gateways |

---

## Complete Schema Definition

### 1. Organization and Site Hierarchy (ISA-95 Levels 3-4)

```sql
-- Represents the top-level organizational entity
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    industry_sector VARCHAR(100),          -- e.g. 'manufacturing', 'energy', 'utilities', 'building_automation'
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Physical locations where edge gateways are deployed
CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    country_code    CHAR(2),               -- ISO 3166-1 alpha-2
    postal_code     VARCHAR(20),
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    isa95_level     SMALLINT DEFAULT 3,    -- ISA-95 hierarchy level
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

-- Functional areas within a site (e.g. production line, building wing, substation)
CREATE TABLE areas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    area_type       VARCHAR(50) NOT NULL,  -- 'production_line', 'substation', 'hvac_zone', 'utility_area'
    description     TEXT,
    parent_area_id  UUID REFERENCES areas(id) ON DELETE SET NULL,  -- nested areas
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, slug)
);

CREATE INDEX idx_areas_site ON areas(site_id);
CREATE INDEX idx_areas_parent ON areas(parent_area_id);
```

### 2. Edge Gateway Fleet Management

```sql
-- Edge gateway hardware and software identity
CREATE TABLE edge_gateways (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id) ON DELETE RESTRICT,
    area_id             UUID REFERENCES areas(id) ON DELETE SET NULL,
    serial_number       VARCHAR(100) NOT NULL UNIQUE,
    hardware_model      VARCHAR(100) NOT NULL,     -- e.g. 'Robustel R1520', 'Advantech UNO-2484G'
    hardware_revision   VARCHAR(20),
    cpu_architecture    VARCHAR(20) NOT NULL,       -- 'arm64', 'armhf', 'x86_64'
    ram_mb              INTEGER NOT NULL,
    storage_mb          INTEGER NOT NULL,
    os_name             VARCHAR(50) NOT NULL,       -- 'linux', 'windows_iot'
    os_version          VARCHAR(50),
    agent_version       VARCHAR(50),                -- currently installed edge agent version
    ip_address_lan      INET,
    ip_address_wan      INET,
    mac_address         MACADDR,
    status              VARCHAR(20) NOT NULL DEFAULT 'provisioning',
        -- 'provisioning', 'online', 'offline', 'degraded', 'decommissioned'
    last_heartbeat_at   TIMESTAMPTZ,
    last_sync_at        TIMESTAMPTZ,
    provisioned_at      TIMESTAMPTZ,
    decommissioned_at   TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_gateway_status CHECK (
        status IN ('provisioning', 'online', 'offline', 'degraded', 'decommissioned')
    )
);

CREATE INDEX idx_gateways_site ON edge_gateways(site_id);
CREATE INDEX idx_gateways_status ON edge_gateways(status);
CREATE INDEX idx_gateways_heartbeat ON edge_gateways(last_heartbeat_at);

-- Gateway resource utilization snapshots (for fleet health monitoring)
CREATE TABLE gateway_health_snapshots (
    id              BIGSERIAL PRIMARY KEY,
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    cpu_percent     REAL,
    memory_used_mb  REAL,
    memory_total_mb REAL,
    disk_used_mb    REAL,
    disk_total_mb   REAL,
    uptime_seconds  BIGINT,
    wan_latency_ms  REAL,
    wan_connected   BOOLEAN NOT NULL DEFAULT true,
    temperature_c   REAL                            -- hardware temperature sensor if available
);

CREATE INDEX idx_health_gateway_time ON gateway_health_snapshots(gateway_id, recorded_at DESC);

-- Gateway tags for grouping and fleet segmentation
CREATE TABLE gateway_tags (
    gateway_id  UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    tag_key     VARCHAR(100) NOT NULL,
    tag_value   VARCHAR(255) NOT NULL,
    PRIMARY KEY (gateway_id, tag_key)
);
```

### 3. Protocol Adapters and Device Registry

```sql
-- Catalogue of supported protocol adapter types
CREATE TABLE protocol_adapter_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    protocol_name   VARCHAR(50) NOT NULL UNIQUE,   -- 'opcua', 'modbus_tcp', 'modbus_rtu', 'mqtt', 'bacnet', 'dnp3', 'profinet'
    display_name    VARCHAR(100) NOT NULL,
    transport       VARCHAR(20) NOT NULL,          -- 'tcp', 'serial', 'udp'
    default_port    INTEGER,
    description     TEXT,
    requires_licence BOOLEAN NOT NULL DEFAULT false, -- e.g. OPC-UA vendor licence
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Instances of protocol adapters running on specific gateways
CREATE TABLE protocol_adapter_instances (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id          UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    adapter_type_id     UUID NOT NULL REFERENCES protocol_adapter_types(id) ON DELETE RESTRICT,
    name                VARCHAR(255) NOT NULL,
    enabled             BOOLEAN NOT NULL DEFAULT true,
    connection_host     VARCHAR(255),              -- target host for TCP protocols
    connection_port     INTEGER,
    serial_device       VARCHAR(100),              -- e.g. '/dev/ttyUSB0' for Modbus RTU
    baud_rate           INTEGER,                   -- serial protocols
    poll_interval_ms    INTEGER NOT NULL DEFAULT 1000,
    timeout_ms          INTEGER NOT NULL DEFAULT 5000,
    retry_count         INTEGER NOT NULL DEFAULT 3,
    status              VARCHAR(20) NOT NULL DEFAULT 'stopped',
        -- 'running', 'stopped', 'error', 'connecting'
    last_error          TEXT,
    last_data_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_adapter_status CHECK (
        status IN ('running', 'stopped', 'error', 'connecting')
    )
);

CREATE INDEX idx_adapter_gateway ON protocol_adapter_instances(gateway_id);

-- Industrial devices discovered or registered on each adapter
CREATE TABLE devices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    adapter_instance_id UUID NOT NULL REFERENCES protocol_adapter_instances(id) ON DELETE CASCADE,
    gateway_id          UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    area_id             UUID REFERENCES areas(id) ON DELETE SET NULL,
    device_address      VARCHAR(255) NOT NULL,     -- protocol-specific: Modbus unit ID, OPC-UA node path, BACnet device instance
    name                VARCHAR(255) NOT NULL,
    device_type         VARCHAR(100),              -- 'plc', 'vfd', 'temperature_controller', 'power_meter', 'hvac_controller'
    manufacturer        VARCHAR(255),
    model               VARCHAR(255),
    firmware_version    VARCHAR(50),
    serial_number       VARCHAR(100),
    isa95_equipment_class VARCHAR(100),            -- ISA-95 equipment classification
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
        -- 'active', 'inactive', 'maintenance', 'decommissioned'
    discovered_at       TIMESTAMPTZ,
    last_seen_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (adapter_instance_id, device_address)
);

CREATE INDEX idx_devices_gateway ON devices(gateway_id);
CREATE INDEX idx_devices_adapter ON devices(adapter_instance_id);
CREATE INDEX idx_devices_area ON devices(area_id);
```

### 4. Sensor / Data Point Registry

```sql
-- Individual data points (sensors, registers, variables) on each device
CREATE TABLE data_points (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id           UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    display_name        VARCHAR(255),
    protocol_address    VARCHAR(255) NOT NULL,     -- Modbus register address, OPC-UA NodeId, BACnet object identifier
    data_type           VARCHAR(30) NOT NULL,      -- 'float32', 'float64', 'int16', 'int32', 'uint16', 'uint32', 'boolean', 'string'
    unit_of_measure     VARCHAR(50),               -- 'celsius', 'bar', 'rpm', 'kwh', 'percent', 'mm_s' (vibration velocity)
    scaling_factor      DOUBLE PRECISION DEFAULT 1.0,
    scaling_offset      DOUBLE PRECISION DEFAULT 0.0,
    min_valid_value     DOUBLE PRECISION,          -- physical validity bounds
    max_valid_value     DOUBLE PRECISION,
    deadband_value      DOUBLE PRECISION,          -- minimum change to trigger a new reading
    poll_priority       SMALLINT NOT NULL DEFAULT 1, -- 1=high, 2=normal, 3=low
    is_writable         BOOLEAN NOT NULL DEFAULT false,
    is_critical         BOOLEAN NOT NULL DEFAULT false,  -- always forwarded to cloud
    enabled             BOOLEAN NOT NULL DEFAULT true,
    isa95_property_name VARCHAR(100),              -- ISA-95 property mapping
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (device_id, protocol_address)
);

CREATE INDEX idx_datapoints_device ON data_points(device_id);
CREATE INDEX idx_datapoints_critical ON data_points(is_critical) WHERE is_critical = true;
```

### 5. Telemetry Data Storage

```sql
-- Raw telemetry readings — the highest-volume table
-- Partitioned by time for efficient retention management
CREATE TABLE telemetry_readings (
    id              BIGSERIAL,
    data_point_id   UUID NOT NULL REFERENCES data_points(id) ON DELETE CASCADE,
    gateway_id      UUID NOT NULL,                 -- denormalized for partition pruning
    recorded_at     TIMESTAMPTZ NOT NULL,          -- timestamp from the edge device
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- timestamp when cloud received it
    value_numeric   DOUBLE PRECISION,
    value_string    VARCHAR(500),
    value_boolean   BOOLEAN,
    quality         SMALLINT NOT NULL DEFAULT 192, -- OPC-UA quality code; 192 = Good
    sync_status     VARCHAR(10) NOT NULL DEFAULT 'synced',
        -- 'pending', 'synced', 'failed'

    PRIMARY KEY (recorded_at, id)
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions (example for 2026)
CREATE TABLE telemetry_readings_2026_01 PARTITION OF telemetry_readings
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE telemetry_readings_2026_02 PARTITION OF telemetry_readings
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by automated job

CREATE INDEX idx_telemetry_datapoint_time ON telemetry_readings(data_point_id, recorded_at DESC);
CREATE INDEX idx_telemetry_gateway_time ON telemetry_readings(gateway_id, recorded_at DESC);
CREATE INDEX idx_telemetry_sync ON telemetry_readings(sync_status) WHERE sync_status != 'synced';

-- Pre-aggregated telemetry summaries (5-minute windows)
CREATE TABLE telemetry_aggregates_5min (
    data_point_id   UUID NOT NULL REFERENCES data_points(id) ON DELETE CASCADE,
    bucket_start    TIMESTAMPTZ NOT NULL,
    value_min       DOUBLE PRECISION,
    value_max       DOUBLE PRECISION,
    value_avg       DOUBLE PRECISION,
    value_stddev    DOUBLE PRECISION,
    value_count     INTEGER NOT NULL,
    value_sum       DOUBLE PRECISION,
    quality_good_pct REAL,                         -- percentage of readings with good quality

    PRIMARY KEY (data_point_id, bucket_start)
) PARTITION BY RANGE (bucket_start);

-- Hourly aggregates for longer-term trending
CREATE TABLE telemetry_aggregates_1hr (
    data_point_id   UUID NOT NULL REFERENCES data_points(id) ON DELETE CASCADE,
    bucket_start    TIMESTAMPTZ NOT NULL,
    value_min       DOUBLE PRECISION,
    value_max       DOUBLE PRECISION,
    value_avg       DOUBLE PRECISION,
    value_stddev    DOUBLE PRECISION,
    value_count     INTEGER NOT NULL,
    value_sum       DOUBLE PRECISION,
    quality_good_pct REAL,

    PRIMARY KEY (data_point_id, bucket_start)
) PARTITION BY RANGE (bucket_start);
```

### 6. Alerts and Threshold Rules

```sql
-- Alert rule definitions configured per data point or device
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    data_point_id   UUID REFERENCES data_points(id) ON DELETE CASCADE,     -- specific point
    device_id       UUID REFERENCES devices(id) ON DELETE CASCADE,         -- all points on device
    condition_type  VARCHAR(30) NOT NULL,
        -- 'threshold_high', 'threshold_low', 'range_outside', 'rate_of_change', 'flatline', 'quality_bad'
    threshold_value DOUBLE PRECISION,
    threshold_high  DOUBLE PRECISION,
    threshold_low   DOUBLE PRECISION,
    duration_ms     INTEGER,                       -- condition must persist for this duration
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
        -- 'info', 'warning', 'critical', 'emergency'
    enabled         BOOLEAN NOT NULL DEFAULT true,
    cooldown_ms     INTEGER DEFAULT 300000,        -- minimum time between repeated alerts (5 min default)
    notification_channels TEXT[],                  -- array of channel IDs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_alert_severity CHECK (
        severity IN ('info', 'warning', 'critical', 'emergency')
    )
);

-- Alert event log
CREATE TABLE alert_events (
    id              BIGSERIAL PRIMARY KEY,
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    data_point_id   UUID REFERENCES data_points(id),
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id),
    triggered_at    TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    trigger_value   DOUBLE PRECISION,
    severity        VARCHAR(20) NOT NULL,
    message         TEXT,
    acknowledged_by UUID,                          -- user who acknowledged
    acknowledged_at TIMESTAMPTZ,
    notes           TEXT
);

CREATE INDEX idx_alerts_rule ON alert_events(alert_rule_id, triggered_at DESC);
CREATE INDEX idx_alerts_gateway ON alert_events(gateway_id, triggered_at DESC);
CREATE INDEX idx_alerts_unresolved ON alert_events(resolved_at) WHERE resolved_at IS NULL;
```

### 7. Data Pipeline Configuration

```sql
-- Visual dataflow pipeline definitions
CREATE TABLE data_pipelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Individual processing nodes within a pipeline
CREATE TABLE pipeline_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id     UUID NOT NULL REFERENCES data_pipelines(id) ON DELETE CASCADE,
    node_type       VARCHAR(50) NOT NULL,
        -- 'source', 'filter_threshold', 'filter_deadband', 'aggregate_window',
        -- 'transform_scale', 'transform_unit', 'ml_inference', 'sink_mqtt', 'sink_cloud', 'sink_local_store'
    label           VARCHAR(255),
    position_x      INTEGER,                       -- visual editor coordinates
    position_y      INTEGER,
    config_json     TEXT NOT NULL DEFAULT '{}',     -- node-specific configuration
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pipeline_nodes_pipeline ON pipeline_nodes(pipeline_id);

-- Connections between pipeline nodes (edges in the dataflow graph)
CREATE TABLE pipeline_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id     UUID NOT NULL REFERENCES data_pipelines(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES pipeline_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES pipeline_nodes(id) ON DELETE CASCADE,
    source_port     VARCHAR(50) NOT NULL DEFAULT 'output',
    target_port     VARCHAR(50) NOT NULL DEFAULT 'input',

    UNIQUE (pipeline_id, source_node_id, target_node_id, source_port, target_port)
);
```

### 8. Edge ML Model Management

```sql
-- ML model registry (cloud-side catalogue)
CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,
        -- 'anomaly_detection', 'predictive_maintenance', 'adaptive_filter', 'protocol_discovery'
    framework       VARCHAR(50) NOT NULL,          -- 'onnx', 'tflite', 'pytorch_mobile'
    target_arch     VARCHAR(20)[] NOT NULL,         -- ['arm64', 'armhf', 'x86_64']
    input_schema    TEXT,                          -- expected input tensor shape/type description
    output_schema   TEXT,                          -- output tensor description
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Versioned model artifacts
CREATE TABLE ml_model_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES ml_models(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,
    artifact_url    VARCHAR(1024) NOT NULL,        -- S3/blob/registry URL for the model file
    artifact_hash   VARCHAR(64) NOT NULL,          -- SHA-256 of the artifact
    artifact_size_bytes BIGINT NOT NULL,
    accuracy_metric DOUBLE PRECISION,              -- training accuracy or F1 score
    training_dataset_info TEXT,
    min_ram_mb      INTEGER,                       -- minimum gateway RAM to run this model
    inference_time_ms REAL,                        -- benchmarked inference time on reference hardware
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
        -- 'draft', 'testing', 'released', 'deprecated'
    released_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (model_id, version)
);

-- Deployments of models to specific gateways
CREATE TABLE ml_model_deployments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id    UUID NOT NULL REFERENCES ml_model_versions(id) ON DELETE RESTRICT,
    gateway_id          UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    deployed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    status              VARCHAR(20) NOT NULL DEFAULT 'deploying',
        -- 'deploying', 'active', 'inactive', 'failed', 'rolling_back'
    last_inference_at   TIMESTAMPTZ,
    inference_count     BIGINT DEFAULT 0,
    avg_inference_ms    REAL,
    error_message       TEXT,

    UNIQUE (model_version_id, gateway_id)
);

CREATE INDEX idx_ml_deploy_gateway ON ml_model_deployments(gateway_id);
```

### 9. OTA Update and Deployment Management

```sql
-- Software release packages
CREATE TABLE software_releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50) NOT NULL UNIQUE,
    release_type    VARCHAR(30) NOT NULL,           -- 'agent_core', 'protocol_adapter', 'ml_runtime', 'os_patch'
    target_arch     VARCHAR(20)[] NOT NULL,
    artifact_url    VARCHAR(1024) NOT NULL,
    artifact_hash   VARCHAR(64) NOT NULL,
    artifact_size_bytes BIGINT NOT NULL,
    release_notes   TEXT,
    min_agent_version VARCHAR(50),                  -- minimum existing version required
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
        -- 'draft', 'testing', 'released', 'recalled'
    released_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Rollout campaigns — staged OTA deployment to gateway fleets
CREATE TABLE rollout_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    release_id      UUID NOT NULL REFERENCES software_releases(id) ON DELETE RESTRICT,
    name            VARCHAR(255) NOT NULL,
    strategy        VARCHAR(30) NOT NULL DEFAULT 'staged',
        -- 'immediate', 'staged', 'canary'
    stage_pct       INTEGER[] DEFAULT '{10, 50, 100}',  -- percentage at each stage
    max_failure_pct REAL DEFAULT 5.0,               -- auto-halt if failure rate exceeds this
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- 'pending', 'in_progress', 'paused', 'completed', 'aborted'
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Per-gateway update status within a campaign
CREATE TABLE rollout_gateway_status (
    id              BIGSERIAL PRIMARY KEY,
    campaign_id     UUID NOT NULL REFERENCES rollout_campaigns(id) ON DELETE CASCADE,
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    stage           INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- 'pending', 'downloading', 'installing', 'verifying', 'completed', 'failed', 'rolled_back'
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    previous_version VARCHAR(50),

    UNIQUE (campaign_id, gateway_id)
);

CREATE INDEX idx_rollout_status_campaign ON rollout_gateway_status(campaign_id, status);
```

### 10. Security and Certificate Management

```sql
-- TLS certificates for mutual authentication
CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID REFERENCES edge_gateways(id) ON DELETE CASCADE,
    cert_type       VARCHAR(30) NOT NULL,          -- 'gateway_client', 'ca_root', 'ca_intermediate', 'mqtt_broker'
    subject_cn      VARCHAR(255) NOT NULL,
    issuer_cn       VARCHAR(255) NOT NULL,
    serial_number   VARCHAR(100) NOT NULL,
    not_before      TIMESTAMPTZ NOT NULL,
    not_after       TIMESTAMPTZ NOT NULL,
    fingerprint_sha256 VARCHAR(64) NOT NULL,
    public_key_pem  TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
        -- 'active', 'expired', 'revoked', 'pending_rotation'
    revoked_at      TIMESTAMPTZ,
    revocation_reason VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certs_gateway ON certificates(gateway_id);
CREATE INDEX idx_certs_expiry ON certificates(not_after) WHERE status = 'active';

-- Audit log for security-relevant operations
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_type      VARCHAR(20) NOT NULL,          -- 'user', 'gateway', 'system', 'api_key'
    actor_id        VARCHAR(255) NOT NULL,
    action          VARCHAR(100) NOT NULL,          -- 'gateway.provision', 'cert.rotate', 'model.deploy', 'config.push'
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     VARCHAR(255) NOT NULL,
    details         TEXT,
    ip_address      INET,
    success         BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_audit_actor ON audit_log(actor_id, timestamp DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id, timestamp DESC);
CREATE INDEX idx_audit_time ON audit_log(timestamp DESC);
```

### 11. Cloud Sync and Store-and-Forward Tracking

```sql
-- Tracks synchronization state between edge and cloud
CREATE TABLE sync_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    direction       VARCHAR(10) NOT NULL,          -- 'upload', 'download', 'bidirectional'
    records_sent    BIGINT DEFAULT 0,
    records_received BIGINT DEFAULT 0,
    bytes_transferred BIGINT DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
        -- 'in_progress', 'completed', 'failed', 'partial'
    error_message   TEXT
);

CREATE INDEX idx_sync_gateway ON sync_sessions(gateway_id, started_at DESC);

-- Data tiering policy per gateway or per data point
CREATE TABLE data_tiering_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID REFERENCES edge_gateways(id) ON DELETE CASCADE,
    data_point_id   UUID REFERENCES data_points(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    tier            VARCHAR(30) NOT NULL,
        -- 'raw_all', 'raw_critical_only', 'aggregate_5min', 'aggregate_1hr', 'alerts_only'
    bandwidth_limit_kbps INTEGER,                  -- optional bandwidth cap for this policy
    local_retention_hours INTEGER DEFAULT 168,     -- 7 days default local retention
    cloud_retention_days INTEGER DEFAULT 365,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    priority        SMALLINT NOT NULL DEFAULT 5,   -- lower = higher priority
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_tiering_target CHECK (
        gateway_id IS NOT NULL OR data_point_id IS NOT NULL
    )
);
```

### 12. User and Access Management

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(30) NOT NULL DEFAULT 'operator',
        -- 'admin', 'engineer', 'operator', 'viewer'
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Fine-grained site-level access control
CREATE TABLE user_site_access (
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    site_id     UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    role        VARCHAR(30) NOT NULL,              -- role at this specific site
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by  UUID REFERENCES users(id),

    PRIMARY KEY (user_id, site_id)
);

-- API keys for programmatic access
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    key_hash        VARCHAR(255) NOT NULL,         -- SHA-256 hash of the API key
    key_prefix      VARCHAR(10) NOT NULL,          -- first few characters for identification
    scopes          TEXT[] NOT NULL DEFAULT '{}',   -- 'telemetry:read', 'devices:write', 'fleet:admin'
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);
```

---

## Edge-Side SQLite Schema (Store-and-Forward Buffer)

On each gateway, a lightweight SQLite database provides the offline buffer:

```sql
-- Buffered telemetry awaiting cloud sync
CREATE TABLE telemetry_buffer (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    data_point_id   TEXT NOT NULL,                 -- UUID as text in SQLite
    recorded_at     TEXT NOT NULL,                  -- ISO 8601 timestamp
    value_numeric   REAL,
    value_string    TEXT,
    value_boolean   INTEGER,                       -- SQLite has no native boolean
    quality         INTEGER NOT NULL DEFAULT 192,
    sync_status     TEXT NOT NULL DEFAULT 'pending',-- 'pending', 'sent', 'confirmed'
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_buffer_sync ON telemetry_buffer(sync_status) WHERE sync_status = 'pending';
CREATE INDEX idx_buffer_time ON telemetry_buffer(recorded_at);

-- Buffered alert events awaiting sync
CREATE TABLE alert_buffer (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    alert_rule_id   TEXT NOT NULL,
    data_point_id   TEXT,
    triggered_at    TEXT NOT NULL,
    trigger_value   REAL,
    severity        TEXT NOT NULL,
    message         TEXT,
    sync_status     TEXT NOT NULL DEFAULT 'pending',
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Local configuration cache (last known good config from cloud)
CREATE TABLE config_cache (
    config_key      TEXT PRIMARY KEY,
    config_value    TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## Pros and Cons

### Pros

1. **Full referential integrity** — Foreign keys enforce consistency across the equipment hierarchy, preventing orphaned sensors or dangling references. This is critical when managing hundreds of gateways with thousands of devices.

2. **ISA-95 alignment** — The Organization > Site > Area > Device > Data Point hierarchy maps directly to ISA-95 levels, making the schema immediately comprehensible to OT engineers and compatible with MES/ERP integration.

3. **Mature tooling ecosystem** — PostgreSQL has decades of tooling for backup, replication, monitoring (pg_stat_statements), and migration. Operations teams can use standard DBA skills.

4. **Strong query flexibility** — Complex analytical queries (e.g., "show me all critical alerts across all sites where vibration exceeded threshold in the last 24 hours grouped by equipment type") are natural in SQL with JOINs.

5. **Partition-based retention** — Monthly partitions on telemetry tables allow dropping old data instantly by detaching partitions, without expensive DELETE operations.

6. **Transaction safety** — ACID transactions ensure that fleet-wide operations (staged rollouts, certificate rotations) are atomic and recoverable.

7. **TimescaleDB upgrade path** — If pure PostgreSQL partitioning becomes insufficient for telemetry volume, TimescaleDB can be layered on without changing the schema, converting telemetry tables to hypertables.

### Cons

1. **Telemetry write throughput ceiling** — At very high ingest rates (>100,000 readings/second across the fleet), PostgreSQL's row-based storage becomes a bottleneck compared to purpose-built time-series databases. Partitioning and connection pooling help but have limits.

2. **Schema rigidity for diverse protocols** — Industrial devices vary enormously. A Modbus temperature sensor and an OPC-UA CNC controller have very different metadata. The normalized schema requires either many nullable columns or a separate metadata table pattern that adds JOIN complexity.

3. **Storage efficiency** — Row-based storage with full indexing is 5-10x less space-efficient than columnar time-series formats for telemetry data. A fleet generating 50 million readings/day will accumulate terabytes faster than with InfluxDB or TimescaleDB compressed hypertables.

4. **Edge-cloud schema sync complexity** — Maintaining schema consistency between the cloud PostgreSQL instance and hundreds of edge SQLite instances requires careful migration orchestration. Schema changes must be rolled out to edge devices via OTA.

5. **No native time-series functions** — Window functions, gap filling, interpolation, and downsampling must be implemented in application code or SQL views, whereas dedicated time-series databases provide these out of the box.

6. **Connection overhead** — Each gateway maintaining a persistent PostgreSQL connection consumes server memory. At 1,000+ gateways, connection pooling with PgBouncer becomes mandatory.

---

## Migration and Scaling Considerations

### Vertical Scaling Path
- Start with a single PostgreSQL 16 instance with 32 GB RAM, NVMe storage
- Enable `pg_partman` for automated partition management on telemetry tables
- Use `pg_cron` for scheduled aggregate materialization
- Add read replicas for dashboard queries (operations dashboard reads from replica)

### Horizontal Scaling Path
- Shard by organization_id or site_id using Citus extension for PostgreSQL
- Each shard holds a complete slice of the hierarchy (site + gateways + devices + telemetry)
- Cross-shard queries (fleet-wide analytics) use Citus distributed SQL

### TimescaleDB Migration
- Convert telemetry_readings to a TimescaleDB hypertable: `SELECT create_hypertable('telemetry_readings', 'recorded_at', migrate_data => true);`
- Replace manual aggregate tables with TimescaleDB continuous aggregates
- Enable native compression for 10-20x storage reduction on historical telemetry
- This migration is non-destructive and can be done table-by-table

### Data Retention Strategy
- Raw telemetry: 30-90 days in hot storage (depending on customer tier)
- 5-minute aggregates: 1 year
- Hourly aggregates: 5 years
- Alert events and audit log: 7 years (compliance)
- Old partitions archived to object storage (S3/GCS) before drop

### Estimated Storage (1,000 gateways, 10 devices each, 5 sensors per device)
- 50,000 data points polled every second = 4.3 billion readings/day
- At ~100 bytes/row with indexes: ~430 GB/day raw
- With 90-day raw retention: ~39 TB hot storage
- This volume strongly suggests moving to TimescaleDB compression or a dedicated time-series approach for telemetry while keeping all other tables in standard PostgreSQL

---

## Technology Recommendations

1. **Use PostgreSQL as the system-of-record** for all non-telemetry data: fleet registry, device metadata, pipeline configs, ML models, OTA campaigns, users, certificates, and audit logs.

2. **Add TimescaleDB extension** for the telemetry_readings and aggregate tables from day one — the volume projections make this essential.

3. **Use SQLite on edge gateways** for the store-and-forward buffer — it runs on 128 MB ARM devices, requires no server process, and handles the write volumes of a single gateway easily.

4. **Deploy PgBouncer** in transaction pooling mode between the application and PostgreSQL to manage connection overhead from hundreds of gateway sync sessions.

5. **Implement Flyway migrations** with a version table that is also replicated to edge SQLite schemas, ensuring edge-cloud schema consistency during OTA updates.

6. **Consider Citus** for multi-tenant deployments where different organizations need data isolation and independent scaling.
