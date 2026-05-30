# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Industrial IoT Edge Platform (452) · Generated: 2026-05-25

## Overview

This model uses PostgreSQL with a deliberate mix of normalized relational tables for stable, well-understood entities and JSONB columns for data that varies by protocol, device type, or deployment context. The core insight is that an industrial IoT edge platform must handle extreme heterogeneity — a Modbus temperature sensor, an OPC-UA CNC controller, and a BACnet HVAC system all have radically different metadata, configuration parameters, and telemetry shapes. Forcing all of these into a fixed relational schema leads to either hundreds of nullable columns or an explosion of type-specific tables.

The hybrid approach normalizes what is universal (organizational hierarchy, gateway identity, telemetry timestamps, alert rules, user accounts) while using JSONB for what varies (protocol-specific configuration, device-specific metadata, sensor calibration parameters, ML model hyperparameters, pipeline node configurations). PostgreSQL's JSONB support includes GIN indexing, JSON path queries, partial indexing, and schema validation via CHECK constraints — providing the query power of a document database without abandoning SQL.

---

## Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Primary Database | PostgreSQL 16+ with JSONB | Single engine for relational + document workloads |
| Time-series extension | TimescaleDB | Hypertables for telemetry; JSONB columns in hypertables for flexible sensor payloads |
| Edge-local DB | SQLite with JSON1 extension | JSON support built into SQLite for edge store-and-forward |
| Schema validation | pg_jsonschema or CHECK constraints | Enforce structure on JSONB columns where needed |
| Search indexing | GIN indexes on JSONB | Query into document columns without deserialization |
| Migration tool | Flyway | Versioned migrations; JSONB columns handle additive changes without migrations |

---

## Design Principles

1. **Normalize the universal, document the variable.** If a column exists for every row and is used in JOINs, WHERE clauses, or foreign keys, it belongs in a relational column. If it varies by protocol, device type, or deployment, it goes in JSONB.

2. **Typed JSONB with validation.** JSONB columns are not unstructured blobs. Each one has a documented JSON schema and, where practical, a CHECK constraint or pg_jsonschema validation to prevent garbage data.

3. **Relational columns for query patterns.** Fields that appear in WHERE, ORDER BY, GROUP BY, or JOIN clauses are always relational columns, even if they also appear in the JSONB payload. This avoids forcing the query planner to extract values from JSONB at query time.

4. **JSONB for protocol abstraction.** Each industrial protocol (OPC-UA, Modbus, BACnet, DNP3) has unique connection parameters, addressing schemes, and data representations. JSONB columns absorb this diversity without protocol-specific tables.

---

## Complete Schema Definition

### 1. Organization and Site Hierarchy

These entities are stable and universal — fully normalized.

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
        -- Flexible org-wide settings:
        -- {
        --   "default_timezone": "America/Chicago",
        --   "industry_sector": "manufacturing",
        --   "data_retention_days": 365,
        --   "security_policy": {"min_tls_version": "1.3", "cert_rotation_days": 90},
        --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"}
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    location        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "address": {"line1": "123 Industrial Pkwy", "city": "Houston", "state": "TX",
        --               "country": "US", "postal_code": "77001"},
        --   "coordinates": {"lat": 29.7604, "lng": -95.3698},
        --   "facility_type": "refinery",
        --   "building_codes": ["IBC-2021", "NFPA-70"]
        -- }
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- Extensible site metadata:
        -- {
        --   "isa95_level": 3,
        --   "regulatory_zone": "NERC-CIP",
        --   "network_topology": "purdue_model",
        --   "backup_power": true,
        --   "wan_redundancy": "4G_failover"
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_sites_org ON sites(organization_id);
CREATE INDEX idx_sites_metadata ON sites USING GIN (metadata);

CREATE TABLE areas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    area_type       VARCHAR(50) NOT NULL,
    parent_area_id  UUID REFERENCES areas(id) ON DELETE SET NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "floor": 2,
        --   "zone": "hazardous_area_zone_1",
        --   "isa95_work_cell": "assembly_line_3",
        --   "environmental_class": "IP65",
        --   "max_temperature_c": 55
        -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, slug)
);

