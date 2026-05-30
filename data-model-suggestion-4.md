# Data Model Suggestion 4: Time-Series + Graph Hybrid (TimescaleDB + Apache AGE)

> Project: Industrial IoT Edge Platform (452) · Generated: 2026-05-25

## Overview

This model uses a domain-specific combination of two specialized storage paradigms optimized for the two core data shapes in industrial IoT:

1. **Time-series storage (TimescaleDB)** for high-frequency telemetry, gateway health metrics, and alert event history — the dominant data volume in any IIoT platform.

2. **Graph storage (Apache AGE on PostgreSQL)** for the equipment hierarchy, device relationships, protocol topology, data pipeline flows, and digital twin connectivity — the dominant data complexity.

The key insight is that industrial IoT platforms are fundamentally about **relationships between physical things** (which equipment connects to which gateway via which protocol, which sensor feeds which pipeline node, which alert rule monitors which data points across which sites) AND about **time-stamped measurements from those things**. Neither a pure relational model nor a pure time-series model handles both well. This hybrid uses each paradigm where it excels.

Apache AGE (A Graph Extension) runs inside PostgreSQL, meaning both the graph model and the time-series model share the same database engine. This eliminates cross-database synchronization complexity while providing native Cypher query language for graph traversals and native SQL + TimescaleDB functions for time-series analytics.

---

## Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Database Engine | PostgreSQL 16+ | Single engine hosts both AGE and TimescaleDB |
| Graph Extension | Apache AGE 1.5+ | Cypher query language on PostgreSQL; no separate graph DB needed |
| Time-Series Extension | TimescaleDB 2.x | Hypertables, compression, continuous aggregates for telemetry |
| Edge-Local DB | SQLite 3 + JSON1 | Lightweight store-and-forward buffer |
| Visualization | Grafana (time-series) + custom graph viewer | Grafana for telemetry dashboards; D3.js or vis.js for topology visualization |
| Query API | GraphQL | Natural fit for graph + time-series queries with nested relationships |

### Why Apache AGE over Neo4j?

- **Single database process**: AGE runs inside PostgreSQL alongside TimescaleDB. No separate Neo4j instance to manage, back up, monitor, or secure.
- **Transactional consistency**: Graph mutations and time-series inserts can share a PostgreSQL transaction.
- **Lower operational cost**: No additional database license, no separate cluster, no cross-database sync logic.
- **SQL interoperability**: AGE graph queries can be combined with standard SQL in the same statement, enabling queries that join graph traversals with time-series aggregations.

---

## Graph Schema (Apache AGE / Cypher)

### Graph Creation

```sql
-- Load AGE extension
CREATE EXTENSION IF NOT EXISTS age;
LOAD 'age';
SET search_path = ag_catalog, "$user", public;

-- Create the industrial IoT graph
SELECT create_graph('iiot');
```

### Vertex (Node) Labels

