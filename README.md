# Industrial IoT Edge Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A vendor-agnostic edge computing platform that collects industrial telemetry using native OT protocols, applies local filtering and anomaly detection, and synchronises actionable intelligence to the cloud -- eliminating cloud lock-in while cutting data backhaul costs by up to 80%.

Industrial facilities run equipment that speaks decades-old fieldbus protocols (Modbus, OPC-UA, PROFIBUS, DNP3, BACnet) and generates high-frequency telemetry that is prohibitively expensive to stream raw to the cloud. This project delivers lightweight edge agents for ruggedised gateway hardware that collect, normalise, and filter data at the source, then sync processed results and alerts to any cloud back-end. The result is lower bandwidth costs, offline resilience, and sub-100 ms local decision-making -- without tying operators to a single hyperscaler.

---

## Why Industrial IoT Edge Platform?

- **Cloud provider lock-in is the norm.** AWS IoT Greengrass and Azure IoT Edge both deliver strong edge capabilities, but migrating between them requires significant rearchitecting. There is no mature vendor-agnostic alternative with comparable protocol breadth.

- **Legacy protocol support is shallow outside proprietary stacks.** AWS Greengrass requires SiteWise or custom components for Modbus and PROFIBUS; Azure IoT Edge relies on separate OPC Publisher modules. Neither platform offers broad, built-in coverage of the five-plus protocols found in a typical facility.

- **Open-source options are heavyweight and dated.** Eclipse Kura provides the widest open-source protocol support, but its Java runtime demands 512 MB+ RAM, its web UI is not designed for enterprise-scale fleet management, and community support can be slow without a Eurotech commercial agreement.

- **Edge ML and AI capabilities are locked behind cloud ecosystems.** Running anomaly detection or predictive maintenance models at the edge today requires SageMaker (AWS) or Azure ML packaging pipelines, reinforcing vendor dependency.

- **Cost pressure is real.** Organisations can achieve an average 80% reduction in data backhaul costs by filtering noise at the source, but current platforms make this difficult to configure without deep cloud-specific expertise.

---

## Key Features

### Protocol Adapters

- OPC-UA client for connecting to SCADA and DCS systems (with OPC Foundation vendor licence compliance)
- Modbus TCP/RTU adapter for the most widely deployed industrial serial protocol
- MQTT broker and client for device-to-device and device-to-cloud messaging
- BACnet adapter for building automation and HVAC environments
- DNP3 adapter for energy and utility SCADA systems

### Edge Processing Engine

- Configurable data tiering: send raw telemetry, windowed aggregates, or threshold-triggered alerts based on content and bandwidth policy
- Local store-and-forward buffer using high-endurance storage to maintain data integrity during WAN outages
- Offline-first architecture with automatic cloud synchronisation and conflict resolution when connectivity resumes
- Edge ML inference runtime for deploying pre-trained anomaly detection models without cloud round-trips

### Fleet Management & Deployment

- Container or OSGi component-based application deployment from a central management console
- OTA edge software updates with staged rollout and automatic rollback on failure
- Secure tunnel for remote access to edge device management interfaces
- Mutual TLS for all cloud communication with certificate rotation capability

### Visual Configuration & Operations

- Visual dataflow editor for configuring data routing and filtering pipelines without writing code
- Cloud-agnostic MQTT broker connectivity supporting AWS IoT Core, Azure IoT Hub, and generic MQTT endpoints
- Unified operations dashboard with real-time and historical views across the entire edge fleet

### Advanced Capabilities (Backlog)

- Digital twin integration pushing normalised telemetry to cloud-side facility models
- OT network security monitoring detecting anomalous industrial protocol traffic patterns
- ISA-95 semantic normalisation layer standardising diverse OT data into a unified information model
- Protocol auto-discovery using observed network traffic to suggest device types and configuration

---

## AI-Native Advantage

This platform treats AI as a first-class edge citizen rather than a cloud-only add-on. Predictive maintenance models for vibration, temperature, and pressure analysis run directly on gateway hardware, delivering sub-100 ms inference without WAN dependency. Adaptive edge filtering uses ML to dynamically adjust which data streams are forwarded to the cloud based on anomaly likelihood, reducing backhaul without losing critical signals. Automated protocol discovery identifies connected device types and protocol parameters from observed traffic patterns, dramatically reducing commissioning time for heterogeneous OT environments.

---

## Tech Stack & Deployment

The platform targets ruggedised ARM Cortex-A series gateways with as little as 128 MB RAM -- significantly lighter than Eclipse Kura's 512 MB minimum or the full container runtimes required by AWS Greengrass and Azure IoT Edge. Deployment modes include self-hosted on-premises gateways with optional cloud management, and hybrid configurations where edge agents operate autonomously during WAN outages and sync when connectivity is available.

Key standards and protocols: OPC-UA (IEC 62541), Modbus (public domain), DNP3, BACnet, MQTT. Industrial cybersecurity alignment with IEC 62443 for network zoning and secure communications. The component model supports both OSGi hot-pluggable modules and container-based workloads depending on hardware capability.

---

## Market Context

The industrial edge computing market is characterised as a mandatory architectural shift rather than an emerging option, with organisations reporting up to 80% reduction in data backhaul costs through edge filtering. Current incumbents -- AWS IoT Greengrass, Azure IoT Edge, and Portainer Edge -- are either hyperscaler-locked or lack native industrial protocol support. Eclipse Kura is open-source (EPL-2.0) but resource-heavy and enterprise-unfriendly without commercial support. Primary buyers are industrial operations teams, OT engineers, and plant IT departments in manufacturing, energy, utilities, and building automation.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

Notable licensing considerations: OPC-UA implementations require the OPC Foundation Vendor License Agreement for commercial use. Eclipse Kura components used under EPL-2.0 require disclosure of modifications to EPL-covered files but allow proprietary modules via standard OSGi interfaces. Modbus, DNP3, and BACnet are published standards available under open terms.
