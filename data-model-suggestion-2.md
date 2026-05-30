# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Industrial IoT Edge Platform (452) · Generated: 2026-05-25

## Overview

This model treats every state change in the industrial IoT edge platform as an immutable event. The system uses Command Query Responsibility Segregation (CQRS) to separate the write path (event ingestion and command processing) from the read path (materialized views and projections optimized for specific queries). This is a natural fit for industrial telemetry because sensor readings are inherently append-only facts about the physical world, and the platform must maintain a complete, auditable history of all device states, configuration changes, and operational decisions.

The event store is the single source of truth. Read models are disposable projections that can be rebuilt from the event stream at any time. This provides exceptional auditability, temporal queries ("what was the state of this device at 3:47 AM?"), and the ability to add new analytics by replaying history.

---

## Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Event Store | PostgreSQL with append-only tables | Reliable, transactional, supports partitioning for retention |
| Event Bus | Apache Kafka or NATS JetStream | Ordered, durable event streaming between edge and cloud; Kafka for large scale, NATS for lighter deployments |
| Read Model DB | PostgreSQL (materialized views) | SQL-queryable projections for dashboards and APIs |
| Time-series Read Model | TimescaleDB or ClickHouse | Dedicated projection for telemetry analytics |
| Edge Event Buffer | SQLite + embedded NATS | Lightweight store-and-forward on constrained gateways |
| Projection Engine | Custom service (Go/Rust) | Subscribes to event streams and maintains read models |
| Snapshot Store | PostgreSQL | Periodic aggregate snapshots to avoid full replay |

---

## Event Store Schema

### Core Event Store Tables

```sql
-- The immutable event log — the single source of truth for the entire system
-- Every state change, sensor reading, configuration update, and operational action
-- is recorded as an event in this table.
CREATE TABLE events (
    event_id        BIGSERIAL,
    stream_id       VARCHAR(500) NOT NULL,         -- e.g. 'gateway:abc123', 'device:xyz789', 'campaign:def456'
    stream_type     VARCHAR(100) NOT NULL,          -- 'gateway', 'device', 'data_point', 'telemetry', 'pipeline', 'ml_model', 'rollout', 'certificate'
    event_type      VARCHAR(200) NOT NULL,          -- e.g. 'TelemetryRecorded', 'DeviceDiscovered', 'AlertTriggered'
    event_version   INTEGER NOT NULL DEFAULT 1,     -- schema version of the event payload
    sequence_num    BIGINT NOT NULL,                -- per-stream ordering (optimistic concurrency)
    occurred_at     TIMESTAMPTZ NOT NULL,           -- when the event actually happened (edge timestamp)
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- when it was persisted in the store
    correlation_id  UUID,                           -- links related events across streams
    causation_id    BIGINT,                         -- event_id of the event that caused this one
    actor_type      VARCHAR(20),                    -- 'gateway', 'user', 'system', 'scheduler'
    actor_id        VARCHAR(255),
    payload         JSONB NOT NULL,                 -- event-specific data
    metadata        JSONB DEFAULT '{}',             -- tracing, routing, edge origin info

    PRIMARY KEY (occurred_at, event_id),

    -- Optimistic concurrency: no two events in the same stream can share a sequence number
    CONSTRAINT uq_stream_sequence UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (occurred_at);

-- Partitioned monthly for retention management
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- Additional partitions created by automated job

-- Indexes for common access patterns
CREATE INDEX idx_events_stream ON events(stream_id, sequence_num);
CREATE INDEX idx_events_type ON events(event_type, occurred_at DESC);
CREATE INDEX idx_events_correlation ON events(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_stream_type_time ON events(stream_type, occurred_at DESC);

-- Snapshots to avoid replaying entire streams
-- Periodically captures the aggregated state of a stream
CREATE TABLE snapshots (
    stream_id       VARCHAR(500) NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    sequence_num    BIGINT NOT NULL,               -- snapshot is valid up to this sequence number
    snapshot_data   JSONB NOT NULL,                 -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (stream_id, sequence_num)
);

-- Projection checkpoints — tracks how far each read model has consumed
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(200) PRIMARY KEY,
    last_event_id   BIGINT NOT NULL,
    last_occurred_at TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'running',
        -- 'running', 'paused', 'rebuilding', 'error'
    error_message   TEXT
);

-- Dead letter queue for events that fail projection processing
CREATE TABLE dead_letter_events (
    id              BIGSERIAL PRIMARY KEY,
    event_id        BIGINT NOT NULL,
    projection_name VARCHAR(200) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
```