```cypher
-- Organization hierarchy
// Organization node
CREATE (:Organization {
    id: 'org-uuid',
    name: 'Acme Industrial',
    slug: 'acme-industrial',
    industry_sector: 'manufacturing',
    default_timezone: 'America/Chicago',
    created_at: '2026-01-15T10:00:00Z'
})

// Site node
CREATE (:Site {
    id: 'site-uuid',
    name: 'Houston Refinery',
    slug: 'houston-refinery',
    timezone: 'America/Chicago',
    latitude: 29.7604,
    longitude: -95.3698,
    facility_type: 'refinery',
    isa95_level: 3,
    regulatory_zone: 'NERC-CIP',
    created_at: '2026-01-15T10:00:00Z'
})

// Area node (supports nesting via CONTAINS_AREA relationship)
CREATE (:Area {
    id: 'area-uuid',
    name: 'Distillation Unit A',
    slug: 'distillation-unit-a',
    area_type: 'production_line',
    zone_classification: 'hazardous_zone_1',
    isa95_work_cell: 'distillation_train_1',
    created_at: '2026-01-15T10:00:00Z'
})

// Edge Gateway node
CREATE (:Gateway {
    id: 'gw-uuid',
    serial_number: 'RBT-1520-00142',
    hardware_model: 'Robustel R1520',
    cpu_architecture: 'arm64',
    ram_mb: 256,
    storage_mb: 8192,
    os_name: 'linux',
    os_version: '6.1.0',
    agent_version: '2.4.1',
    status: 'online',
    ip_lan: '192.168.1.100',
    ip_wan: '203.0.113.42',
    last_heartbeat_at: '2026-05-25T14:30:00Z',
    provisioned_at: '2025-06-01T08:00:00Z'
})

// Protocol Adapter node
CREATE (:ProtocolAdapter {
    id: 'adapter-uuid',
    protocol: 'modbus_tcp',
    name: 'Modbus - Pump Controllers',
    connection_host: '192.168.1.50',
    connection_port: 502,
    poll_interval_ms: 1000,
    status: 'running',
    enabled: true
})

// Industrial Device node
CREATE (:Device {
    id: 'device-uuid',
    name: 'Pump #3 VFD',
    device_type: 'vfd',
    manufacturer: 'ABB',
    model: 'ACS880',
    firmware_version: '7.41',
    serial_number: 'ABB-VFD-00456',
    device_address: '{"unit_id": 3}',
    isa95_equipment_class: 'variable_frequency_drive',
    status: 'active',
    discovered_at: '2025-06-01T09:00:00Z'
})

// Data Point (Sensor/Register) node
CREATE (:DataPoint {
    id: 'dp-uuid',
    name: 'motor_bearing_temperature',
    display_name: 'Motor Bearing Temperature',
    protocol_address: '{"register_type": "holding", "address": 40001, "count": 2}',
    data_type: 'float32',
    unit_of_measure: 'celsius',
    scaling_factor: 0.1,
    scaling_offset: -40.0,
    min_valid_value: -50.0,
    max_valid_value: 200.0,
    deadband_value: 0.5,
    is_critical: true,
    is_writable: false,
    isa95_property: 'Equipment.Temperature.ProcessValue'
})

// ML Model node
CREATE (:MLModel {
    id: 'model-uuid',
    name: 'Pump Anomaly Detector',
    model_type: 'anomaly_detection',
    framework: 'onnx',
    version: '1.2.0',
    accuracy: 0.94,
    min_ram_mb: 64,
    inference_time_ms: 38.2
})

// Alert Rule node
CREATE (:AlertRule {
    id: 'rule-uuid',
    name: 'Bearing Overtemperature',
    condition_type: 'threshold_high',
    threshold_value: 85.0,
    duration_ms: 5000,
    severity: 'critical',
    enabled: true
})

// Pipeline node (represents an entire dataflow pipeline)
CREATE (:Pipeline {
    id: 'pipeline-uuid',
    name: 'Pump Monitoring Pipeline',
    version: 3,
    enabled: true
})

// Pipeline Processing Node (individual nodes in the dataflow)
CREATE (:PipelineNode {
    id: 'pnode-uuid',
    node_type: 'filter_deadband',
    label: 'Deadband Filter',
    config: '{"deadband_value": 0.5, "deadband_type": "absolute"}'
})

// Software Release node
CREATE (:SoftwareRelease {
    id: 'release-uuid',
    version: '2.4.1',
    release_type: 'agent_core',
    artifact_hash: 'sha256:abc123...',
    status: 'released',
    released_at: '2026-05-01T00:00:00Z'
})

// Certificate node
CREATE (:Certificate {
    id: 'cert-uuid',
    cert_type: 'gateway_client',
    subject_cn: 'gw-rbt1520-00142.iot.acme.com',
    fingerprint_sha256: 'A1B2C3...',
    not_before: '2026-01-01T00:00:00Z',
    not_after: '2027-01-01T00:00:00Z',
    status: 'active'
})
```

### Edge (Relationship) Types

```cypher
// Organizational hierarchy
(:Organization)-[:OWNS_SITE]->(:Site)
(:Site)-[:CONTAINS_AREA]->(:Area)
(:Area)-[:CONTAINS_AREA]->(:Area)               // nested areas

// Gateway deployment
(:Site)-[:DEPLOYS_GATEWAY]->(:Gateway)
(:Area)-[:HOSTS_GATEWAY]->(:Gateway)

// Protocol connectivity
(:Gateway)-[:RUNS_ADAPTER]->(:ProtocolAdapter)
(:ProtocolAdapter)-[:CONNECTS_TO]->(:Device)

// Device hierarchy and data points
(:Area)-[:CONTAINS_DEVICE]->(:Device)
(:Device)-[:HAS_DATA_POINT]->(:DataPoint)

// Alert monitoring
(:AlertRule)-[:MONITORS]->(:DataPoint)
(:AlertRule)-[:MONITORS_DEVICE]->(:Device)

// ML model deployment
(:MLModel)-[:DEPLOYED_ON {deployed_at, status, inference_count}]->(:Gateway)
(:MLModel)-[:ANALYZES]->(:DataPoint)

// Pipeline data flow
(:Gateway)-[:RUNS_PIPELINE]->(:Pipeline)
(:Pipeline)-[:HAS_NODE]->(:PipelineNode)
(:PipelineNode)-[:FEEDS {source_port, target_port}]->(:PipelineNode)
(:DataPoint)-[:SOURCE_FOR]->(:PipelineNode)
(:PipelineNode)-[:SINKS_TO {destination_type}]->(:Gateway)    // local store
// Note: pipeline data flow creates a directed acyclic graph (DAG)

// Software and security
(:SoftwareRelease)-[:INSTALLED_ON {installed_at, previous_version}]->(:Gateway)
(:Certificate)-[:AUTHENTICATES]->(:Gateway)
(:Certificate)-[:ISSUED_BY]->(:Certificate)      // CA chain

// Device-to-device relationships (digital twin / physical topology)
(:Device)-[:DRIVES {coupling_type}]->(:Device)           // motor drives pump
(:Device)-[:FEEDS_INTO {pipe_diameter_mm}]->(:Device)     // pump feeds heat exchanger
(:Device)-[:MONITORS_EQUIPMENT]->(:Device)                // sensor monitors motor
(:Device)-[:POWERED_BY {circuit_id}]->(:Device)           // device powered by panel
(:DataPoint)-[:CORRELATES_WITH {correlation_coefficient}]->(:DataPoint)  // statistical correlation
```