CREATE INDEX idx_areas_site ON areas(site_id);
```

### 2. Edge Gateway Fleet

Gateway identity and status are relational; hardware specifics and operational parameters go in JSONB.

```sql
CREATE TABLE edge_gateways (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id) ON DELETE RESTRICT,
    area_id             UUID REFERENCES areas(id) ON DELETE SET NULL,
    serial_number       VARCHAR(100) NOT NULL UNIQUE,
    hardware_model      VARCHAR(100) NOT NULL,
    cpu_architecture    VARCHAR(20) NOT NULL,       -- query-critical: filter deployments by arch
    status              VARCHAR(20) NOT NULL DEFAULT 'provisioning',
    agent_version       VARCHAR(50),
    last_heartbeat_at   TIMESTAMPTZ,
    last_sync_at        TIMESTAMPTZ,

    -- Hardware details vary by model — JSONB absorbs this
    hardware_specs      JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "ram_mb": 256,
        --   "storage_mb": 8192,
        --   "storage_type": "emmc",
        --   "cpu_cores": 4,
        --   "cpu_freq_mhz": 1200,
        --   "has_tpm": true,
        --   "network_interfaces": [
        --     {"name": "eth0", "type": "ethernet", "speed_mbps": 1000},
        --     {"name": "wwan0", "type": "4g_lte", "carrier": "Verizon"}
        --   ],
        --   "serial_ports": ["/dev/ttyUSB0", "/dev/ttyUSB1"],
        --   "gpio_pins": 8,
        --   "operating_temp_range_c": [-40, 75]
        -- }

    -- Network configuration — varies by deployment
    network_config      JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "ip_lan": "192.168.1.100",
        --   "ip_wan": "203.0.113.42",
        --   "mac_address": "aa:bb:cc:dd:ee:ff",
        --   "dns_servers": ["8.8.8.8", "8.8.4.4"],
        --   "ntp_servers": ["pool.ntp.org"],
        --   "proxy": {"host": "proxy.corp.com", "port": 3128},
        --   "firewall_rules": [...]
        -- }

    -- OS and runtime details
    runtime_info        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "os_name": "linux",
        --   "os_version": "6.1.0",
        --   "os_distribution": "Yocto",
        --   "container_runtime": "containerd",
        --   "container_runtime_version": "1.7.2"
        -- }

    -- Tags for fleet segmentation and filtering
    tags                JSONB NOT NULL DEFAULT '{}',
        -- {"environment": "production", "priority": "critical", "customer": "acme_corp"}

    provisioned_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_gateway_status CHECK (
        status IN ('provisioning', 'online', 'offline', 'degraded', 'decommissioned')
    )
);

CREATE INDEX idx_gateways_site ON edge_gateways(site_id);
CREATE INDEX idx_gateways_status ON edge_gateways(status);
CREATE INDEX idx_gateways_arch ON edge_gateways(cpu_architecture);
CREATE INDEX idx_gateways_heartbeat ON edge_gateways(last_heartbeat_at);
CREATE INDEX idx_gateways_tags ON edge_gateways USING GIN (tags);

-- Gateway health: high-frequency data with JSONB for variable metrics
CREATE TABLE gateway_health_snapshots (
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    recorded_at     TIMESTAMPTZ NOT NULL,
    cpu_percent     REAL,
    memory_used_mb  REAL,
    wan_connected   BOOLEAN NOT NULL DEFAULT true,

    -- Variable health metrics depend on hardware model
    extended_metrics JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "disk_used_mb": 2048,
        --   "disk_total_mb": 8192,
        --   "temperature_c": 42.5,
        --   "wan_latency_ms": 23.4,
        --   "uptime_seconds": 864000,
        --   "cellular_signal_dbm": -67,
        --   "io_wait_percent": 2.1,
        --   "container_count": 4,
        --   "buffer_queue_depth": 1250,
        --   "power_supply_voltage": 12.1
        -- }

    PRIMARY KEY (gateway_id, recorded_at)
);

-- Convert to hypertable for time-series access
SELECT create_hypertable('gateway_health_snapshots', 'recorded_at',
    chunk_time_interval => INTERVAL '1 day');