---

## Event Type Catalogue

### Gateway Lifecycle Events

```
Stream: gateway:{gateway_id}
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `GatewayProvisioned` | `{serial_number, hardware_model, cpu_arch, ram_mb, storage_mb, os_name, os_version, site_id, area_id}` | New gateway registered in the fleet |
| `GatewayConnected` | `{ip_address_wan, ip_address_lan, agent_version}` | Gateway established cloud connection |
| `GatewayDisconnected` | `{reason, last_heartbeat_at, wan_connected}` | Gateway lost cloud connection |
| `GatewayHeartbeatReceived` | `{cpu_percent, memory_used_mb, disk_used_mb, uptime_seconds, temperature_c, wan_latency_ms}` | Periodic health report |
| `GatewayConfigurationPushed` | `{config_version, config_diff, pushed_by}` | Configuration update sent to gateway |
| `GatewayConfigurationApplied` | `{config_version, applied_at}` | Gateway confirmed config applied |
| `GatewaySoftwareUpdated` | `{from_version, to_version, release_id}` | Agent software updated on gateway |
| `GatewaySoftwareUpdateFailed` | `{target_version, error, rolled_back_to}` | Update failed, possible rollback |
| `GatewayDecommissioned` | `{reason, decommissioned_by}` | Gateway removed from active fleet |
| `GatewayRelocated` | `{from_site_id, to_site_id, from_area_id, to_area_id}` | Gateway moved to different site/area |

### Device and Sensor Events

```
Stream: device:{device_id}
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `DeviceDiscovered` | `{adapter_instance_id, device_address, device_type, manufacturer, model, protocol}` | New device found via protocol scan |
| `DeviceRegistered` | `{name, area_id, isa95_equipment_class, metadata}` | Device formally registered by operator |
| `DeviceStatusChanged` | `{previous_status, new_status, reason}` | Device went online/offline/maintenance |
| `DataPointAdded` | `{data_point_id, name, protocol_address, data_type, unit_of_measure, scaling_factor}` | New sensor/register configured on device |
| `DataPointModified` | `{data_point_id, changes}` | Sensor configuration modified |
| `DataPointRemoved` | `{data_point_id, reason}` | Sensor removed from monitoring |
| `DeviceFirmwareDetected` | `{firmware_version, previous_version}` | Device firmware version change detected |

### Telemetry Events

```
Stream: telemetry:{data_point_id}:{date}  (daily streams to manage stream size)
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `TelemetryRecorded` | `{value_numeric, value_string, value_boolean, quality, gateway_id}` | Single sensor reading |
| `TelemetryBatchRecorded` | `{readings: [{data_point_id, value, quality, timestamp}]}` | Batch of readings (efficiency optimization) |
| `TelemetryQualityDegraded` | `{data_point_id, quality_code, previous_quality, reason}` | Signal quality dropped below threshold |
| `TelemetryGapDetected` | `{data_point_id, gap_start, gap_end, expected_interval_ms}` | Missing readings detected |
| `TelemetryBackfilled` | `{data_point_id, from, to, reading_count, source}` | Gap filled from store-and-forward buffer |

### Alert Events

```
Stream: alert:{alert_rule_id}
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `AlertRuleCreated` | `{name, data_point_id, condition_type, threshold, severity, notification_channels}` | New alert rule defined |
| `AlertRuleModified` | `{changes}` | Alert rule configuration changed |
| `AlertTriggered` | `{data_point_id, trigger_value, gateway_id, message}` | Alert condition met |
| `AlertResolved` | `{resolution_value, duration_ms}` | Alert condition cleared |
| `AlertAcknowledged` | `{acknowledged_by, notes}` | Operator acknowledged the alert |
| `AlertEscalated` | `{from_severity, to_severity, reason}` | Alert severity increased |

### Pipeline Configuration Events