### Example Graph Queries

```cypher
-- 1. Find all data points on devices connected to a specific gateway,
--    traversing through protocol adapters
MATCH (gw:Gateway {serial_number: 'RBT-1520-00142'})
      -[:RUNS_ADAPTER]->(adapter:ProtocolAdapter)
      -[:CONNECTS_TO]->(device:Device)
      -[:HAS_DATA_POINT]->(dp:DataPoint)
WHERE dp.is_critical = true
RETURN device.name, dp.name, dp.unit_of_measure, adapter.protocol

-- 2. Find the complete physical topology upstream and downstream of a pump
MATCH path = (upstream)-[*1..5]->(pump:Device {name: 'Pump #3 VFD'})-[*1..5]->(downstream)
WHERE ALL(r IN relationships(path) WHERE type(r) IN ['DRIVES', 'FEEDS_INTO', 'POWERED_BY'])
RETURN path

-- 3. Impact analysis: what is affected if gateway goes offline?
MATCH (gw:Gateway {id: 'gw-uuid'})
OPTIONAL MATCH (gw)-[:RUNS_ADAPTER]->(adapter)-[:CONNECTS_TO]->(device)-[:HAS_DATA_POINT]->(dp)
OPTIONAL MATCH (rule:AlertRule)-[:MONITORS]->(dp)
OPTIONAL MATCH (model:MLModel)-[:DEPLOYED_ON]->(gw)
OPTIONAL MATCH (pipeline:Pipeline)<-[:RUNS_PIPELINE]-(gw)
RETURN gw.serial_number,
       collect(DISTINCT device.name) AS affected_devices,
       count(DISTINCT dp) AS affected_data_points,
       collect(DISTINCT rule.name) AS affected_alert_rules,
       collect(DISTINCT model.name) AS affected_ml_models,
       collect(DISTINCT pipeline.name) AS affected_pipelines

-- 4. Find the data pipeline path from sensor to cloud
MATCH path = (dp:DataPoint {name: 'motor_bearing_temperature'})
             -[:SOURCE_FOR]->(first:PipelineNode)
             -[:FEEDS*1..10]->(last:PipelineNode)
WHERE last.node_type STARTS WITH 'sink_'
RETURN [node IN nodes(path) | node.label] AS pipeline_path,
       last.node_type AS destination

-- 5. Certificate chain verification
MATCH chain = (leaf:Certificate {gateway_id: 'gw-uuid'})
              -[:ISSUED_BY*1..5]->(root:Certificate)
WHERE root.cert_type = 'ca_root'
RETURN [cert IN nodes(chain) | cert.subject_cn] AS cert_chain,
       ALL(cert IN nodes(chain) WHERE cert.status = 'active') AS all_valid

-- 6. Find devices with correlated sensor anomalies (digital twin insight)
MATCH (dp1:DataPoint)-[c:CORRELATES_WITH]->(dp2:DataPoint)
WHERE c.correlation_coefficient > 0.8
MATCH (dp1)<-[:HAS_DATA_POINT]-(d1:Device)
MATCH (dp2)<-[:HAS_DATA_POINT]-(d2:Device)
RETURN d1.name, dp1.name, d2.name, dp2.name, c.correlation_coefficient
ORDER BY c.correlation_coefficient DESC

-- 7. Full ISA-95 hierarchy traversal
MATCH path = (org:Organization)-[:OWNS_SITE]->(site:Site)
             -[:CONTAINS_AREA]->(area:Area)
             -[:CONTAINS_DEVICE]->(device:Device)
             -[:HAS_DATA_POINT]->(dp:DataPoint)
WHERE org.slug = 'acme-industrial'
RETURN org.name, site.name, area.name, device.name, dp.name, dp.unit_of_measure
ORDER BY site.name, area.name, device.name
```

---

## Time-Series Schema (TimescaleDB)

### Telemetry Hypertables