```

### 3. Protocol Adapters — Protocol Diversity via JSONB

```sql
CREATE TABLE protocol_adapter_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    protocol_name   VARCHAR(50) NOT NULL UNIQUE,
    display_name    VARCHAR(100) NOT NULL,
    transport       VARCHAR(20) NOT NULL,
    default_port    INTEGER,

    -- Protocol-specific configuration schema (JSON Schema format)
    config_schema   JSONB NOT NULL,
        -- Defines what fields are valid in connection_config for this protocol type
        -- Example for Modbus TCP:
        -- {
        --   "type": "object",
        --   "required": ["host", "port", "unit_id"],
        --   "properties": {
        --     "host": {"type": "string"},
        --     "port": {"type": "integer", "default": 502},
        --     "unit_id": {"type": "integer", "minimum": 1, "maximum": 247},
        --     "byte_order": {"type": "string", "enum": ["big_endian", "little_endian"]},
        --     "word_order": {"type": "string", "enum": ["big_endian", "little_endian"]}
        --   }
        -- }

    -- Protocol-specific data point addressing schema
    address_schema  JSONB NOT NULL,
        -- Defines valid address formats for data points on this protocol
        -- Example for Modbus:
        -- {
        --   "type": "object",
        --   "required": ["register_type", "address"],
        --   "properties": {
        --     "register_type": {"type": "string", "enum": ["holding", "input", "coil", "discrete"]},
        --     "address": {"type": "integer", "minimum": 0, "maximum": 65535},
        --     "count": {"type": "integer", "minimum": 1, "maximum": 125}
        --   }
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Protocol adapter instances running on specific gateways
CREATE TABLE protocol_adapter_instances (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id          UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    adapter_type_id     UUID NOT NULL REFERENCES protocol_adapter_types(id) ON DELETE RESTRICT,
    name                VARCHAR(255) NOT NULL,
    enabled             BOOLEAN NOT NULL DEFAULT true,
    poll_interval_ms    INTEGER NOT NULL DEFAULT 1000,
    status              VARCHAR(20) NOT NULL DEFAULT 'stopped',

    -- Protocol-specific connection configuration — validated against adapter_type.config_schema
    connection_config   JSONB NOT NULL,
        -- Modbus TCP example:
        -- {"host": "192.168.1.50", "port": 502, "unit_id": 1, "byte_order": "big_endian"}
        --
        -- OPC-UA example:
        -- {
        --   "endpoint_url": "opc.tcp://192.168.1.100:4840",
        --   "security_mode": "SignAndEncrypt",
        --   "security_policy": "Basic256Sha256",
        --   "auth_type": "certificate",
        --   "cert_thumbprint": "A1B2C3...",
        --   "subscription_interval_ms": 500,
        --   "max_monitored_items": 1000
        -- }
        --
        -- BACnet example:
        -- {
        --   "network_interface": "eth0",
        --   "bbmd_address": "192.168.1.1",
        --   "bbmd_port": 47808,
        --   "device_instance_range": [1, 4194303]
        -- }
        --
        -- DNP3 example:
        -- {
        --   "host": "192.168.1.75",
        --   "port": 20000,
        --   "master_address": 1,
        --   "outstation_address": 10,
        --   "unsolicited_responses": true
        -- }

    -- Runtime state (updated by the gateway, not user-editable)
    runtime_state       JSONB NOT NULL DEFAULT '{}',
        -- {"last_error": null, "connected_since": "...", "messages_per_second": 42.5}

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_adapter_status CHECK (
        status IN ('running', 'stopped', 'error', 'connecting')
    )
);

CREATE INDEX idx_adapter_gateway ON protocol_adapter_instances(gateway_id);
CREATE INDEX idx_adapter_config ON protocol_adapter_instances USING GIN (connection_config);
```

### 4. Device and Sensor Registry

```sql
CREATE TABLE devices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    adapter_instance_id UUID NOT NULL REFERENCES protocol_adapter_instances(id) ON DELETE CASCADE,
    gateway_id          UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    area_id             UUID REFERENCES areas(id) ON DELETE SET NULL,
    name                VARCHAR(255) NOT NULL,
    device_type         VARCHAR(100),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    discovered_at       TIMESTAMPTZ,
    last_seen_at        TIMESTAMPTZ,

    -- Protocol-specific device identity — varies completely by protocol
    device_address      JSONB NOT NULL,
        -- Modbus: {"unit_id": 1}
        -- OPC-UA: {"node_id": "ns=2;s=Device1", "browse_path": "/Objects/Device1"}
        -- BACnet: {"device_instance": 1234, "network_number": 1, "mac_address": "0A"}
        -- DNP3:   {"outstation_address": 10}

    -- Device metadata — manufacturer, model, capabilities vary by type
    device_metadata     JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "manufacturer": "Siemens",
        --   "model": "S7-1500",
        --   "firmware_version": "2.9.4",
        --   "serial_number": "SN-12345",
        --   "isa95_equipment_class": "PLC",
        --   "nameplate": {
        --     "rated_power_kw": 7.5,
        --     "rated_speed_rpm": 1750,
        --     "voltage": "480V 3-phase"
        --   },
        --   "maintenance": {
        --     "last_service_date": "2025-11-15",
        --     "next_service_date": "2026-05-15",
        --     "warranty_expires": "2027-01-15"
        --   },
        --   "capabilities": ["firmware_update", "time_sync", "diagnostics"]
        -- }

    -- Custom user-defined attributes
    custom_attributes   JSONB NOT NULL DEFAULT '{}',
        -- {"cost_center": "CC-4500", "criticality": "high", "asset_tag": "AST-00142"}

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Ensure unique device per adapter using JSONB equality
    UNIQUE (adapter_instance_id, device_address)
);

CREATE INDEX idx_devices_gateway ON devices(gateway_id);
CREATE INDEX idx_devices_type ON devices(device_type);
CREATE INDEX idx_devices_metadata ON devices USING GIN (device_metadata);
CREATE INDEX idx_devices_custom ON devices USING GIN (custom_attributes);

-- Data points (sensors, registers, variables)
CREATE TABLE data_points (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id           UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    display_name        VARCHAR(255),
    data_type           VARCHAR(30) NOT NULL,       -- relational: used in query logic
    unit_of_measure     VARCHAR(50),                -- relational: used in display and conversion
    is_critical         BOOLEAN NOT NULL DEFAULT false,
    is_writable         BOOLEAN NOT NULL DEFAULT false,
    enabled             BOOLEAN NOT NULL DEFAULT true,

    -- Protocol-specific addressing — completely different per protocol
    protocol_address    JSONB NOT NULL,
        -- Modbus: {"register_type": "holding", "address": 40001, "count": 2, "data_format": "float32_be"}
        -- OPC-UA: {"node_id": "ns=2;s=Device1.Temperature", "attribute": "Value", "sampling_interval_ms": 100}
        -- BACnet: {"object_type": "analog-input", "object_instance": 1, "property": "present-value"}
        -- DNP3:   {"point_type": "analog_input", "index": 0, "variation": 4, "class": 1}

    -- Scaling, calibration, and processing parameters
    processing_config   JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "scaling": {"factor": 0.1, "offset": -40.0},
        --   "deadband": {"type": "absolute", "value": 0.5},
        --   "valid_range": {"min": -50.0, "max": 200.0},
        --   "smoothing": {"type": "exponential_moving_average", "alpha": 0.3},
        --   "poll_priority": 1,
        --   "custom_transform": "value * 9/5 + 32"
        -- }

    -- ISA-95 semantic mapping
    semantic_mapping    JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "isa95_property": "Equipment.Temperature.ProcessValue",
        --   "unified_name": "motor_bearing_temp",
        --   "engineering_unit_code": "CEL",
        --   "opc_ua_node_class": "Variable"
        -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (device_id, name)
);