```
Stream: pipeline:{pipeline_id}
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `PipelineCreated` | `{gateway_id, name, nodes: [], edges: []}` | New data pipeline defined |
| `PipelineNodeAdded` | `{node_id, node_type, config}` | Processing node added to pipeline |
| `PipelineNodeRemoved` | `{node_id}` | Node removed from pipeline |
| `PipelineEdgeAdded` | `{source_node_id, target_node_id, source_port, target_port}` | Connection added between nodes |
| `PipelineDeployed` | `{version, deployed_to_gateway}` | Pipeline pushed to edge gateway |
| `PipelineStarted` | `{version}` | Pipeline began processing on gateway |
| `PipelineStopped` | `{reason}` | Pipeline stopped on gateway |

### ML Model Lifecycle Events

```
Stream: ml_model:{model_id}
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `ModelRegistered` | `{name, model_type, framework, target_arch, input_schema, output_schema}` | New model added to registry |
| `ModelVersionPublished` | `{version, artifact_url, artifact_hash, accuracy_metric, min_ram_mb}` | New model version released |
| `ModelDeployedToGateway` | `{model_version_id, gateway_id}` | Model pushed to edge device |
| `ModelInferenceCompleted` | `{gateway_id, inference_time_ms, input_summary, output_summary, anomaly_detected}` | Single inference execution |
| `ModelDeploymentFailed` | `{gateway_id, error, resource_usage}` | Model could not run on gateway |

### OTA Rollout Events

```
Stream: rollout:{campaign_id}
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `RolloutCampaignCreated` | `{release_id, strategy, stage_pct, max_failure_pct, target_gateways}` | New OTA campaign defined |
| `RolloutStageStarted` | `{stage, percentage, gateway_count}` | Stage of rollout initiated |
| `RolloutGatewayStarted` | `{gateway_id, previous_version}` | Individual gateway update started |
| `RolloutGatewayCompleted` | `{gateway_id, duration_ms}` | Gateway updated successfully |
| `RolloutGatewayFailed` | `{gateway_id, error, rolled_back}` | Gateway update failed |
| `RolloutCampaignCompleted` | `{success_count, failure_count, duration_ms}` | Campaign finished |
| `RolloutCampaignAborted` | `{reason, failure_rate, aborted_by}` | Campaign stopped due to failures |

### Security Events

```
Stream: certificate:{cert_id}  |  Stream: security:{gateway_id}
```

| Event Type | Payload | Description |
|-----------|---------|-------------|
| `CertificateIssued` | `{gateway_id, cert_type, subject_cn, not_before, not_after, fingerprint}` | New certificate provisioned |
| `CertificateRotated` | `{gateway_id, old_fingerprint, new_fingerprint, new_not_after}` | Certificate renewed |
| `CertificateRevoked` | `{reason, revoked_by}` | Certificate invalidated |
| `SecurityIncidentDetected` | `{gateway_id, incident_type, details, severity}` | Anomalous security event |
| `AccessGranted` | `{user_id, resource_type, resource_id, role}` | User access provisioned |
| `AccessRevoked` | `{user_id, resource_type, resource_id, reason}` | User access removed |

---

## Read Model Projections

### Projection 1: Fleet Status View

Materialized from: `GatewayProvisioned`, `GatewayConnected`, `GatewayDisconnected`, `GatewayHeartbeatReceived`, `GatewaySoftwareUpdated`, `GatewayDecommissioned`

```sql
-- Current state of every gateway in the fleet
-- Rebuilt by replaying all gateway lifecycle events
CREATE TABLE rm_fleet_status (
    gateway_id          UUID PRIMARY KEY,
    serial_number       VARCHAR(100) NOT NULL,
    hardware_model      VARCHAR(100) NOT NULL,
    site_id             UUID NOT NULL,
    area_id             UUID,
    agent_version       VARCHAR(50),
    status              VARCHAR(20) NOT NULL,       -- derived from latest connect/disconnect events
    cpu_percent         REAL,
    memory_used_mb      REAL,
    disk_used_mb        REAL,
    temperature_c       REAL,
    wan_latency_ms      REAL,
    wan_connected       BOOLEAN,
    uptime_seconds      BIGINT,
    last_heartbeat_at   TIMESTAMPTZ,
    last_connected_at   TIMESTAMPTZ,
    last_disconnected_at TIMESTAMPTZ,
    provisioned_at      TIMESTAMPTZ,
    device_count        INTEGER DEFAULT 0,          -- maintained by DeviceDiscovered/DeviceRemoved
    data_point_count    INTEGER DEFAULT 0,
    active_alert_count  INTEGER DEFAULT 0,          -- maintained by AlertTriggered/AlertResolved
    last_event_seq      BIGINT NOT NULL             -- sequence number of last processed event
);

