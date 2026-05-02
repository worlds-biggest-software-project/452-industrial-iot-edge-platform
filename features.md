# Industrial IoT Edge Platform — Feature & Functionality Survey

> Candidate #452 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| AWS IoT Greengrass | Managed cloud / edge | Commercial (AWS) | https://aws.amazon.com/greengrass/ |
| Azure IoT Edge | Managed cloud / edge | Commercial (Microsoft) | https://azure.microsoft.com/products/iot-edge |
| Portainer Edge | SaaS / self-hosted | Commercial (Business Edition) | https://www.portainer.io |
| Robustel | Hardware + software | Commercial | https://robustel.com |
| Eclipse Kura | Open source | Eclipse Public Licence 2.0 | https://eclipse.dev/kura/ |

## Feature Analysis by Solution

### AWS IoT Greengrass

**Core features**
- Extends AWS Lambda and container workloads to edge hardware, enabling local compute without continuous cloud connectivity
- Greengrass components: modular software packages deployed and managed from the AWS cloud to edge devices
- Local MQTT message broker for device-to-device communication within the facility without cloud round-trips
- Secrets management and certificate rotation at the edge via integration with AWS Secrets Manager
- Stream manager: local buffering of data streams with configurable export to AWS IoT Core, S3, or Kinesis when connectivity is available
- OTA component updates pushed from AWS IoT Jobs

**Differentiating features**
- ML inference at the edge: deploy SageMaker-trained models to Greengrass for local anomaly detection and process optimisation
- Native integration with the full AWS service ecosystem (CloudWatch, S3, Kinesis, Lambda) for cloud-tier processing
- Greengrass Nucleus as a lightweight Java runtime deployable on resource-constrained Linux hardware

**UX patterns**
- AWS Console: fleet deployment, component version management, and OTA update status in the IoT Core console
- AWS IoT SiteWise: industrial asset modelling and data aggregation layered on top of Greengrass-collected telemetry
- CloudFormation and Terraform support for infrastructure-as-code edge deployments

**Integration points**
- OPC-UA and Modbus via AWS IoT SiteWise connector at the edge
- AWS SageMaker for edge ML model deployment
- AWS IoT TwinMaker for digital twin construction from edge sensor data

**Known gaps**
- AWS ecosystem lock-in: migrating to another platform requires significant rearchitecting
- Protocol adapter breadth for legacy industrial protocols is narrower without SiteWise; direct Modbus/PROFIBUS support requires custom components
- Greengrass v2 has a steeper learning curve than simpler edge platforms for teams without AWS expertise

**Licence / IP notes**
- Proprietary commercial service. Greengrass Core runtime is available under the Apache 2.0 licence for the open-source components; full service requires AWS account.

---

### Azure IoT Edge

**Core features**
- Container-based runtime deploying Docker modules to edge devices managed from Azure IoT Hub
- Module composition: each edge workload (protocol adapter, ML inference, data pipeline) runs as a separate container with defined message routes between modules
- Azure IoT Edge runtime: security daemon and module runtime for Linux and Windows edge devices
- OTA deployment manifests: declarative JSON configuration pushed from IoT Hub defining desired module composition and configuration
- Offline operation: edge devices continue processing and caching data during WAN disconnection; sync resumes automatically on reconnection

**Differentiating features**
- Deepest integration in the Microsoft ecosystem: Azure Machine Learning for edge ML, Azure Stream Analytics for SQL-based edge data processing, and Azure Digital Twins for facility modelling
- Module marketplace: pre-built certified modules from ISVs available in the Azure Marketplace for direct deployment to edge fleets
- Windows IoT Enterprise support: the only major platform natively supporting Windows-based industrial PCs alongside Linux

**UX patterns**
- Azure Portal and IoT Hub for device registry, module deployment, and twin synchronisation
- VS Code IoT Edge extension for local module development and testing before cloud deployment
- Azure Monitor integration for edge device and module health metrics

**Integration points**
- OPC Publisher module (Microsoft): connects to OPC-UA servers on plant equipment and publishes telemetry to IoT Hub
- Azure Stream Analytics on IoT Edge for windowed aggregation and filtering at the edge
- Azure Machine Learning for edge model packaging and deployment via Azure IoT Edge

**Known gaps**
- Requires running containers on edge hardware; not suitable for very constrained microcontrollers or embedded systems
- Configuration complexity for large heterogeneous fleets with many different module compositions
- Microsoft cloud lock-in comparable to AWS Greengrass; multi-cloud edge is not natively supported

**Licence / IP notes**
- Azure IoT Edge runtime: open-source under MIT licence on GitHub. Azure IoT Hub and cloud services: commercial. Module marketplace: third-party ISV licensing varies.

---

### Eclipse Kura

**Core features**
- Open-source Java-based IoT edge framework running on industrial gateways
- Protocol adapters: OPC-UA, Modbus, S7 (Siemens), DNP3, and BACnet built in or available via Eclipse Marketplace
- Cloud connectivity: pre-built connectors for AWS IoT, Azure IoT Hub, Eclipse Kapua, and generic MQTT brokers
- Wire framework: visual dataflow programming enabling non-engineers to configure data routing, filtering, and transformation pipelines between protocol adapters and cloud connectors
- OSGi component model: modular, hot-pluggable software components enabling field updates without full gateway restart