```sql
-- Core telemetry readings — the highest-volume table
CREATE TABLE telemetry_readings (
    recorded_at     TIMESTAMPTZ NOT NULL,
    data_point_id   UUID NOT NULL,
    gateway_id      UUID NOT NULL,
    value_numeric   DOUBLE PRECISION,
    value_string    VARCHAR(500),
    value_boolean   BOOLEAN,
    quality         SMALLINT NOT NULL DEFAULT 192   -- OPC-UA quality code
);

SELECT create_hypertable('telemetry_readings', 'recorded_at',
    chunk_time_interval => INTERVAL '1 day',
    create_default_indexes => false);

-- Optimized indexes for common access patterns
CREATE INDEX idx_telemetry_dp_time ON telemetry_readings(data_point_id, recorded_at DESC);
CREATE INDEX idx_telemetry_gw_time ON telemetry_readings(gateway_id, recorded_at DESC);

-- Enable columnar compression for historical data
ALTER TABLE telemetry_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'data_point_id,gateway_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);
SELECT add_compression_policy('telemetry_readings', INTERVAL '7 days');

-- Multi-value telemetry for complex sensors (vibration, power meters)
CREATE TABLE telemetry_extended (
    recorded_at     TIMESTAMPTZ NOT NULL,
    data_point_id   UUID NOT NULL,
    gateway_id      UUID NOT NULL,
    values          JSONB NOT NULL,
        -- Vibration: {"x_mm_s": 2.4, "y_mm_s": 1.8, "z_mm_s": 3.1, "freq_hz": 47.2}
        -- Power: {"v_l1": 231.2, "v_l2": 230.8, "i_l1": 12.4, "pf": 0.92, "thd": 3.2}
    quality         SMALLINT NOT NULL DEFAULT 192
);

SELECT create_hypertable('telemetry_extended', 'recorded_at',
    chunk_time_interval => INTERVAL '1 day');

ALTER TABLE telemetry_extended SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'data_point_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);
SELECT add_compression_policy('telemetry_extended', INTERVAL '7 days');

-- Gateway health metrics
CREATE TABLE gateway_health (
    recorded_at     TIMESTAMPTZ NOT NULL,
    gateway_id      UUID NOT NULL,
    cpu_percent     REAL,
    memory_used_mb  REAL,
    disk_used_mb    REAL,
    wan_connected   BOOLEAN,
    wan_latency_ms  REAL,
    temperature_c   REAL,
    uptime_seconds  BIGINT,
    buffer_depth    INTEGER                         -- store-and-forward queue depth
);

SELECT create_hypertable('gateway_health', 'recorded_at',
    chunk_time_interval => INTERVAL '1 day');

ALTER TABLE gateway_health SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'gateway_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);
SELECT add_compression_policy('gateway_health', INTERVAL '3 days');
```

### Continuous Aggregates

```sql
-- 5-minute telemetry summaries
CREATE MATERIALIZED VIEW telemetry_5min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', recorded_at) AS bucket,
    data_point_id,
    gateway_id,
    MIN(value_numeric) AS val_min,
    MAX(value_numeric) AS val_max,
    AVG(value_numeric) AS val_avg,
    STDDEV(value_numeric) AS val_stddev,
    COUNT(*) AS reading_count,
    -- First and last values in the window (useful for cumulative counters)
    first(value_numeric, recorded_at) AS val_first,
    last(value_numeric, recorded_at) AS val_last
FROM telemetry_readings
GROUP BY bucket, data_point_id, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('telemetry_5min',
    start_offset => INTERVAL '1 hour',
    end_offset => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes');

-- Hourly summaries
CREATE MATERIALIZED VIEW telemetry_1hr
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', recorded_at) AS bucket,
    data_point_id,
    gateway_id,
    MIN(value_numeric) AS val_min,
    MAX(value_numeric) AS val_max,
    AVG(value_numeric) AS val_avg,
    STDDEV(value_numeric) AS val_stddev,
    COUNT(*) AS reading_count,
    first(value_numeric, recorded_at) AS val_first,
    last(value_numeric, recorded_at) AS val_last
FROM telemetry_readings
GROUP BY bucket, data_point_id, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('telemetry_1hr',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Daily summaries for long-term trending
CREATE MATERIALIZED VIEW telemetry_1day
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', recorded_at) AS bucket,
    data_point_id,
    gateway_id,
    MIN(value_numeric) AS val_min,
    MAX(value_numeric) AS val_max,
    AVG(value_numeric) AS val_avg,
    STDDEV(value_numeric) AS val_stddev,
    COUNT(*) AS reading_count,
    first(value_numeric, recorded_at) AS val_first,
    last(value_numeric, recorded_at) AS val_last
FROM telemetry_readings
GROUP BY bucket, data_point_id, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('telemetry_1day',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- Gateway health: hourly averages
CREATE MATERIALIZED VIEW gateway_health_1hr
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', recorded_at) AS bucket,
    gateway_id,
    AVG(cpu_percent) AS avg_cpu,
    MAX(cpu_percent) AS max_cpu,
    AVG(memory_used_mb) AS avg_memory,
    AVG(wan_latency_ms) AS avg_latency,
    MAX(wan_latency_ms) AS max_latency,
    COUNT(*) FILTER (WHERE wan_connected = false) AS disconnect_count,
    MAX(buffer_depth) AS max_buffer_depth
FROM gateway_health
GROUP BY bucket, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('gateway_health_1hr',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Tiered retention
SELECT add_retention_policy('telemetry_readings', INTERVAL '90 days');
SELECT add_retention_policy('telemetry_extended', INTERVAL '90 days');
SELECT add_retention_policy('telemetry_5min', INTERVAL '1 year');
SELECT add_retention_policy('telemetry_1hr', INTERVAL '3 years');
SELECT add_retention_policy('telemetry_1day', INTERVAL '10 years');
SELECT add_retention_policy('gateway_health', INTERVAL '30 days');
SELECT add_retention_policy('gateway_health_1hr', INTERVAL '2 years');
```