CREATE INDEX idx_dp_device ON data_points(device_id);
CREATE INDEX idx_dp_critical ON data_points(is_critical) WHERE is_critical = true;
CREATE INDEX idx_dp_address ON data_points USING GIN (protocol_address);
CREATE INDEX idx_dp_semantic ON data_points USING GIN (semantic_mapping);
```

### 5. Telemetry Storage

```sql
-- Telemetry uses a narrow schema with an optional JSONB column for
-- multi-value readings (e.g., a vibration sensor reporting X/Y/Z axes)
CREATE TABLE telemetry_readings (
    recorded_at     TIMESTAMPTZ NOT NULL,
    data_point_id   UUID NOT NULL,
    gateway_id      UUID NOT NULL,
    value_numeric   DOUBLE PRECISION,
    quality         SMALLINT NOT NULL DEFAULT 192,

    -- For sensors that produce structured/multi-dimensional readings
    extended_value  JSONB,
        -- Vibration sensor: {"x_mm_s": 2.4, "y_mm_s": 1.8, "z_mm_s": 3.1, "dominant_frequency_hz": 47.2}
        -- Power meter: {"voltage_l1": 231.2, "voltage_l2": 230.8, "voltage_l3": 231.5,
        --               "current_l1": 12.4, "power_factor": 0.92, "thd_percent": 3.2}
        -- String/enum value: {"state": "running", "mode": "auto"}
        -- GPS/position: {"latitude": 29.7604, "longitude": -95.3698, "altitude_m": 15.2}
        -- NULL for simple scalar readings (value_numeric is sufficient)

    -- Optional per-reading metadata
    reading_metadata JSONB
        -- {"source": "store_and_forward", "original_timestamp": "...", "batch_id": "..."}
);

-- TimescaleDB hypertable
SELECT create_hypertable('telemetry_readings', 'recorded_at',
    chunk_time_interval => INTERVAL '1 day');

-- Compression for historical data
ALTER TABLE telemetry_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'data_point_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);
SELECT add_compression_policy('telemetry_readings', INTERVAL '7 days');

-- Retention
SELECT add_retention_policy('telemetry_readings', INTERVAL '90 days');

-- Indexes
CREATE INDEX idx_telemetry_dp_time ON telemetry_readings(data_point_id, recorded_at DESC);
CREATE INDEX idx_telemetry_gateway ON telemetry_readings(gateway_id, recorded_at DESC);
-- GIN index on extended_value only for rows that have it (partial index saves space)
CREATE INDEX idx_telemetry_extended ON telemetry_readings USING GIN (extended_value)
    WHERE extended_value IS NOT NULL;

-- Continuous aggregates
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
    COUNT(*) AS reading_count
FROM telemetry_readings
GROUP BY bucket, data_point_id, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('telemetry_5min',
    start_offset => INTERVAL '1 hour',
    end_offset => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '5 minutes');

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
    COUNT(*) AS reading_count
