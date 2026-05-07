# Standards & API Reference

> Project: Industrial IoT Edge Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### Industrial Communication Protocols

- **OPC-UA (Open Platform Communications Unified Architecture)** — https://opcfoundation.org/
  Platform-independent protocol for secure machine-to-machine communication. Standardized as IEC 62541. Provides diagnostics, discovery, security, and complex information model rendering. De-facto standard for modern industrial automation integration.

- **Modbus** — https://www.modbus.org/
  Oldest industrial protocol (1979), still widely used. Supports serial (RTU, ASCII) and TCP/IP variants. Simple master-slave architecture for register-based data exchange. Most commonly used protocol in legacy industrial systems.

- **PROFIBUS / PROFINET** — https://www.profibus.com/
  PROFIBUS is the established fieldbus standard for automation. PROFINET is the successor, an open standard for Industrial Ethernet with real-time data exchange capabilities. Enables integration with modern IT systems while maintaining legacy device support.

- **DNP3 (Distributed Network Protocol)** — https://www.dnp.org/
  Suite of open protocols for SCADA systems and electrical grid infrastructure. Commonly used in utility substations for serial connectivity and remote terminal unit (RTU) communication.

- **BACnet** — https://www.bacnet.org/
  Communication protocol for building automation and control systems (fire detection, access control, HVAC). Originally ASHRAE/ANSI Standard 135-1995, now ISO 16484-5. Used less in pure industrial contexts, more in facility management.

- **IEC 61850** — https://en.wikipedia.org/wiki/IEC_61850
  Set of open protocols for electrical utility communication. Used by SCADA masters, remote terminal units (RTUs), and intelligent electronic devices (IEDs) in electrical substations and power systems.

### Manufacturing Data Standards

- **ISA-95 (ANSI/ISA-95)** — https://www.isa.org/standards-and-publications/isa-standards/isa-95-standard
  International standard for enterprise-control system integration in manufacturing. Defines data models and integration patterns between ERP and MES (Manufacturing Execution Systems). Standardized globally as IEC/ISO 62264.

- **ISA-88 (ANSI/ISA-88)** — Batch Control Standard
  Provides specifications for batch control systems used in process industries. Complements ISA-95 with batch-specific operational requirements and sequencing.

- **B2MML (Business-to-Manufacturing Markup Language)** — https://github.com/MESAInternational/B2MML-BatchML
  XML and JSON schema implementations of ISA-95 and ISA-88. Provides standard way to structure manufacturing execution data for integration between systems.

- **ISA-JSON Format** — https://isa-specs.readthedocs.io/en/latest/isajson.html
  JSON serialization specification for ISA standards, enabling structured data exchange compatible with cloud APIs and modern edge platforms.

### Data Model & API Specifications

- **OpenAPI 3.1** — https://www.openapis.org/
  Industry standard for documenting REST APIs. Used by AWS SiteWise, Azure Industrial IoT, and other edge platforms for gateway management and data ingestion APIs.

- **JSON Schema** — Standard for validating industrial telemetry and configuration data structures in REST payloads.

- **InfluxDB Line Protocol** — Standard text format for time-series data ingest, optimized for industrial metrics with timestamps and tags.

- **Prometheus Metrics Format** — Common format for exposing infrastructure and application metrics from edge gateways.

### Security & Compliance Standards

- **IEC 62443** — https://en.wikipedia.org/wiki/IEC_62443
  Industrial control systems security standard. Defines security levels (SL 1-4) for industrial environments. Increasingly critical for IIoT deployments handling sensitive operational data.

- **NIST Cybersecurity Framework** — https://www.nist.gov/cyberframework
  Comprehensive guidance for identifying, protecting, detecting, responding to, and recovering from cybersecurity risks. Widely referenced in industrial IoT security implementations.

- **ISO/IEC 27001** — https://www.iso.org/standard/27001
  Information security management system standard. Applies to industrial edge platforms handling operational technology (OT) data and control systems.

- **IEC 61508** — Functional Safety standard
  Establishes requirements for electrical/electronic/programmable electronic safety systems. Critical for safety-critical industrial applications.

- **TLS 1.2 / 1.3** — https://datatracker.ietf.org/doc/html/rfc8446
  Encryption standard for securing edge-to-cloud communication. Mandatory for protecting sensitive industrial telemetry and commands.

- **Modbus Security Suite** — Standard extension for secure authentication and encryption over legacy Modbus TCP.

### Edge Computing Standards

- **IETF RFC 8288 — Web Linking** — Standard for linking resources in REST APIs used by edge gateway management.

- **IETF RFC 7231 — HTTP/1.1 Semantics** — Foundation for REST API design in gateway management interfaces.

## Similar Products — Developer Documentation & APIs

### AWS IoT SiteWise
- **Description:** Fully managed industrial data ingestion, storage, and visualization service. Operates both at edge (SiteWise Edge) and in cloud with OPC-UA and protocol adapter support.
- **API Documentation:** https://docs.aws.amazon.com/iot-sitewise/latest/userguide/what-is-sitewise.html
- **Gateway Management API:** 
  - CreateGateway: https://docs.aws.amazon.com/iot-sitewise/latest/APIReference/API_CreateGateway.html
  - DescribeGateway: https://docs.aws.amazon.com/iot-sitewise/latest/APIReference/API_DescribeGateway.html
  - ListGateways: https://docs.aws.amazon.com/iot-sitewise/latest/APIReference/API_ListGateways.html
- **Data Ingestion Guide:** https://docs.aws.amazon.com/iot-sitewise/latest/userguide/industrial-data-ingestion.html
- **Supported Protocols:** OPC-UA, Modbus TCP, Ethernet/IP (EIP)
- **Standards:** REST/JSON API, OpenAPI 3.1 documentation
- **Authentication:** AWS IAM, X.509 certificates
- **Edge Runtime:** Greengrass, GreengrassV2, Siemens IE