CREATE INDEX idx_rm_fleet_site ON rm_fleet_status(site_id);
CREATE INDEX idx_rm_fleet_status ON rm_fleet_status(status);
```

### Projection 2: Device Registry View

Materialized from: `DeviceDiscovered`, `DeviceRegistered`, `DeviceStatusChanged`, `DataPointAdded`, `DataPointModified`, `DataPointRemoved`

```sql
-- Current state of all registered industrial devices
CREATE TABLE rm_device_registry (
    device_id           UUID PRIMARY KEY,
    gateway_id          UUID NOT NULL,
    adapter_instance_id UUID NOT NULL,
    device_address      VARCHAR(255) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    device_type         VARCHAR(100),
    manufacturer        VARCHAR(255),
    model               VARCHAR(255),
    firmware_version    VARCHAR(50),
    protocol            VARCHAR(50) NOT NULL,
    area_id             UUID,
    isa95_equipment_class VARCHAR(100),
    status              VARCHAR(20) NOT NULL,
    discovered_at       TIMESTAMPTZ,
    last_seen_at        TIMESTAMPTZ,
    data_point_count    INTEGER DEFAULT 0,
    last_event_seq      BIGINT NOT NULL
);

CREATE INDEX idx_rm_devices_gateway ON rm_device_registry(gateway_id);

-- Current data point configurations
CREATE TABLE rm_data_points (
    data_point_id       UUID PRIMARY KEY,
    device_id           UUID NOT NULL,
    name                VARCHAR(255) NOT NULL,
    protocol_address    VARCHAR(255) NOT NULL,
    data_type           VARCHAR(30) NOT NULL,
    unit_of_measure     VARCHAR(50),
    scaling_factor      DOUBLE PRECISION,
    min_valid_value     DOUBLE PRECISION,
    max_valid_value     DOUBLE PRECISION,
    is_critical         BOOLEAN NOT NULL DEFAULT false,
    enabled             BOOLEAN NOT NULL DEFAULT true,
    latest_value        DOUBLE PRECISION,           -- updated on each TelemetryRecorded
    latest_value_at     TIMESTAMPTZ,
    latest_quality      SMALLINT,
    last_event_seq      BIGINT NOT NULL
);

CREATE INDEX idx_rm_dp_device ON rm_data_points(device_id);
```

### Projection 3: Telemetry Time-Series View

Materialized from: `TelemetryRecorded`, `TelemetryBatchRecorded`

```sql
-- This projection writes to a TimescaleDB hypertable for efficient time-series queries
-- Separate from the event store, optimized for range scans and aggregation
CREATE TABLE rm_telemetry (
    recorded_at     TIMESTAMPTZ NOT NULL,
    data_point_id   UUID NOT NULL,
    gateway_id      UUID NOT NULL,
    value_numeric   DOUBLE PRECISION,
    value_string    VARCHAR(500),
    value_boolean   BOOLEAN,
    quality         SMALLINT NOT NULL DEFAULT 192
);

-- Convert to hypertable (TimescaleDB)
SELECT create_hypertable('rm_telemetry', 'recorded_at',
    chunk_time_interval => INTERVAL '1 day');

-- Add compression policy
ALTER TABLE rm_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'data_point_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);
SELECT add_compression_policy('rm_telemetry', INTERVAL '7 days');

-- Continuous aggregate: 5-minute summaries
CREATE MATERIALIZED VIEW rm_telemetry_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', recorded_at) AS bucket,
    data_point_id,
    gateway_id,
    MIN(value_numeric) AS value_min,
    MAX(value_numeric) AS value_max,
    AVG(value_numeric) AS value_avg,
    STDDEV(value_numeric) AS value_stddev,
    COUNT(*) AS reading_count