### Alert Event History

```sql
CREATE TABLE alert_events (
    triggered_at    TIMESTAMPTZ NOT NULL,
    alert_rule_id   UUID NOT NULL,
    data_point_id   UUID,
    gateway_id      UUID NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    trigger_value   DOUBLE PRECISION,
    resolved_at     TIMESTAMPTZ,
    duration_ms     BIGINT,
    message         TEXT,
    acknowledged_by VARCHAR(255),
    acknowledged_at TIMESTAMPTZ
);

SELECT create_hypertable('alert_events', 'triggered_at',
    chunk_time_interval => INTERVAL '1 month');

CREATE INDEX idx_alerts_rule ON alert_events(alert_rule_id, triggered_at DESC);
CREATE INDEX idx_alerts_severity ON alert_events(severity, triggered_at DESC);
CREATE INDEX idx_alerts_unresolved ON alert_events(resolved_at)
    WHERE resolved_at IS NULL;

-- ML inference results
CREATE TABLE ml_inference_log (
    inferred_at     TIMESTAMPTZ NOT NULL,
    model_id        UUID NOT NULL,
    gateway_id      UUID NOT NULL,
    inference_ms    REAL NOT NULL,
    anomaly_score   DOUBLE PRECISION,
    prediction      VARCHAR(100),
    input_summary   JSONB,                          -- summary of input features
    output_detail   JSONB                           -- full model output
);

SELECT create_hypertable('ml_inference_log', 'inferred_at',
    chunk_time_interval => INTERVAL '1 week');
```

---

## Supporting Relational Tables

Some entities are simpler to manage as standard PostgreSQL tables with foreign key constraints:

```sql
-- Users and authentication (no graph benefit here)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,                 -- references Organization vertex id
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(30) NOT NULL DEFAULT 'operator',
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    preferences     JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Software release artifacts
CREATE TABLE software_releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version         VARCHAR(50) NOT NULL UNIQUE,
    release_type    VARCHAR(30) NOT NULL,
    artifact_url    VARCHAR(1024) NOT NULL,
    artifact_hash   VARCHAR(64) NOT NULL,
    artifact_size_bytes BIGINT NOT NULL,
    target_architectures TEXT[] NOT NULL,
    min_agent_version VARCHAR(50),
    release_notes   TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    released_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- OTA rollout campaigns
CREATE TABLE rollout_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    release_id      UUID NOT NULL REFERENCES software_releases(id),
    name            VARCHAR(255) NOT NULL,
    strategy        JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    progress        JSONB NOT NULL DEFAULT '{}',
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rollout_gateway_status (
    campaign_id     UUID NOT NULL REFERENCES rollout_campaigns(id) ON DELETE CASCADE,
    gateway_id      UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    previous_version VARCHAR(50),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    PRIMARY KEY (campaign_id, gateway_id)
);

-- Audit log (time-series-like, high volume)
CREATE TABLE audit_log (
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_type      VARCHAR(20) NOT NULL,
    actor_id        VARCHAR(255) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     VARCHAR(255) NOT NULL,
    success         BOOLEAN NOT NULL DEFAULT true,
    context         JSONB NOT NULL DEFAULT '{}'
);

SELECT create_hypertable('audit_log', 'timestamp',
    chunk_time_interval => INTERVAL '1 month');
```

---

## Cross-Paradigm Queries

The true power of this model is combining graph traversals with time-series analytics in a single database:

```sql
-- Query 1: Get average temperature for all critical sensors in a specific area,
-- traversing the graph to find them and then querying the time-series data.

-- Step 1: Use Cypher to find data point IDs
SELECT * FROM cypher('iiot', $$
    MATCH (area:Area {name: 'Distillation Unit A'})
          -[:CONTAINS_DEVICE]->(device:Device)
          -[:HAS_DATA_POINT]->(dp:DataPoint)
    WHERE dp.is_critical = true AND dp.unit_of_measure = 'celsius'
    RETURN dp.id AS data_point_id, device.name AS device_name, dp.name AS sensor_name
$$) AS (data_point_id agtype, device_name agtype, sensor_name agtype);

-- Step 2: Use those IDs in a TimescaleDB query
WITH critical_sensors AS (
    SELECT dp_id::uuid AS data_point_id
    FROM cypher('iiot', $$
        MATCH (:Area {name: 'Distillation Unit A'})
              -[:CONTAINS_DEVICE]->(:Device)
              -[:HAS_DATA_POINT]->(dp:DataPoint {is_critical: true})
        RETURN dp.id AS dp_id
    $$) AS (dp_id agtype)
)
SELECT
    cs.data_point_id,
    time_bucket('1 hour', t.recorded_at) AS hour,
    AVG(t.value_numeric) AS avg_temp,
    MAX(t.value_numeric) AS max_temp,
    MIN(t.value_numeric) AS min_temp
FROM telemetry_readings t
JOIN critical_sensors cs ON t.data_point_id = cs.data_point_id
WHERE t.recorded_at > now() - INTERVAL '24 hours'
GROUP BY cs.data_point_id, hour
ORDER BY hour DESC;


-- Query 2: Impact analysis — if a motor is flagged for maintenance,
-- find all downstream equipment and their recent sensor anomalies

-- Graph traversal: find equipment downstream of the motor
WITH downstream_devices AS (
    SELECT device_id::uuid
    FROM cypher('iiot', $$
        MATCH (motor:Device {name: 'Motor #3'})
              -[:DRIVES|FEEDS_INTO*1..5]->(downstream:Device)
        RETURN downstream.id AS device_id
    $$) AS (device_id agtype)
),
downstream_points AS (
    SELECT dp_id::uuid AS data_point_id
    FROM downstream_devices dd,
    LATERAL (
        SELECT dp_id FROM cypher('iiot', $$
            MATCH (d:Device)-[:HAS_DATA_POINT]->(dp:DataPoint)
            WHERE d.id = $device_id
            RETURN dp.id AS dp_id
        $$, $1) AS (dp_id agtype)
    ) sub
)
-- Time-series query: find anomalies on downstream sensors
SELECT
    dp.data_point_id,
    COUNT(*) FILTER (WHERE t.value_numeric > 
        (SELECT val_avg + 3 * val_stddev FROM telemetry_1hr 
         WHERE data_point_id = dp.data_point_id 
         ORDER BY bucket DESC LIMIT 1)
    ) AS anomaly_count
FROM telemetry_readings t
JOIN downstream_points dp ON t.data_point_id = dp.data_point_id
WHERE t.recorded_at > now() - INTERVAL '24 hours'
GROUP BY dp.data_point_id;


-- Query 3: Fleet-wide gateway health correlated with device topology depth
WITH gateway_complexity AS (
    SELECT gw_id::uuid AS gateway_id, device_count::int, adapter_count::int
    FROM cypher('iiot', $$
        MATCH (gw:Gateway)
        OPTIONAL MATCH (gw)-[:RUNS_ADAPTER]->(a:ProtocolAdapter)-[:CONNECTS_TO]->(d:Device)
        RETURN gw.id AS gw_id,
               count(DISTINCT d) AS device_count,
               count(DISTINCT a) AS adapter_count
    $$) AS (gw_id agtype, device_count agtype, adapter_count agtype)
)
SELECT
    gc.gateway_id,
    gc.device_count,
    gc.adapter_count,
    AVG(gh.cpu_percent) AS avg_cpu_24h,
    MAX(gh.cpu_percent) AS max_cpu_24h,
    AVG(gh.memory_used_mb) AS avg_memory_24h
FROM gateway_health gh
JOIN gateway_complexity gc ON gh.gateway_id = gc.gateway_id
WHERE gh.recorded_at > now() - INTERVAL '24 hours'
GROUP BY gc.gateway_id, gc.device_count, gc.adapter_count
ORDER BY gc.device_count DESC;
```

---

## Edge-Side Schema (SQLite)