FROM telemetry_readings
GROUP BY bucket, data_point_id, gateway_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('telemetry_1hr',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### 6. Alert Rules and Events

```sql
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT true,

    -- Target specification — which data points this rule applies to
    target          JSONB NOT NULL,
        -- Single point: {"type": "data_point", "data_point_id": "..."}
        -- All points on device: {"type": "device", "device_id": "..."}
        -- By tag query: {"type": "tag_query", "device_type": "vfd", "custom_attributes": {"criticality": "high"}}

    -- Condition definition — JSONB allows complex multi-condition rules
    condition       JSONB NOT NULL,
        -- Simple threshold: {"type": "threshold_high", "value": 85.0, "duration_ms": 5000}
        -- Range: {"type": "range_outside", "low": 20.0, "high": 80.0}
        -- Rate of change: {"type": "rate_of_change", "max_per_second": 5.0, "window_seconds": 60}
        -- Compound: {"type": "and", "conditions": [
        --   {"type": "threshold_high", "field": "value_numeric", "value": 100.0},
        --   {"type": "threshold_high", "field": "extended_value.thd_percent", "value": 10.0}
        -- ]}
        -- Flatline: {"type": "flatline", "max_unchanged_seconds": 300}
        -- Quality: {"type": "quality_below", "min_quality": 128}

    -- Actions to take when alert fires
    actions         JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"type": "notification", "channel": "email", "recipients": ["ops@example.com"]},
        --   {"type": "notification", "channel": "slack", "webhook_url": "..."},
        --   {"type": "webhook", "url": "https://mes.example.com/api/alerts", "method": "POST"},
        --   {"type": "command", "gateway_id": "...", "command": "stop_pump", "params": {...}}
        -- ]

    cooldown_ms     INTEGER DEFAULT 300000,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_alert_severity CHECK (
        severity IN ('info', 'warning', 'critical', 'emergency')
    )
);

CREATE INDEX idx_alert_rules_org ON alert_rules(organization_id);
CREATE INDEX idx_alert_rules_target ON alert_rules USING GIN (target);

CREATE TABLE alert_events (
    id              BIGSERIAL,
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    gateway_id      UUID NOT NULL,
    triggered_at    TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    severity        VARCHAR(20) NOT NULL,

    -- Trigger context — JSONB captures the full context of what triggered the alert
    trigger_context JSONB NOT NULL,
        -- {
        --   "data_point_id": "...",
        --   "data_point_name": "Motor Bearing Temperature",
        --   "device_name": "Pump #3 VFD",
        --   "trigger_value": 92.4,
        --   "threshold": 85.0,
        --   "condition_type": "threshold_high",
        --   "readings_window": [88.1, 89.3, 90.7, 91.2, 92.4],
        --   "duration_exceeded_ms": 5200
        -- }

    acknowledgement JSONB,
        -- {"acknowledged_by": "user-id", "acknowledged_at": "...", "notes": "Scheduled maintenance"}

    PRIMARY KEY (triggered_at, id)
) PARTITION BY RANGE (triggered_at);

CREATE INDEX idx_alert_events_rule ON alert_events(alert_rule_id, triggered_at DESC);
CREATE INDEX idx_alert_events_unresolved ON alert_events(resolved_at)
    WHERE resolved_at IS NULL;
```

### 7. Data Pipeline Configuration

```sql
-- Pipeline definitions — the dataflow graph
CREATE TABLE data_pipelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    version         INTEGER NOT NULL DEFAULT 1,

    -- The entire pipeline graph as a single JSONB document
    -- This is the approach used by Node-RED and similar visual dataflow tools
    graph           JSONB NOT NULL,
        -- {
        --   "nodes": [
        --     {
        --       "id": "node-1",
        --       "type": "source_modbus",
        --       "label": "Temperature Sensors",
        --       "position": {"x": 100, "y": 200},
        --       "config": {
        --         "adapter_instance_id": "...",
        --         "data_points": ["dp-1", "dp-2", "dp-3"]
        --       }
        --     },
        --     {
        --       "id": "node-2",
        --       "type": "filter_deadband",
        --       "label": "Deadband Filter",
        --       "position": {"x": 300, "y": 200},
        --       "config": {"deadband_value": 0.5, "deadband_type": "absolute"}
        --     },
        --     {
        --       "id": "node-3",
        --       "type": "aggregate_window",
        --       "label": "5-min Average",
        --       "position": {"x": 500, "y": 150},
        --       "config": {"window_size_ms": 300000, "function": "avg"}
        --     },
        --     {
        --       "id": "node-4",
        --       "type": "ml_inference",
        --       "label": "Anomaly Detector",
        --       "position": {"x": 500, "y": 300},
        --       "config": {"model_deployment_id": "...", "threshold": 0.85}
        --     },
        --     {
        --       "id": "node-5",
        --       "type": "sink_mqtt",
        --       "label": "Cloud Sync",
        --       "position": {"x": 700, "y": 200},
        --       "config": {"topic": "plant/area1/telemetry", "qos": 1}
        --     }
        --   ],
        --   "edges": [
        --     {"from": "node-1", "to": "node-2", "from_port": "output", "to_port": "input"},
        --     {"from": "node-2", "to": "node-3", "from_port": "output", "to_port": "input"},
        --     {"from": "node-2", "to": "node-4", "from_port": "output", "to_port": "input"},
        --     {"from": "node-3", "to": "node-5", "from_port": "output", "to_port": "input"},
        --     {"from": "node-4", "to": "node-5", "from_port": "anomaly", "to_port": "input"}
        --   ]
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pipelines_gateway ON data_pipelines(gateway_id);
```

### 8. ML Model Management

```sql
CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,
    framework       VARCHAR(50) NOT NULL,

    -- Model specification — varies significantly by model type and framework
    specification   JSONB NOT NULL,
        -- {
        --   "input_schema": {
        --     "type": "tensor",
        --     "shape": [1, 128],
        --     "dtype": "float32",
        --     "features": ["vibration_x", "vibration_y", "vibration_z", "temperature",
        --                  "current", "speed_rpm", ...],
        --     "normalization": {"type": "z_score", "mean": [...], "std": [...]}
        --   },
        --   "output_schema": {
        --     "type": "tensor",
        --     "shape": [1, 3],
        --     "labels": ["normal", "bearing_fault", "imbalance"],
        --     "threshold": 0.85
        --   },
        --   "preprocessing": [
        --     {"step": "window", "size": 128, "stride": 64},
        --     {"step": "fft", "apply_to": ["vibration_x", "vibration_y", "vibration_z"]}
        --   ],
        --   "target_hardware": {
        --     "min_ram_mb": 64,
        --     "architectures": ["arm64", "x86_64"],
        --     "gpu_required": false,
        --     "benchmark_inference_ms": {"arm64": 45, "x86_64": 12}
        --   }
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ml_model_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES ml_models(id) ON DELETE CASCADE,
    version         VARCHAR(50) NOT NULL,
    artifact_url    VARCHAR(1024) NOT NULL,
    artifact_hash   VARCHAR(64) NOT NULL,
    artifact_size_bytes BIGINT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',

    -- Training and evaluation metrics — different per model type
    training_info   JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "training_dataset": {"size": 50000, "date_range": ["2025-01-01", "2025-12-31"]},
        --   "metrics": {
        --     "accuracy": 0.94,
        --     "precision": 0.91,
        --     "recall": 0.96,
        --     "f1_score": 0.935,
        --     "confusion_matrix": [[4500, 200], [150, 5150]]
        --   },
        --   "hyperparameters": {"learning_rate": 0.001, "epochs": 50, "batch_size": 32},
        --   "framework_version": "onnxruntime 1.17.0"
        -- }

    released_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (model_id, version)
);

CREATE TABLE ml_model_deployments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id    UUID NOT NULL REFERENCES ml_model_versions(id) ON DELETE RESTRICT,
    gateway_id          UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    status              VARCHAR(20) NOT NULL DEFAULT 'deploying',
    deployed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Runtime performance metrics — updated by gateway
    runtime_metrics     JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "inference_count": 15420,
        --   "avg_inference_ms": 38.2,
        --   "p99_inference_ms": 67.1,
        --   "anomalies_detected": 23,
        --   "last_inference_at": "2026-05-25T14:30:00Z",
        --   "memory_usage_mb": 28.4
        -- }

    UNIQUE (model_version_id, gateway_id)
);
```

### 9. OTA Updates and Fleet Deployment

```sql
CREATE TABLE software_releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version         VARCHAR(50) NOT NULL UNIQUE,
    release_type    VARCHAR(30) NOT NULL,
    artifact_url    VARCHAR(1024) NOT NULL,
    artifact_hash   VARCHAR(64) NOT NULL,
    artifact_size_bytes BIGINT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',

    -- Release metadata and requirements
    release_info    JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "release_notes": "Fixed OPC-UA reconnection timeout...",
        --   "target_architectures": ["arm64", "armhf"],
        --   "min_agent_version": "2.3.0",
        --   "min_ram_mb": 128,
        --   "changelog": [...],
        --   "breaking_changes": [],
        --   "dependencies": {"libmodbus": ">=3.1.0"}
        -- }

    released_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rollout_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    release_id      UUID NOT NULL REFERENCES software_releases(id) ON DELETE RESTRICT,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',

    -- Rollout strategy and parameters
    strategy        JSONB NOT NULL,
        -- {
        --   "type": "staged",
        --   "stages": [
        --     {"percentage": 10, "wait_hours": 24, "success_threshold": 0.95},
        --     {"percentage": 50, "wait_hours": 12, "success_threshold": 0.95},
        --     {"percentage": 100, "wait_hours": 0, "success_threshold": 0.90}
        --   ],
        --   "target_filter": {
        --     "sites": ["site-id-1", "site-id-2"],
        --     "tags": {"environment": "production"},
        --     "architectures": ["arm64"]
        --   },
        --   "maintenance_window": {"days": ["Mon", "Tue", "Wed", "Thu", "Fri"],
        --                          "start_hour_utc": 2, "end_hour_utc": 6},
        --   "max_concurrent_updates": 50,
        --   "auto_rollback_on_failure": true
        -- }

    -- Aggregate progress
    progress        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "total": 500, "pending": 200, "in_progress": 15,
        --   "completed": 270, "failed": 5, "rolled_back": 10,
        --   "current_stage": 2, "failure_rate": 0.018
        -- }

    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rollout_gateway_status (
    campaign_id     UUID NOT NULL REFERENCES rollout_campaigns(id) ON DELETE CASCADE,
    gateway_id      UUID NOT NULL REFERENCES edge_gateways(id) ON DELETE CASCADE,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    stage           INTEGER NOT NULL DEFAULT 1,
    previous_version VARCHAR(50),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,

    -- Per-gateway update details
    details         JSONB NOT NULL DEFAULT '{}',
        -- {"download_progress": 0.75, "error": null, "duration_ms": 45000, "rollback_reason": null}

    PRIMARY KEY (campaign_id, gateway_id)
);
```

### 10. Security, Certificates, and Audit

```sql
CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID REFERENCES edge_gateways(id) ON DELETE CASCADE,
    cert_type       VARCHAR(30) NOT NULL,
    subject_cn      VARCHAR(255) NOT NULL,
    fingerprint_sha256 VARCHAR(64) NOT NULL,
    not_before      TIMESTAMPTZ NOT NULL,
    not_after       TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',

    -- Certificate details — vary by type and issuer
    cert_details    JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "issuer_cn": "IIoT Platform CA",
        --   "serial_number": "0A:1B:2C:...",
        --   "key_algorithm": "EC",
        --   "key_size": 256,
        --   "signature_algorithm": "SHA256withECDSA",
        --   "san": ["gateway-abc.iot.example.com"],
        --   "key_usage": ["digitalSignature", "keyEncipherment"],
        --   "extended_key_usage": ["clientAuth"],
        --   "ocsp_url": "https://ocsp.example.com"
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certs_gateway ON certificates(gateway_id);
CREATE INDEX idx_certs_expiry ON certificates(not_after) WHERE status = 'active';

CREATE TABLE audit_log (
    id              BIGSERIAL,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_type      VARCHAR(20) NOT NULL,
    actor_id        VARCHAR(255) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     VARCHAR(255) NOT NULL,
    success         BOOLEAN NOT NULL DEFAULT true,

    -- Full audit context in JSONB — no fixed schema for audit details
    context         JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "ip_address": "203.0.113.42",
        --   "user_agent": "IIoT Dashboard/2.4",
        --   "changes": {"old": {...}, "new": {...}},
        --   "reason": "Scheduled certificate rotation",
        --   "correlation_id": "abc-123"
        -- }

    PRIMARY KEY (timestamp, id)
) PARTITION BY RANGE (timestamp);
```

### 11. Cloud Sync and Data Tiering

```sql
CREATE TABLE data_tiering_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID REFERENCES edge_gateways(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    priority        SMALLINT NOT NULL DEFAULT 5,

    -- Policy rules — JSONB allows complex, composable policies
    policy          JSONB NOT NULL,
        -- {
        --   "scope": {
        --     "type": "all_data_points"  -- or "specific": ["dp-1", "dp-2"] or "by_tag": {"criticality": "high"}
        --   },
        --   "tiers": [
        --     {
        --       "name": "critical_always",
        --       "condition": {"is_critical": true},
        --       "action": "forward_raw",
        --       "destination": "cloud"
        --     },
        --     {
        --       "name": "normal_aggregated",
        --       "condition": {"is_critical": false, "anomaly_score": {"lt": 0.5}},
        --       "action": "forward_aggregate",
        --       "aggregate_window_ms": 300000,
        --       "aggregate_function": "avg",
        --       "destination": "cloud"
        --     },
        --     {
        --       "name": "anomaly_raw",
        --       "condition": {"anomaly_score": {"gte": 0.5}},
        --       "action": "forward_raw",
        --       "destination": "cloud",
        --       "include_context_window_ms": 60000
        --     }
        --   ],
        --   "bandwidth_limit_kbps": 500,
        --   "local_retention_hours": 168,
        --   "cloud_retention_days": 365
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 12. Users and Access Control

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(30) NOT NULL DEFAULT 'operator',
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,

    -- User preferences and profile — varies per user
    preferences     JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "timezone": "America/Chicago",
        --   "dashboard_layout": "compact",
        --   "notification_preferences": {
        --     "email": {"critical": true, "warning": true, "info": false},
        --     "sms": {"critical": true, "warning": false, "info": false},
        --     "slack": {"critical": true, "warning": true, "info": true}
        --   },
        --   "default_site_id": "site-xyz",
        --   "units": "metric"
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_site_access (
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    site_id     UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    role        VARCHAR(30) NOT NULL,
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, site_id)
);
```

---

## Edge-Side SQLite Schema

```sql
-- Store-and-forward buffer with JSON for flexible payloads
CREATE TABLE telemetry_buffer (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    data_point_id   TEXT NOT NULL,
    recorded_at     TEXT NOT NULL,
    value_numeric   REAL,
    extended_value  TEXT,                           -- JSON for multi-value readings
    quality         INTEGER NOT NULL DEFAULT 192,
    sync_status     TEXT NOT NULL DEFAULT 'pending',
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_buffer_sync ON telemetry_buffer(sync_status) WHERE sync_status = 'pending';

-- Local device/config cache using JSON
CREATE TABLE config_cache (
    key             TEXT PRIMARY KEY,
    value           TEXT NOT NULL,                  -- JSON document
    version         INTEGER NOT NULL DEFAULT 1,
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Pipeline graph cached locally for offline operation
CREATE TABLE pipeline_cache (
    pipeline_id     TEXT PRIMARY KEY,
    graph           TEXT NOT NULL,                  -- JSON pipeline graph
    version         INTEGER NOT NULL,
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## Pros and Cons

### Pros

1. **Protocol abstraction without table explosion** — Instead of separate tables for Modbus device config, OPC-UA device config, BACnet device config, and DNP3 device config (each with completely different columns), a single `connection_config JSONB` column on `protocol_adapter_instances` handles all protocols. Adding a new protocol (e.g., PROFINET, EtherNet/IP) requires no schema migration — just a new `protocol_adapter_type` row with its `config_schema`.

2. **Schema evolution without migrations** — Adding a new field to device metadata, hardware specs, or training info requires no ALTER TABLE. The JSONB column naturally absorbs new fields. This is critical for a platform deploying to edge devices where schema migrations must be coordinated across hundreds of gateways.

3. **Rich query capability on flexible data** — PostgreSQL's JSONB operators (`->`, `->>`, `@>`, `?`, `jsonb_path_query`) combined with GIN indexes enable efficient queries like "find all devices where `device_metadata->>'manufacturer' = 'Siemens'`" or "find all gateways with `tags @> '{"environment": "production"}'`" without sacrificing query performance.

4. **Single database engine** — Everything runs on PostgreSQL (with TimescaleDB extension for telemetry). No separate document database, no separate time-series database. This dramatically simplifies operations, backup, monitoring, and team skill requirements.

5. **Validated flexibility** — The `config_schema` column on `protocol_adapter_types` stores a JSON Schema that can validate `connection_config` values before insertion. This provides document-database flexibility with relational-database validation discipline.

6. **Natural fit for visual pipeline editor** — Storing the entire pipeline graph as a single JSONB document matches how visual editors (Node-RED, Kura Wire) serialize their canvas state. Loading and saving a pipeline is a single row read/write rather than reconstructing from many normalized tables.

7. **ISA-95 semantic mapping** — The `semantic_mapping` JSONB on data points allows progressive ISA-95 normalization. Not every sensor needs full semantic mapping from day one — teams can add mappings as they integrate with MES/ERP systems.

### Cons

1. **No foreign key enforcement inside JSONB** — References stored inside JSONB columns (e.g., `data_point_id` values inside a pipeline graph's nodes) are not validated by the database. Application-level validation must enforce these constraints, increasing the risk of dangling references.

2. **JSONB storage overhead** — JSONB stores field names with every row. A column like `connection_config` that stores `{"host": "...", "port": ...}` for every Modbus adapter repeats the field names "host" and "port" in every row. For high-volume tables like telemetry, the `extended_value` JSONB column adds measurable storage overhead compared to dedicated columns.

3. **Query plan opacity** — SQL query planners cannot reason about JSONB column selectivity as well as typed columns. Queries filtering on JSONB fields may produce suboptimal plans. GIN indexes help but are not as efficient as B-tree indexes on typed columns.

4. **Schema documentation burden** — The flexibility of JSONB means the actual schema is documented in code comments and application documentation rather than enforced by DDL. New team members must read documentation to understand what fields are expected in each JSONB column, whereas relational columns are self-documenting.

5. **Migration complexity for JSONB contents** — When a JSONB field structure changes (e.g., renaming `device_metadata.serial` to `device_metadata.serial_number`), an UPDATE statement must modify the JSONB contents of existing rows. This is slower and more error-prone than an ALTER TABLE RENAME COLUMN.

6. **Compression inefficiency** — TimescaleDB compression works best on typed columns. JSONB columns compress less efficiently than equivalent typed columns because the compression algorithm cannot exploit the predictable structure of typed data.

---

## Migration and Scaling Considerations

### Schema Evolution Strategy
- **Relational columns**: standard ALTER TABLE migrations via Flyway
- **JSONB columns**: two approaches:
  - **Lazy migration**: application code handles both old and new JSONB structures; old rows are migrated on next write
  - **Batch migration**: scheduled UPDATE query transforms JSONB structure across all rows; use `jsonb_set()`, `jsonb_strip_nulls()`, and path operations
- **Version tracking**: consider adding a `schema_version` integer column alongside large JSONB columns to enable efficient batch migrations

### Indexing Strategy
- **GIN indexes**: create on JSONB columns used in WHERE clauses; GIN supports `@>` (containment), `?` (key existence), and `?&` (all keys exist)
- **Expression indexes**: for frequently queried JSONB paths, create expression indexes:
  ```sql
  CREATE INDEX idx_device_manufacturer ON devices ((device_metadata->>'manufacturer'));
  CREATE INDEX idx_gateway_env ON edge_gateways ((tags->>'environment'));
  ```
- **Partial GIN indexes**: for JSONB columns that are often NULL (e.g., `extended_value` on telemetry), use partial indexes to save space

### Scaling Path
1. **Single PostgreSQL + TimescaleDB** — handles hundreds of gateways, tens of thousands of data points
2. **Read replicas** — offload dashboard and analytics queries
3. **Citus extension** — shard by organization_id for multi-tenant SaaS deployments; JSONB columns are fully supported in Citus distributed tables
4. **TimescaleDB multi-node** — for telemetry at massive scale

### Storage Estimates
- Device registry with rich JSONB metadata: ~5 KB per device = 50 MB for 10,000 devices (negligible)
- Telemetry with occasional `extended_value`: ~150 bytes/row average = ~650 GB/day for 50,000 points at 1 Hz
- With TimescaleDB compression: ~65 GB/day (10:1 typical compression on mixed typed + JSONB data)
- 90-day raw retention: ~5.8 TB compressed

### Edge-Cloud Sync
- JSONB payloads sync naturally over MQTT as JSON strings
- No schema negotiation needed between edge and cloud — both sides use the same JSON structure
- Schema version mismatches are handled by the application code's lazy migration pattern
- This is a significant operational advantage over pure relational schemas where edge and cloud must have identical DDL

---

## Technology Recommendations

1. **Adopt the hybrid model as the default approach** — it provides the best balance of structure and flexibility for the industrial IoT domain where protocol diversity is the core challenge.

2. **Use `pg_jsonschema` extension** for runtime validation of critical JSONB columns (connection_config, condition, strategy) to prevent malformed data without sacrificing flexibility.

3. **Create expression indexes eagerly** — whenever a JSONB path appears in a WHERE clause in application code, add a corresponding expression index. Review slow query logs monthly to catch missing indexes.

4. **Prefer JSONB over separate tables** for data that:
   - Varies by protocol, device type, or deployment
   - Is read and written as a complete document (e.g., pipeline graph)
   - Changes shape frequently during development
   - Has optional/sparse fields (most rows have only a subset of possible fields)

5. **Prefer relational columns** for data that:
   - Participates in JOINs or foreign keys
   - Is used in aggregate queries (GROUP BY, COUNT, SUM)
   - Has a fixed, universal schema (timestamps, IDs, status enums)
   - Needs to be indexed with B-tree for range scans

6. **Store TimescaleDB continuous aggregate definitions in version control** alongside Flyway migrations — they are as critical as table definitions and must be recreated consistently across environments.