**Differentiating features**
- Broadest industrial protocol coverage of any open-source edge platform — a key differentiator for heterogeneous OT environments
- Wire graphical editor: visual pipeline configuration accessible to operational technology engineers without programming skills
- Eclipse ecosystem: integrates with Eclipse Kapua (cloud IoT platform) and Eclipse hawkBit (OTA updates) for a complete open-source IIoT stack

**UX patterns**
- Web-based configuration console embedded on the gateway itself
- Wire composition canvas for drag-and-drop dataflow configuration
- Deployment package management for updating Kura bundles via the admin interface

**Integration points**
- MQTT to any broker (AWS IoT, Azure IoT Hub, HiveMQ, Mosquitto)
- OPC-UA client for connecting to SCADA and DCS systems
- Eclipse Kapua and Eurotech Everyware Cloud as primary management back-ends

**Known gaps**
- Java runtime requires 512 MB RAM minimum; too heavy for very constrained embedded hardware
- Web UI is dated; not designed for enterprise fleet management at thousands of devices
- Community support can be slow; enterprise support requires Eurotech commercial agreement

**Licence / IP notes**
- Eclipse Public Licence 2.0: modifications to EPL files must be disclosed; proprietary code using EPL components via standard interfaces need not be disclosed.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Protocol adapters for OPC-UA and Modbus at minimum as the most common industrial protocols
- Local data buffering (store-and-forward) maintaining data integrity during WAN outages
- Container or component-based application deployment enabling field software updates
- Cloud synchronisation: configurable data tiering sending raw data, filtered summaries, or only alerts to the cloud
- Remote management: configuration push, software updates, and health monitoring from a central console

### Differentiating Features
- ML inference at the edge: anomaly detection and process optimisation models running locally without cloud round-trip latency
- Visual dataflow programming for operational technology engineers without software development skills (Kura Wire)
- Windows IoT Enterprise support for industrial PCs running Windows-based SCADA (Azure IoT Edge)
- Offline-first architecture with automatic cloud synchronisation and conflict resolution (Azure IoT Edge, AWS Greengrass)
- Digital twin integration: edge telemetry feeding a cloud-side facility model for process simulation and monitoring

### Underserved Areas / Opportunities
- Vendor-agnostic edge platform with broad industrial protocol support that does not require cloud provider lock-in
- Lightweight edge runtime deployable on ARM Cortex-A series hardware with under 128 MB RAM without a full container runtime
- Protocol normalisation as a service: translating heterogeneous OT data into a unified semantic model (ISA-95, OPC-UA information model) before cloud upload
- OT security monitoring: detecting anomalous protocol traffic patterns at the network edge indicating industrial cyberattack (e.g., Modbus write commands outside normal parameters)

### AI-Augmentation Candidates
- Predictive maintenance models running at the edge on vibration, temperature, and pressure data without cloud connectivity
- Automated protocol discovery: AI identification of connected device types and protocol parameters from observed traffic patterns
- Adaptive edge filtering: ML-dynamically adjusting which data streams are forwarded to cloud based on anomaly likelihood
- Natural-language OT configuration: describing equipment connections in plain language and generating gateway configuration automatically

## Legal & IP Summary

Eclipse Kura (EPL-2.0) is the most feature-rich open-source option for industrial protocol support. EPL-2.0 modifications to Kura source files must be disclosed, but proprietary modules added to the OSGi runtime do not trigger copyleft. Azure IoT Edge runtime (MIT) is the most permissively licensed open-source runtime. AWS Greengrass includes Apache 2.0 open-source components but the managed cloud service is commercial. OPC-UA is an IEC 62541 international standard maintained by the OPC Foundation; there is a published patent licence (the OPC Foundation Vendor License Agreement) required for commercial OPC-UA implementations — this is a notable licensing requirement for any new entrant implementing OPC-UA natively. Modbus is a public domain protocol with no patent encumbrances. DNP3 and BACnet are published standards available under open terms. Industrial cybersecurity standards (IEC 62443) impose process requirements on security certifications, not software licensing obligations.

## Recommended Feature Scope

**Must-have (MVP)**:
- Protocol adapters: OPC-UA (with OPC Foundation vendor licence), Modbus TCP/RTU, and MQTT as minimum industrial protocol support
- Local store-and-forward buffer using high-endurance local storage maintaining data integrity during WAN outages
- Configurable data tiering: send raw data, windowed aggregates, or threshold-triggered alerts to cloud based on content and bandwidth policy
- Container or OSGi component-based application deployment from central management console
- Secure tunnel for remote access to edge device management interface from cloud
- Mutual TLS for all cloud communication with certificate rotation capability

**Should-have (v1.1)**:
- Visual dataflow editor for configuring data routing and filtering pipelines without code
- Cloud-agnostic MQTT broker connectivity: AWS IoT Core, Azure IoT Hub, and generic MQTT endpoint support
- OTA edge software updates with staged rollout and automatic rollback on failure
- Edge ML inference runtime: deploy pre-trained anomaly detection models for local execution
- BACnet and DNP3 protocol adapters for building automation and energy/utility environments

**Nice-to-have (backlog)**:
- Digital twin integration pushing normalised telemetry to cloud-side facility models
- OT network security monitoring detecting anomalous industrial protocol traffic patterns
- Predictive maintenance models running on edge hardware for vibration and thermal analysis
- Protocol auto-discovery using observed network traffic patterns to suggest device type and configuration
- ISA-95 semantic normalisation layer standardising diverse OT data into a unified information model