```sql
-- Store-and-forward telemetry buffer
CREATE TABLE telemetry_buffer (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    data_point_id   TEXT NOT NULL,
    recorded_at     TEXT NOT NULL,
    value_numeric   REAL,
    value_string    TEXT,
    value_boolean   INTEGER,
    quality         INTEGER NOT NULL DEFAULT 192,
    sync_status     TEXT NOT NULL DEFAULT 'pending',
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_buffer_sync ON telemetry_buffer(sync_status)
    WHERE sync_status = 'pending';

-- Local device topology cache (simplified graph as adjacency list)
CREATE TABLE local_topology (
    source_id       TEXT NOT NULL,
    source_type     TEXT NOT NULL,              -- 'gateway', 'adapter', 'device', 'data_point'
    target_id       TEXT NOT NULL,
    target_type     TEXT NOT NULL,
    relationship    TEXT NOT NULL,              -- 'runs_adapter', 'connects_to', 'has_data_point'
    properties      TEXT DEFAULT '{}',          -- JSON
    PRIMARY KEY (source_id, target_id, relationship)
);

-- Local config cache
CREATE TABLE config_cache (
    key             TEXT PRIMARY KEY,
    value           TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Alert buffer
CREATE TABLE alert_buffer (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    alert_rule_id   TEXT NOT NULL,
    data_point_id   TEXT,
    triggered_at    TEXT NOT NULL,
    trigger_value   REAL,
    severity        TEXT NOT NULL,
    message         TEXT,
    sync_status     TEXT NOT NULL DEFAULT 'pending'
);
```

---

## Pros and Cons

### Pros

1. **Natural topology modeling** — Industrial equipment hierarchies (ISA-95: Enterprise > Site > Area > Equipment > Sensor) and physical connectivity (motor drives pump, pump feeds exchanger) are inherently graph-shaped. Cypher queries like `MATCH path = (motor)-[:DRIVES*1..5]->(downstream)` express these traversals naturally, whereas SQL requires recursive CTEs or multiple self-joins that are harder to write and optimize.

2. **Impact analysis in constant time** — When a gateway goes offline or a device needs maintenance, graph traversal instantly identifies all affected data points, alert rules, ML models, and downstream equipment. This is the "killer query" for operational decision-making that relational databases handle poorly once the topology grows complex.

3. **Digital twin foundation** — The graph model directly supports digital twin requirements: device relationships, connectivity topology, and physical process flow. As the platform evolves toward digital twin integration (listed in the backlog), the graph model is already in place.

4. **Optimized time-series performance** — TimescaleDB hypertables with compression, continuous aggregates, and retention policies provide order-of-magnitude better performance for telemetry queries compared to standard PostgreSQL tables. Compression ratios of 10-20x reduce storage costs dramatically.

5. **Single database process** — Apache AGE and TimescaleDB both run as PostgreSQL extensions in the same database process. No separate graph database to manage, no cross-database synchronization, shared backup and recovery, single connection pool.

6. **Protocol topology discovery** — When the platform adds protocol auto-discovery (backlog feature), discovered devices and their relationships can be directly inserted as graph vertices and edges, building the facility topology automatically.

7. **Correlation analysis** — The `CORRELATES_WITH` edge between data points enables cross-sensor correlation analysis. When an anomaly is detected on one sensor, graph traversal finds correlated sensors and their recent readings, enabling root cause analysis.

### Cons

1. **Apache AGE maturity** — AGE is younger than Neo4j and has a smaller community. Some Cypher features are not yet supported, and performance optimization for large graphs is less documented. Production deployments at scale are fewer.

2. **Mixed query complexity** — Combining Cypher subqueries with SQL requires wrapping Cypher in `cypher()` function calls and casting AGE's `agtype` results to PostgreSQL types. This is syntactically awkward and adds cognitive load for developers.

3. **Graph indexing limitations** — AGE uses PostgreSQL's GIN indexes for graph vertex properties, which are less specialized than Neo4j's native graph indexes. Large-scale graph traversals (100,000+ vertices) may perform slower than a dedicated graph database.

4. **Extension compatibility risk** — Running both AGE and TimescaleDB simultaneously as PostgreSQL extensions requires version compatibility between the extensions and the PostgreSQL major version. Upgrades must be coordinated carefully. A PostgreSQL major version upgrade may require waiting for both extensions to support the new version.

5. **No native graph visualization** — AGE does not include built-in graph visualization tools. A custom front-end using D3.js, vis.js, or Cytoscape.js is needed for the topology viewer. Neo4j Browser, by contrast, provides this out of the box.

6. **Learning curve** — The team must be proficient in three query languages: SQL (for relational tables), Cypher (for graph queries), and TimescaleDB-specific SQL extensions (for time-series functions). This is a broader skill requirement than a pure PostgreSQL approach.

7. **Graph-relational boundary** — Some data sits at the boundary between graph and relational models (e.g., rollout campaigns, user accounts). Deciding where each entity lives requires judgment calls that may differ between developers, leading to inconsistency without clear guidelines.