### EdgeX Foundry
- **Description:** Vendor-neutral open source framework for IoT edge computing hosted by Linux Foundation. Microservices-based architecture for collecting, processing, and filtering data at the edge.
- **Documentation:** https://docs.edgexfoundry.org/
- **Getting Started:** https://docs.edgexfoundry.org/4.0/getting-started/quick-start/
- **API References:**
  - Core Metadata: https://docs.edgexfoundry.org/4.1/microservices/core/metadata/ApiReference/
  - Device Service: https://docs.edgexfoundry.org/3.1/microservices/device/ApiReference/
  - App Services: https://docs.edgexfoundry.org/3.2/microservices/application/ApiReference/
- **Demonstration:** https://docs.edgexfoundry.org/4.0/walk-through/Ch-Walkthrough/
- **Protocol Support:** MQTT, REST, CoAP, Modbus, OPC-UA through protocol adapters
- **Standards:** Microservices REST APIs, OpenAPI documentation
- **License:** Open-source (Apache 2.0)
- **Deployment:** Docker, Kubernetes, edge servers

### InfluxDB (Time-Series Data)
- **Description:** High-performance time-series database optimized for IoT and industrial metrics. Handles large volumes of data at high speeds. Commonly paired with Telegraf for data collection.
- **Documentation:** https://docs.influxdata.com/
- **API Documentation:** REST API with line protocol ingest
- **Client Libraries:** Python, JavaScript, Go, Java, C#
- **Query Language:** InfluxQL and Flux
- **Integration:** Works with Grafana, Telegraf, MQTT brokers
- **License:** Open-source (AGPL) and commercial options

### Grafana
- **Description:** Open-source platform for querying, visualizing, and alerting on metrics. Supports dozens of data sources including InfluxDB, TimescaleDB, Prometheus, and databases.
- **Documentation:** https://grafana.com/docs/
- **API Documentation:** REST API for dashboard creation, data source management, alerting
- **SDKs/Libraries:** Python, JavaScript, Go
- **Visualization:** Dashboard creation, alert management, multi-source metric correlation
- **Standards:** REST/JSON APIs
- **License:** AGPL (open-source) with commercial options
- **Integration:** Enterprise-grade integrations with industrial monitoring systems

### TimescaleDB
- **Description:** Open-source relational database built on PostgreSQL for time-series data. Provides full SQL compatibility with time-series optimizations and compression.
- **Documentation:** https://docs.timescale.com/
- **API:** Native PostgreSQL protocol with time-series-specific extensions
- **Client Libraries:** All PostgreSQL clients (Python, JavaScript, Go, Java, C#)
- **Time-Series Extensions:** Hypertables for automatic data partitioning and compression
- **Standards:** ANSI SQL, PostgreSQL compatibility
- **License:** Open-source (Timescale License) with commercial options

### Node-RED
- **Description:** Flow-based programming tool for IoT automation and integration. Visual approach to wiring industrial data sources, processing logic, and integrations.
- **Documentation:** https://nodered.org/docs/
- **API Documentation:** REST API for flow management, node control
- **Protocol Support:** MQTT, HTTP, OPC-UA, Modbus (via node libraries)
- **Standard Nodes:** MQTT, HTTP, CSV, JSON parsing, time-series formatting
- **Standards:** REST/JSON APIs
- **License:** Open-source (Apache 2.0)
- **Deployment:** Docker, Kubernetes, ARM devices (Raspberry Pi)

### Azure Industrial IoT
- **Description:** Microsoft cloud platform for collecting, processing, and analyzing industrial data. Supports Protocol Translation (PT) services for OPC-UA to Azure conversion.
- **API Documentation:** Azure IoT platform documentation
- **Supported Protocols:** OPC-UA, MQTT, HTTPS
- **Standards Compliance:** Integration with ISA-95, Industrial Ethernet protocols
- **Cloud Integration:** Time-series analytics, Power BI visualization
- **Authentication:** Azure AD, certificates, connection strings

## Notes

### Standards Consolidation

Industrial communication continues fragmenting between legacy (Modbus, PROFIBUS) and modern (OPC-UA, Ethernet/IP) protocols. Edge platforms must support multiple protocols through adapters. ISA-95 provides data structure standardization, but implementation remains vendor-specific.

### Security Evolution

IEC 62443 adoption is increasing in industrial settings, with security levels driving architectural decisions. Edge platforms must support secure certificate provisioning, encrypted store-and-forward, and secure OTA updates for gateway firmware.

### Time-Series Optimization

Modern platforms increasingly standardize on efficient time-series formats (InfluxDB line protocol, Prometheus metrics) for cloud synchronization, reducing bandwidth and storage costs while enabling real-time analytics.

### Protocol Adapter Patterns

Most platforms use plugin/adapter architecture (EdgeX, AWS SiteWise) to support legacy and new industrial protocols without core rewrites. This pattern is becoming industry best practice.

### Data Standardization Gaps

While ISA-95 standardizes manufacturing data models (MES level), there's no universal standard for raw sensor/device data structures at the edge. Each platform defines its own telemetry format (InfluxDB line protocol, Prometheus, JSON), requiring transformation layers.

### AI-Augmentation Opportunities

- Intelligent protocol bridging and data format conversion using LLMs
- Anomaly detection on time-series industrial data with local ML inference
- Predictive maintenance scheduling optimization based on equipment patterns
- Automated rule generation from operational data patterns
- Smart data filtering and compression at edge to minimize cloud bandwidth