FROM rm_telemetry
GROUP BY bucket, data_point_id, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('rm_telemetry_5min',
    start_offset => INTERVAL '1 hour',
    end_offset => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes');

-- Continuous aggregate: hourly summaries
CREATE MATERIALIZED VIEW rm_telemetry_1hr
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', recorded_at) AS bucket,
    data_point_id,
    gateway_id,
    MIN(value_numeric) AS value_min,
    MAX(value_numeric) AS value_max,
    AVG(value_numeric) AS value_avg,
    STDDEV(value_numeric) AS value_stddev,
    COUNT(*) AS reading_count
FROM rm_telemetry
GROUP BY bucket, data_point_id, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('rm_telemetry_1hr',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Retention: drop raw data after 90 days, keep aggregates longer
SELECT add_retention_policy('rm_telemetry', INTERVAL '90 days');
SELECT add_retention_policy('rm_telemetry_5min', INTERVAL '1 year');
SELECT add_retention_policy('rm_telemetry_1hr', INTERVAL '5 years');
```

### Projection 4: Active Alerts View

Materialized from: `AlertRuleCreated`, `AlertRuleModified`, `AlertTriggered`, `AlertResolved`, `AlertAcknowledged`

```sql
-- Currently active (unresolved) alerts
CREATE TABLE rm_active_alerts (
    alert_event_id      BIGINT PRIMARY KEY,        -- event_id of the AlertTriggered event
    alert_rule_id       UUID NOT NULL,
    rule_name           VARCHAR(255) NOT NULL,
    data_point_id       UUID,
    device_id           UUID,
    gateway_id          UUID NOT NULL,
    site_id             UUID NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    trigger_value       DOUBLE PRECISION,
    triggered_at        TIMESTAMPTZ NOT NULL,
    message             TEXT,
    acknowledged_by     VARCHAR(255),
    acknowledged_at     TIMESTAMPTZ,
    escalation_count    INTEGER DEFAULT 0,
    last_event_seq      BIGINT NOT NULL
);

CREATE INDEX idx_rm_alerts_severity ON rm_active_alerts(severity, triggered_at DESC);
CREATE INDEX idx_rm_alerts_site ON rm_active_alerts(site_id, triggered_at DESC);
CREATE INDEX idx_rm_alerts_gateway ON rm_active_alerts(gateway_id);

-- Alert history (all alerts including resolved)
CREATE TABLE rm_alert_history (
    alert_event_id      BIGINT PRIMARY KEY,
    alert_rule_id       UUID NOT NULL,
    rule_name           VARCHAR(255) NOT NULL,
    data_point_id       UUID,
    gateway_id          UUID NOT NULL,
    site_id             UUID NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    trigger_value       DOUBLE PRECISION,
    triggered_at        TIMESTAMPTZ NOT NULL,
    resolved_at         TIMESTAMPTZ,
    resolution_value    DOUBLE PRECISION,
    duration_ms         BIGINT,
    was_acknowledged    BOOLEAN DEFAULT false
) PARTITION BY RANGE (triggered_at);
```

### Projection 5: Rollout Campaign Tracker

Materialized from: `RolloutCampaignCreated`, `RolloutStageStarted`, `RolloutGatewayStarted`, `RolloutGatewayCompleted`, `RolloutGatewayFailed`, `RolloutCampaignCompleted`, `RolloutCampaignAborted`

```sql
CREATE TABLE rm_rollout_campaigns (
    campaign_id         UUID PRIMARY KEY,
    release_id          UUID NOT NULL,
    release_version     VARCHAR(50),
    name                VARCHAR(255) NOT NULL,
    strategy            VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    current_stage       INTEGER DEFAULT 0,
    total_gateways      INTEGER DEFAULT 0,
    pending_count       INTEGER DEFAULT 0,
    in_progress_count   INTEGER DEFAULT 0,
    completed_count     INTEGER DEFAULT 0,
    failed_count        INTEGER DEFAULT 0,
    rolled_back_count   INTEGER DEFAULT 0,
    failure_rate_pct    REAL DEFAULT 0.0,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    last_event_seq      BIGINT NOT NULL
);

CREATE TABLE rm_rollout_gateways (
    campaign_id     UUID NOT NULL,
    gateway_id      UUID NOT NULL,
    status          VARCHAR(20) NOT NULL,
    stage           INTEGER,
    previous_version VARCHAR(50),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_ms     BIGINT,
    error           TEXT,

    PRIMARY KEY (campaign_id, gateway_id)
);
```

---

## Edge-Side Event Buffer

Each gateway maintains a local event buffer in SQLite that mirrors the event store structure:

```sql
-- Local event buffer on the edge gateway
CREATE TABLE local_events (
    local_event_id  INTEGER PRIMARY KEY AUTOINCREMENT,
    stream_id       TEXT NOT NULL,
    stream_type     TEXT NOT NULL,
    event_type      TEXT NOT NULL,
    event_version   INTEGER NOT NULL DEFAULT 1,
    occurred_at     TEXT NOT NULL,                  -- ISO 8601
    payload         TEXT NOT NULL,                  -- JSON
    metadata        TEXT DEFAULT '{}',
    sync_status     TEXT NOT NULL DEFAULT 'pending',
        -- 'pending', 'sent', 'confirmed', 'failed'
    sync_attempts   INTEGER NOT NULL DEFAULT 0,
    last_sync_error TEXT,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_local_events_sync ON local_events(sync_status)
    WHERE sync_status IN ('pending', 'failed');
CREATE INDEX idx_local_events_stream ON local_events(stream_id, local_event_id);

-- Local snapshot cache for offline operation
CREATE TABLE local_snapshots (
    stream_id       TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    snapshot_data   TEXT NOT NULL,                  -- JSON
    sequence_num    INTEGER NOT NULL,
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Pending commands received from cloud during offline periods
CREATE TABLE pending_commands (
    command_id      TEXT PRIMARY KEY,
    command_type    TEXT NOT NULL,                  -- 'update_config', 'deploy_model', 'start_pipeline'
    payload         TEXT NOT NULL,
    received_at     TEXT NOT NULL,
    executed        INTEGER NOT NULL DEFAULT 0,    -- boolean
    result          TEXT
);
```

---

## Event Flow Architecture

```
Edge Gateway                          Cloud Platform
┌─────────────────┐                  ┌──────────────────────────────────┐
│ Protocol Adapter │                  │                                  │
│   (OPC-UA,       │──TelemetryRecorded──►│  Event Store (PostgreSQL)        │
│    Modbus, etc.) │                  │  ┌─────────────────────────────┐ │
│                  │                  │  │ events table (append-only)  │ │
│ Edge Processing  │                  │  └─────────────┬───────────────┘ │
│  Engine          │──AlertTriggered──►│                │                │
│                  │                  │  ┌─────────────▼───────────────┐ │
│ Local Event      │                  │  │ Event Bus (Kafka/NATS)      │ │
│  Buffer (SQLite) │◄──sync──────────►│  └──┬──────┬──────┬──────┬────┘ │
│                  │                  │     │      │      │      │      │
│ ML Inference     │                  │  ┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐  │
│  Runtime         │──ModelInference──►│  │Fleet││Devce││Telem││Alert│  │
│                  │                  │  │Proj.││Proj.││Proj.││Proj.│  │
└─────────────────┘                  │  └──┬──┘└──┬──┘└──┬──┘└──┬──┘  │
                                     │     │      │      │      │      │
                                     │  ┌──▼──────▼──────▼──────▼──┐   │
                                     │  │  Read Model Database(s)  │   │
                                     │  │  (PostgreSQL/TimescaleDB) │   │
                                     │  └──────────────────────────┘   │
                                     │                                  │
                                     │  REST/GraphQL API ◄── Dashboard  │
                                     └──────────────────────────────────┘
```

---

## Command Handlers

Commands are validated and converted into events. Here are the key command-to-event mappings:

```
Command: ProvisionGateway
  Validates: serial_number unique, site exists, hardware specs meet minimum
  Emits: GatewayProvisioned
  Side effects: generates initial certificate, creates edge config package

Command: RegisterDevice
  Validates: adapter running, device_address reachable, no duplicate
  Emits: DeviceRegistered
  Side effects: starts polling data points

Command: ConfigureDataPoint
  Validates: device exists, protocol_address valid for adapter type, data_type compatible
  Emits: DataPointAdded
  Side effects: adds to polling schedule on gateway

Command: IngestTelemetry (from edge gateway)
  Validates: data_point_id exists, value within physical bounds, timestamp not in future
  Emits: TelemetryRecorded (or TelemetryBatchRecorded)
  Side effects: none (projections handle downstream logic)

Command: CreateAlertRule
  Validates: data_point or device exists, thresholds sensible, channels valid
  Emits: AlertRuleCreated
  Side effects: deploys rule to edge gateway for local evaluation

Command: DeployModel
  Validates: model version exists, gateway meets hardware requirements, storage available
  Emits: ModelDeployedToGateway
  Side effects: initiates model transfer to gateway

Command: StartRolloutCampaign
  Validates: release exists, target gateways meet prerequisites, no conflicting campaign
  Emits: RolloutCampaignCreated, then RolloutStageStarted
  Side effects: begins staged deployment process
```

---

## Snapshot Strategy

To avoid replaying millions of events for frequently-accessed aggregates:

```sql
-- Snapshot every 1,000 events per gateway stream
-- Example snapshot for a gateway stream:
INSERT INTO snapshots (stream_id, stream_type, sequence_num, snapshot_data)
VALUES (
    'gateway:abc-123',
    'gateway',
    15000,
    '{
        "gateway_id": "abc-123",
        "serial_number": "RBT-1520-00142",
        "hardware_model": "Robustel R1520",
        "status": "online",
        "agent_version": "2.4.1",
        "site_id": "site-xyz",
        "area_id": "area-001",
        "last_heartbeat_at": "2026-05-25T14:30:00Z",
        "device_count": 12,
        "data_point_count": 87,
        "active_alert_count": 2,
        "provisioned_at": "2025-01-15T10:00:00Z"
    }'
);

-- Rebuilding state: load snapshot, then replay events after snapshot's sequence_num
-- SELECT * FROM snapshots WHERE stream_id = 'gateway:abc-123' ORDER BY sequence_num DESC LIMIT 1;
-- SELECT * FROM events WHERE stream_id = 'gateway:abc-123' AND sequence_num > 15000 ORDER BY sequence_num;
```

For telemetry streams, snapshots are not used because the read model is a time-series projection that is queried by time range, not by current state. Instead, the projection checkpoint tracks progress.

---

## Pros and Cons

### Pros

1. **Complete audit trail** — Every state change is preserved as an immutable event. For regulated industries (energy, utilities, pharmaceuticals), this provides compliance-ready audit history without additional logging infrastructure. You can answer "what was the state of gateway X at 3:47 AM on March 15?" by replaying events up to that timestamp.

2. **Natural fit for telemetry** — Sensor readings are inherently append-only facts. Event sourcing does not fight the data's natural shape — it embraces it. There is no impedance mismatch between "what happened" and "how we store it."

3. **Temporal queries and replay** — Replay enables powerful debugging: when an anomaly detection model produces unexpected results, engineers can replay the exact input telemetry stream to understand why. This is invaluable for commissioning and tuning edge configurations.

4. **Independent read model evolution** — New analytics requirements (e.g., "we need vibration frequency analysis across all pumps") can be satisfied by adding a new projection that replays existing telemetry events. No schema migration, no data backfill — the data is already there.

5. **Edge-cloud synchronization resilience** — The edge buffer is just a local event log. Sync is simply "send all events with sync_status = pending." Ordering is preserved by sequence numbers. Conflict resolution is trivial because events are facts about what happened — there are no conflicting updates.

6. **Decoupled write and read scaling** — The event store (write path) and read models (query path) scale independently. High-frequency telemetry ingestion does not compete with dashboard queries for database resources.

7. **Fault isolation** — If a projection fails, the event store is unaffected. The projection can be repaired and rebuilt from the event log without data loss. This is critical for a platform managing industrial infrastructure.

### Cons

1. **Complexity overhead** — Event sourcing requires building and maintaining projection engines, snapshot logic, dead-letter handling, and checkpoint management. This is significantly more complex than a CRUD application and demands team expertise in event-driven architecture.

2. **Eventual consistency** — Read models are updated asynchronously after events are persisted. Dashboards may show stale data (typically milliseconds to seconds of lag). For a fleet management dashboard this is acceptable, but operators expecting real-time must understand the model.

3. **Event schema evolution** — When the payload schema of an event changes (e.g., adding a field to `TelemetryRecorded`), all projections must handle both old and new versions. This requires versioned event schemas and upcasting logic that adds maintenance burden.

4. **Telemetry volume challenge** — A fleet of 1,000 gateways with 50,000 data points at 1 Hz produces 4.3 billion events per day. Storing every reading as an individual event in the event store is expensive. The practical approach is to batch telemetry events (`TelemetryBatchRecorded`) and use the telemetry projection as the primary query surface, treating the raw events as archival.

5. **Debugging difficulty** — When a read model shows unexpected data, tracing the issue back through the event stream to find the causative event requires tooling that does not exist out of the box. Event store browsers and correlation tracers must be built.

6. **Storage costs** — The event store retains everything forever (or for long retention periods). Combined with read model storage, total storage is higher than a mutable-state approach. Monthly partitioning and archival to object storage mitigates this.

7. **Snapshot management** — Without snapshots, rebuilding the state of a long-lived gateway stream (years of events) becomes prohibitively slow. Snapshot frequency must be tuned per stream type, adding operational complexity.

---

## Migration and Scaling Considerations

### Starting Small
- Begin with PostgreSQL as both event store and read model database
- Use a single application service as the projection engine
- Kafka/NATS can be deferred — internal pub/sub (PostgreSQL LISTEN/NOTIFY) works for low volumes
- Snapshots can be deferred until streams exceed 10,000 events

### Growing to Production Scale
- Introduce Kafka or NATS JetStream as the event bus between event store and projections
- Run separate projection services for each read model (fleet, devices, telemetry, alerts)
- Move telemetry projection to dedicated TimescaleDB instance
- Deploy event store on a separate PostgreSQL cluster from read models

### Large-Scale Fleet (10,000+ gateways)
- Partition Kafka topics by site or organization for parallelism
- Shard the event store by stream_type (separate partitions for telemetry vs. lifecycle events)
- Use ClickHouse or another columnar store for fleet-wide analytical projections
- Consider Apache Flink or Kafka Streams for stateful stream processing of telemetry aggregations

### Event Store Retention
- Lifecycle events (gateway, device, security): retain indefinitely — these are low-volume and audit-critical
- Telemetry events: retain 30-90 days in the event store; the time-series projection provides long-term access
- Alert events: retain 7 years (regulatory compliance in energy sector)
- Archive old event partitions to object storage (Parquet format) for cold replay capability

### Rebuilding Projections
- Small projections (fleet status, device registry): rebuild from snapshots + recent events in minutes
- Telemetry projection: rebuild from archived events can take hours for large datasets; consider maintaining dual projections (active + standby) for zero-downtime rebuilds
- Always maintain projection checkpoints so that a crashed projector resumes from its last position rather than replaying everything

---

## Technology Recommendations

1. **Use PostgreSQL as the event store** for simplicity and transactional guarantees. The append-only pattern works well with PostgreSQL's WAL-based replication and partitioning.

2. **Deploy NATS JetStream** as the event bus for small-to-medium deployments (under 5,000 gateways). It is lighter than Kafka, supports durable streams, and has excellent Go/Rust client libraries suitable for edge runtimes.

3. **Upgrade to Kafka** for large-scale deployments where topic partitioning, consumer groups, and exactly-once semantics become important.

4. **Use TimescaleDB for the telemetry projection** — this is the highest-volume read model and benefits enormously from hypertable compression and continuous aggregates.

5. **Batch telemetry events** on the edge gateway (collect 100-1000 readings, emit a single `TelemetryBatchRecorded` event) to reduce event count by 2-3 orders of magnitude while preserving individual reading granularity in the payload.

6. **Implement event versioning from day one** using the `event_version` field. Use an upcaster pattern in projection engines to transform old event formats to current format before processing.

7. **Build a projection health dashboard** that monitors checkpoint lag, dead-letter queue depth, and rebuild status. This is operational tooling that event-sourced systems require from the start.