8. **Backup and restore complexity** — While all data is in one PostgreSQL instance, AGE stores graph data in schema-specific tables that must be backed up and restored together. Partial restores (e.g., restoring only the graph) require understanding AGE's internal storage format.

---

## Migration and Scaling Considerations

### Development Phase
- Start with a single PostgreSQL instance hosting both AGE and TimescaleDB
- Use the graph model for all topology/relationship data from day one
- Use TimescaleDB hypertables for all time-stamped data from day one
- Use standard tables for users, releases, campaigns
- Build a simple topology viewer early (D3.js force-directed graph) to validate the graph model with stakeholders

### Growth Phase (100-1,000 gateways)
- Add PostgreSQL read replicas for dashboard queries
- Enable TimescaleDB compression policies aggressively
- Monitor AGE graph query performance; add property indexes on frequently filtered vertex properties
- Consider separating the graph write path (topology changes are infrequent) from the time-series write path (telemetry is continuous)

### Scale Phase (1,000-10,000 gateways)
- TimescaleDB multi-node for telemetry sharding across multiple PostgreSQL instances
- The graph remains on a single node (topology size grows linearly with fleet size; 10,000 gateways with 100,000 devices is still a small graph by graph database standards)
- If AGE performance becomes a bottleneck, evaluate migrating the graph layer to Neo4j while keeping TimescaleDB for time-series. The Cypher queries remain identical.

### Fallback Plan: Neo4j Migration
If Apache AGE proves insufficiently mature for production:
- Cypher queries transfer directly to Neo4j with minimal changes
- Graph data exports as CSV or JSON for Neo4j import
- Application code switches from `cypher('iiot', ...)` function calls to Neo4j Bolt driver calls
- TimescaleDB remains untouched — only the graph layer migrates
- Cross-database queries require application-level joins rather than single SQL statements

### Data Retention Strategy
| Data Type | Hot (SSD) | Warm (HDD) | Cold (Object Storage) |
|-----------|-----------|------------|----------------------|
| Raw telemetry | 90 days | — | Archive after 90 days |
| 5-min aggregates | 1 year | — | Archive after 1 year |
| Hourly aggregates | 3 years | — | Archive after 3 years |
| Daily aggregates | 10 years | — | — |
| Graph topology | Indefinite | — | — |
| Alert events | 2 years | 5 years | 7 years total |
| Gateway health | 30 days | — | — |
| Audit log | 1 year | 6 years | 7 years total |
| ML inference log | 90 days | — | Archive for retraining |

### Storage Estimates (1,000 gateways, 50,000 data points at 1 Hz)
- Raw telemetry: ~100 bytes/row x 4.3B rows/day = ~430 GB/day uncompressed
- With TimescaleDB compression (10-20x): ~25-43 GB/day
- 90-day retention: ~2.3-3.9 TB compressed
- Graph data: ~50,000 vertices + ~200,000 edges = ~100 MB (negligible)
- Continuous aggregates add ~10% to storage
- Total hot storage: ~4-5 TB

---

## Technology Recommendations

1. **Adopt this model if digital twin and topology analysis are strategic priorities.** The graph layer provides capabilities that are extremely difficult to retrofit onto a pure relational model.

2. **Use AGE for equipment relationships and topology; use TimescaleDB for everything time-stamped.** Keep the boundary crisp: if it has a timestamp and is append-mostly, it goes in a hypertable. If it represents an entity or a relationship, it goes in the graph.

3. **Build the topology viewer early.** The graph model's value becomes tangible when operators can visualize the physical topology, click on a device, and see its upstream/downstream connections alongside real-time telemetry. This is a differentiating user experience that competitors lack.

4. **Start with AGE but have a Neo4j migration plan.** AGE's advantage is operational simplicity (single PostgreSQL instance). If graph query performance or feature limitations become blockers, the Cypher queries port directly to Neo4j.

5. **Use GraphQL as the API layer.** GraphQL's nested query structure maps naturally to graph traversals with embedded time-series data:
   ```graphql
   query {
     site(slug: "houston-refinery") {
       areas {
         devices {
           name
           dataPoints(critical: true) {
             name
             latestReading { value, timestamp }
             last24hStats { avg, max, min }
           }
         }
       }
     }
   }
   ```

6. **Consider Apache IoTDB as an alternative time-series layer** if the fleet grows beyond TimescaleDB's comfortable range. IoTDB's tree-based schema (root.site.area.device.sensor) mirrors the graph hierarchy and is purpose-built for industrial IoT at massive scale (billions of time series). However, this adds a second database to manage.

7. **Invest in GIN indexes for graph vertex properties** that appear in Cypher WHERE clauses. AGE performance depends heavily on efficient property lookups, and missing indexes on frequently queried properties (status, device_type, is_critical) will cause full graph scans.
